# Hosting a Static React App on AWS: S3 + CloudFront


## Prerequisites for this tutorial

1. An AWS account with CLI access configured (aws configure completed)
2. IAM credentials with permissions for S3 and CloudFront
3. A React build output in a local /build folder

In the repository, we have a React app that we want to host on AWS using S3 and CloudFront.

The first thing we need to do is to build the app. We can do this by running the following commands:

`npm install`

`npm run build`

This will have createda a `build` folder in the root directory of the project.

---

## Manually by hand

First, let's do the process by hand so we can get an understanding for what is happening.

### 1. Create the S3 Bucket (Private by Design)

First, create your storage container.

1. Browse to the S3 console.
2. Create a bucket (e.g. tutorial1).
   1. Note the bucket name has to be unique across all buckets created on earth.
   2. Make sure you keep a note of the bucket name you will need it later
3. Keep "Block all public access" checked. 
   1. Note: In a production environment, you never want your S3 endpoint exposed. By blocking public access, you prevent users from bypassing your CDN (CloudFront), which would otherwise lead to higher S3 egress costs and bypass your security headers or Web Application Firewall (WAF) rules.

Make a note of the distribution ID, you will need it later.

---

### 2. Setup CloudFront & Origin Access Control (OAC)

1. Browse to the CloudFront console.
2. Create a new distribution.
3. Select your plan
4. Provide a distribution name and description
5. Select the origin type to be S3
6. Select the bucket you created earlier
7. Use the default settings for the rest of the fields.

---

### 3. Configure Cloudfront 

React apps are Single Page Applications (SPAs), so we need to handle the "client-side routing" problem where refreshing a sub-page (like /dashboard) throws a 403/404 from S3.

1. Select the distribution you created earlier.
2. On the "General" tab, click "Edit" and set the "Default Root Object" to "index.html". 
   1. By default, CloudFront is just a pass-through. If a user hits https://your-app.com/, CloudFront sends a request to S3 for the "root" of the bucket. Without this setting: S3 doesn't know what to return for a "folder" request (since S3 is a flat key-value store, not a real file system). It will either return a 403 Forbidden (because directory listing is disabled) or a 404 Not Found. With this setting: CloudFront automatically appends index.html to any request ending in a /. This ensures your React entry point is served.
   2. One quick heads-up: Setting the Default Root Object only works for the root of the site. If a user navigates to https://your-app.com/dashboard and hits refresh, CloudFront won't automatically point that to index.html. 
3. On the "Behaviors" tab make sure the "Viewer protocol policy" is set to "Redirect HTTP to HTTPS".
   1. Standard security hygiene. It ensures all data in transit is encrypted.

Once done, you should be able to browse to your CloudFront distribution URL, you can find the "Distribution domain name" in the "Details" section after selecting your distribution. It should look like xxxxxxxxx.cloudfront.net. Put this url into your browser. You should an access denied error. This is because you have not uploaded your files.

---

### 4. Uploading your files to S3

1. Browse to the S3 console and select your bucket. 
2. Upload the contents of the build folder to the root of your bucket. 
3. Refresh the browser and you should see the app.


That's it! Not too difficult. Doing this through the console is a bit tedious, and a lot of behind-the-scenes work has been done for you. Let's now do the same thing using the AWS CLI and go into a bit more detail.

---

## Via AWS CLI


### Architecture Overview

```
User → CloudFront (CDN + HTTPS) → OAC → S3 Bucket (no public access)
```
The user is requesting a resource from CloudFront. CloudFront uses the OAC to authenticate itself to S3 and then serves the resource.

The core principle: **S3 is never publicly accessible.** Only CloudFront touches it, and only through an explicitly authorised identity. This is the most important security constraint in this entire setup.

---

### 1. Create the S3 Bucket with Restricted Public Access

#### 1.1 - Create the bucket

```bash
aws s3api create-bucket --bucket <BUCKET_NAME> --region <region> --create-bucket-configuration LocationConstraint=<region>
```

> If you're deploying to `us-east-1`, don't need `--create-bucket-configuration LocationConstraint=<region>`. The `us-east-1` region is the only one that omits this flag - an AWS quirk worth remembering.

#### 1.2 - Block all public access

Block all public access on the bucket, we'll use OAC to grant access to CloudFront.

```bash
aws s3api put-public-access-block \
  --bucket <BUCKET_NAME> \
  --public-access-block-configuration '{
    "BlockPublicAcls": true,
    "IgnorePublicAcls": true,
    "BlockPublicPolicy": true,
    "RestrictPublicBuckets": true
  }'
```

##### Why does this matter?

Each of the four public-access flags serves a distinct purpose:

