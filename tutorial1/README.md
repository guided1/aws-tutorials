# Hosting a Static React App on AWS: S3 + CloudFront


## Prerequisites for this tutorial

1. An AWS account with CLI access configured (aws configure completed)
2. IAM credentials with permissions for S3 and CloudFront
3. A React build output in a local /build folder
4. A domain name (optional, but assumed for the HTTPS section)

In the repository, we have a React app that we want to host on AWS using S3 and CloudFront.

The first thing we need to do is to build the app. We can do this by running the following commands:

`npm install`

`npm run build`

This will have createda a `build` folder in the root directory of the project.


## By hand

First, let's do the process by hand so we can get a feel for what is happening.

### Create the S3 Bucket (Private by Design)

First, create your storage container.

1. Browse to the S3 console.
2. Create a bucket (e.g. tutorial1).
   1. Note the bucket name has to be unique across all buckets created on earth.
   2. Make sure you keep a note of the bucket name you will need it later
3. Keep "Block all public access" checked. 
   1. Note: In a production environment, you never want your S3 endpoint exposed. By blocking public access, you prevent users from bypassing your CDN (CloudFront), which would otherwise lead to higher S3 egress costs and bypass your security headers or Web Application Firewall (WAF) rules.

Make a note of the distribution ID, you will need it later.



### Setup CloudFront & Origin Access Control (OAC)

1. Browse to the CloudFront console.
1. Create a new distribution.
2. Select your plan
3. Provide a distribution name and description
4. Select the origin type to be S3
5. Select the bucket you created earlier
6. Use the default settings for the rest of the fields.



### Configure Cloudfront 

React apps are Single Page Applications (SPAs), so we need to handle the "client-side routing" problem where refreshing a sub-page (like /dashboard) throws a 403/404 from S3.

1. Select the distribution you created earlier.
2. On the "General" tab, click "Edit" and set the "Default Root Object" to "index.html". 
   3. By default, CloudFront is just a pass-through. If a user hits https://your-app.com/, CloudFront sends a request to S3 for the "root" of the bucket. Without this setting: S3 doesn't know what to return for a "folder" request (since S3 is a flat key-value store, not a real file system). It will either return a 403 Forbidden (because directory listing is disabled) or a 404 Not Found. With this setting: CloudFront automatically appends index.html to any request ending in a /. This ensures your React entry point is served.
   4. One quick heads-up: Setting the Default Root Object only works for the root of the site. If a user navigates to https://your-app.com/dashboard and hits refresh, CloudFront won't automatically point that to index.html. 

2. On the "Behaviors" tab make sure the "Viewer protocol policy" is set to "Redirect HTTP to HTTPS".
   3. Standard security hygiene. It ensures all data in transit is encrypted.

Once done you should be able to browse to your CloudFront distribution URL, you can find the "Distribution domain name" in the "Details" section after selecting your distribution. It should look like xxxxxxxxx.cloudfront.net. Put this url into your browser. You should an access denied error. This is because you have not uploaded your files.

### Uploading your files to S3

1. Browse to the S3 console and select your bucket. 
2. Upload the contents of the build folder to the root of your bucket. 
3. Refresh the browser and you should see the app.


That's it! Not too difficult. Doing this through the console is a bit tedious, and a lot of behind-the-scenes work has been done for you. Let's now dow the same thing using the AWS CLI and go into a bit more detail.

## By cli

What is actually happening here with the user browses to the cloudfront distribution URL?

### Architecture Overview

User → CloudFront (CDN + HTTPS) → OAC → S3 Bucket (no public access)

The core principle: S3 is never publicly accessible. Only CloudFront touches it, and only through an explicitly authorized identity. This is the single most important security constraint in this entire setup. This is what the OAC is for.