| Flag                    | What it blocks                                                                                                |
|-------------------------|---------------------------------------------------------------------------------------------------------------|
| `BlockPublicAcls`       | Prevents anyone from attaching a public ACL to objects (e.g., `public-read`) via PUT Object ACL or PUT Object |
| `IgnorePublicAcls`      | Makes any *existing* public ACLs on objects ineffective - a belt-and-suspenders measure                       |
| `BlockPublicPolicy`     | Rejects any bucket policy that would grant access to `"Principal": "*"`                                       |
| `RestrictPublicBuckets` | Ensures only the bucket owner and AWS services can access the bucket, even if a policy technically allows `*` |

Together, these four flags create a **defense-in-depth** posture. Even if a misconfigured deployment script or a future IAM policy attempts to open the bucket, the block is enforced at the bucket level - independent of any policy you write. This is an account-level safety net.


#### 1.3 - Disable versioning (optional but recommended for cost control)

This is set by default, but it's good to be explicit.

```bash
aws s3api put-bucket-versioning \
  --bucket <BUCKET_NAME> \
  --versioning-configuration Status=Suspended
```


---

### 2. Set Up Origin Access Control (OAC)

OAC is CloudFront's mechanism for authenticating itself to S3. It replaced the older Origin Access Identity (OAI) and is strictly better: it supports all HTTP methods, works with S3's Server-Side Encryption (SSE-KMS), and signs requests with `AWS4-HMAC-SHA256`.

#### 2.1 - Create the OAC

```bash
aws cloudfront create-origin-access-control \
  --origin-access-control-config '{
    "Name": "my-react-app-oac",
    "Description": "OAC for my-react-app S3 origin",
    "SigningProtocol": "sigv4",
    "SigningBehavior": "always",
    "OriginAccessControlOriginType": "s3"
  }'
```

Save the `Id` from the response - you will need it when creating the CloudFront distribution.

#### 2.2 - Attach a bucket policy granting access only to CloudFront

Replace `<OAC_ID>` and `<BUCKET_NAME>` with your actual values. Retrieve your account ID first:

```bash
aws sts get-caller-identity --query Account --output text
```

> **Note:** The `Condition` block with `SourceArn` is critical. Without it, the policy grants access to *any* CloudFront distribution in *any* account that points at your bucket. With it, only your specific distribution is authorised. You'll need to come back and fill in `<DISTRIBUTION_ID>` after step 3.

#### Why OAC over a public bucket?

A public bucket relies on *you* never making a configuration mistake. OAC flips this: the bucket is locked down by default, and access is *explicitly granted* to a cryptographically signed identity. If someone discovers your S3 URL directly (e.g., via DNS enumeration or a leaked log), they hit a `403`. The only valid path to your content is through CloudFront. This eliminates an entire class of origin-exposure attacks.

---

### 3. Configure CloudFront for HTTPS

#### 3.1 - Create the distribution

```bash
aws cloudfront create-distribution \
  --distribution-config '{
    "CallerReference": "my-react-app-'$(date +%s)'",
    "Origins": {
      "Quantity": 1,
      "Items": [
        {
          "Id": "S3Origin",
          "DomainName": "<BUCKET_NAME>.s3.<REGION>.amazonaws.com",
          "S3OriginConfig": {
            "OriginAccessIdentity": ""
          },
          "OriginAccessControlId": "<OAC_ID>"
        }
      ]
    },
    "DefaultCacheBehavior": {
      "TargetOriginId": "S3Origin",
      "ViewerProtocolPolicy": "redirect-to-https",
      "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6",
      "AllowedMethods": {
        "Quantity": 2,
        "Items": ["GET", "HEAD"],
        "CachedMethods": {
          "Quantity": 2,
          "Items": ["GET", "HEAD"]
        }
      },
      "Compress": true
    },
    "CustomErrorResponses": {
      "Quantity": 1,
      "Items": [
        {
          "ErrorCode": 403,
          "ResponsePagePath": "/index.html",
          "ResponseCode": "200",
          "ErrorCachingMinTTL": 300
        }
      ]
    },
    "Comment": "React SPA hosted on S3",
    "Enabled": true,
    "DefaultRootObject": "index.html"
  }'
```

##### DefaultCacheBehavior

Why these specific HTTPS settings?

| Setting                                                     | Why                                                                                                                                            |
|-------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------|
| `ViewerProtocolPolicy: redirect-to-https`                   | Forces any `http://` request to upgrade to `https://` via a 301. Prevents cleartext credential/cookie transmission.                            |
| `CachePolicyId` (the AWS-managed `CachingOptimized` policy) | Uses a sensible default TTL and respects `Cache-Control` headers from your origin. Avoids you having to manually define forwarding rules.      |
| `Compress: true`                                            | Enables gzip/brotli at the edge. For a React bundle, this typically yields 60–80% size reduction, directly impacting TTFB and Core Web Vitals. |



##### CustomErrorResponses

The `CustomErrorResponses` block is critical. React SPAs use client-side routing. A request to `/about` hits S3 as a literal file path and returns a `404` - but you want CloudFront to serve `index.html` and let React's router handle it.

> Use `403` (not `404`) as the error code here. Because public access is blocked, S3 returns `403` for missing files, not `404`. This is a subtle but common deployment gotcha.

#### 3.2 - Allow CloudFront to read from the bucket

```bash
aws s3api put-bucket-policy \
  --bucket <BUCKET_NAME> \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "AllowCloudFrontToReadS3",
        "Effect": "Allow",
        "Principal": {
          "Service": "cloudfront.amazonaws.com"
        },
        "Action": "s3:GetObject",
        "Resource": "arn:aws:s3:::<BUCKET_NAME>/*",
        "Condition": {
          "StringEquals": {
            "AWS:SourceArn": "arn:aws:cloudfront::<ACCOUNT_ID>:distribution/<DISTRIBUTION_ID>"
          }
        }
      }
    ]
  }'
```

---

### 4. Sync Your Local Build Folder via AWS CLI

#### 4.1 - Initial sync

```bash
aws s3 sync ./build s3://<BUCKET_NAME> \
  --delete \
  --cache-control "public, max-age=31536000, immutable"
```

#### 4.2 - Separate cache strategy for `index.html`

Hashed assets (`main.abc123.js`) are immutable - they can be cached forever. But `index.html` must always be fresh so users get the latest bundle manifest. Override it:

```bash
aws s3 cp ./build/index.html s3://<BUCKET_NAME>/index.html \
  --cache-control "public, max-age=0, must-revalidate"
```

#### 4.3 - Invalidate the CloudFront cache

After a deployment, edge locations may still serve a stale `index.html`. Invalidate the cache:

```bash
aws cloudfront create-invalidation \
  --distribution-id <DISTRIBUTION_ID> \
  --paths /index.html
```

Only invalidate `index.html`. All other assets have new hashes in their filenames, so CloudFront naturally fetches the new versions on the next request. Invalidating `/*` is wasteful and incurs cost after the first 1,000 paths/month.

### Why this two-tier caching strategy?

This is the single most impactful performance pattern for SPA deployments. The logic:

1. Your bundler (Vite, webpack, etc.) outputs files like `main.a1b2c3.js`. The hash in the filename *is* the cache key. If the content changes, the hash changes, the filename changes, and it's a cache miss - which is exactly what you want.
2. `index.html` contains `<script src="main.a1b2c3.js">`. It's the manifest that points to the current hashes. If it's cached for a year, users never see new deploys.
3. By setting `max-age=0, must-revalidate` on `index.html`, the browser always revalidates it (a lightweight `If-Modified-Since` check). If it hasn't changed, the server returns `304 Not Modified` - no body transferred. If it has, the browser gets the new manifest and pulls only the changed chunks.

The result: near-instant deploys with aggressive long-term caching and no wasted bandwidth.

---

### Full Deployment Script (Putting It All Together)

```bash
#!/usr/bin/env bash
set -euo pipefail

BUCKET="<BUCKET_NAME>"
DIST_ID="<DISTRIBUTION_ID>"

echo "→ Syncing assets with long-term cache..."
aws s3 sync ./build s3://$BUCKET \
  --delete \
  --exclude "index.html" \
  --cache-control "public, max-age=31536000, immutable"

echo "→ Uploading index.html with no-cache..."
aws s3 cp ./build/index.html s3://$BUCKET/index.html \
  --cache-control "public, max-age=0, must-revalidate"

echo "→ Invalidating CloudFront cache..."
aws cloudfront create-invalidation \
  --distribution-id $DIST_ID \
  --paths /index.html

echo "✓ Deploy complete."
```

---

### Post-Deployment Verification Checklist

- [ ] `curl -I https://<your-cloudfront-domain>` - confirm `HTTP/2 200` and `x-cache: Miss` on first hit, `Hit` on second.
- [ ] `curl -I http://<your-cloudfront-domain>` - confirm `301` redirect to HTTPS.
- [ ] `curl -I https://<your-s3-bucket-url>/index.html` - confirm `403`. The origin must never be directly reachable.
- [ ] Navigate to a client-side route (e.g., `/about`) - confirm it renders correctly (the 403→200 error page rewrite is working).
- [ ] Check `Cache-Control` headers on a hashed asset vs. `index.html` - confirm the two-tier strategy is in effect.

---

### Security Model Summary

| Layer                         | Control                                 | Threat Mitigated                                         |
|-------------------------------|-----------------------------------------|----------------------------------------------------------|
| S3 Public Access Block        | All four flags enabled                  | Accidental public exposure                               |
| Bucket Policy + OAC Condition | `SourceArn` scoped to your distribution | Origin accessed by unauthorised CloudFront distributions |
| CloudFront Viewer Policy      | `redirect-to-https`                     | Man-in-the-middle on cleartext HTTP                      |
| HTTPS                         | TLS 1.2+ enforced                       | Credential/session interception                          |
| OAC (sigv4, always)           | Request signing on every origin call    | Unsigned/forged requests to S3                           |
