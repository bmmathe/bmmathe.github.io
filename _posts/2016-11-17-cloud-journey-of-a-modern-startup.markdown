---
title:  "Cloud Journey of a Modern start-up"
date:   2016-11-14 18:20:36
categories: cloud
---
A couple months ago a colleague and I decided to work on an off-hours project to gain more skills in non-Microsoft technologies.  We wanted to branch out and see what it's like out there for start-ups.  Being on a budget and completely self-funded we wanted to find a cloud provider that cost little to nothing to operate our web site.  We decided to use common start-up technologies like React, node, Linux, and chose to use a no-SQL document database.  This is a far departure from our typical Enterprise stack which consists of .Net services, ASP.Net web sites, and SQL Server partly deployed on-prem and partly in Azure.

I want to compare and contrast the different services we explored over the past month or so and share where we landed.

For the Enterprise Azure made perfect sense.  Every developer at this shop has C#, ASP.Net, and SQL experience.  Azure credits are built into our Enterprise License Agreement with Microsoft which pretty much locked us into their platform.  As I began to delve into Azure service offerings, I was pleasantly surprised at how easy it was to spin up a new web site or web service in the cloud.  I can deploy directly from Visual Studio to the cloud without even logging into the Azure Portal.  Even if I wanted to create a more advanced deployment scenario leveraging resource groups, it was still very easy to get things up and running.  We went from a few days for different internal departments to stand up a new resource to only a few seconds in the cloud.  This was liberating.  Not to mention we have pretty much every common, best-practice technology within a button push away.  We were able to experiment with pub/sub using Azure Service Bus, Docker container, and even Linux (shhhhh). However, when it came down to choosing a service for a start-up with no money, I wanted to leverage a serverless offering that required little to no maintenance.

### Container Services
I originally thought it would be cool to host Docker containers for my web site, API, and database but as I started looking most cloud provider offerings allow you to create a cluster of hosts that you deploy your container to.  This is too expensive for what I wanted so I decided to focus solely on managed PaaS services.

### Requirements and Considerations
- Cheap as possible, less than a gym membership for hosting a site with little to no traffic including web site, API, and database
- Serverless, I don't want to mainain any server configuration or patching environments
- No platform lock, I didn't want to build something that locks me into a platform
- Features, the provider needs to offer a wide array of services a start-up needs such as web hosting, email services, notifications (SMS), database, caching, queueing, etc.
- Custom domain support and HTTPS

### Microsoft Azure

Azure's serverless PaaS offering is App Services.  Their free offering offers 60 minutes CPU time per day, 1GB RAM, 1GB, but no custom domain support, no SLA, no auto-scale.  The free plan isn't really feasible for offering a site that needs to run 24x7.  The basic plan offers 1 core, 1.75GB RAM, 10GB storage, custom domain, 99.9% update but no auto-scaling and it's $56/mo.  That is too much for what I wanted to spend for one instance.      

### Google Cloud Platform

I was excited about GCP because they seem like the underdog compared to Azure and AWS. I was really hoping their pricing would be super competitive and I've heard great things about App Engine especially from starups.  For new customers Google offers a $300 credit for signing up.  [App Engine](https://cloud.google.com/appengine/) is a PaaS offering which is basically offers a container about $36/mo plus other benefits like auto-scaling other features you can use for a small price like No-SQL database, memcache, search, logging and user authentication.

The cool thing with App Engine is you get 28 instance hours for free per day then the $0.05/hr ($36/month) price kicks in.  I'm still a little confused about what an "instance" is.  If for example an instance is a single microservice then hosting an API and a web site will put me over the 28 instance hours per day (2 instances / day = 48 hours per day). At the bottom of [this page](https://cloud.google.com/appengine/docs/flexible/nodejs/an-overview-of-app-engine) it reads a Free app an have a maximum of 5 services per app.  I plan on experimenting more with App Engine in the future but because I was confused about the pricing I forwent this service.  

### AWS
AWS seems to be the defacto standard for start-ups.  Ultimately this is the service I chose.  

Here is a list of services we chose to use for the first iteration of AWS
- S3 for static web site hosting (html, Javascript, css)
- Cloudfront mainly for CDN support
- Lambda and API Gateway for our serverless API
- DynamoDB no-SQL key-value database
- Route53 DNS manager

For our needs this is a perfect serverless stack that seems to be feature rich and very flexible.  The most costly service in this setup is the CDN which may fall under the 12 months free deal since I haven't been charged yet. Unfortunately none of the options we looked at supported static web site hosting with a custom domain and HTTPS support.  If you don't need a custom domain then you can host directly out of the bucket using HTTPS without having to use the CDN service.  

#### Setup and Configuration

##### S3 

Simple drag and drop files in the browser. Couldn't be easier.

##### Cloudfront and Route53.  

Since my background is development I don't have a lot of experience setting up DNS entries so it took me a while to figure out how to redirect www requests to my apex domain name.  Reference [this article](http://docs.aws.amazon.com/AmazonS3/latest/dev/website-hosting-custom-domain-walkthrough.html).  It seemed a little cumbersome for something that seems so simple.  

##### Deployment

Deployment is something else. After a deployment to your S3 bucket your web site does not update automatically because the CDN caches your site for a period of time (24 hours by default).  You can invalidate your cache easily enough but you only get a fixed number of invalidations per month before you are charged for it.  The recommended approach is to version your files which will invalidate the cache for that particular file.  The problem I ran into is that the index.html file (Root Object) can not be versioned.  After several hours of research I think I finally found the answer which is to set the cache header for the file in the S3 bucket but I never got around to experimenting with it before we left AWS.

##### Lambda and API Gateway  

Amazon provides a few common recipes that you can choose from when configuring your Lambda function.  I chose the recipe that exposes Lambda functions as a microservice API to my DynamoDB database.  There were a few fields I had to fill out in the wizard and it spun everything up for me.  The only issue I had was enabling CORS.  I followed some instructions that showed me how to enable CORS, a simple button in Actions for my function however it didn't explain that you need to deploy your changes!  I spent an embarrassing amount of time hunting down this issue until I realized you have to deploy your saved changes.

##### DynamoDB  

Ok, I'm new to no-SQL so I didn't plan out all my queries ahead of time which caused me to make a lot of changes to how my partition and sort keys were setup.  Not to mention I still don't understand when I should create indexes versus just creating another table.  The main problem with DynamoDB is the ungodly query language.  You don't really query the database actually, you write "expressions" in a json payload.  It took me days to get all my inserts, updates, and reads configured properly.  Something that would take me 10 minutes to do in SQL took hours of reading documentation, Googling, and just experimenting.  Granted once I spent that time I can write queries much faster now and have a better understanding of no-SQL data strategies in general.  I recommend watching the DynamoDB videos [here](https://aws.amazon.com/dynamodb/getting-started/).

Once I got everything working pretty much how I wanted... we left AWS to try a Google offering.

### Google Firebase
My colleague heard of a Google offering called Firebase from a friend and knowing that I had struggled with DynamoDB and was still struggling with deployments, he suggested we try Firebase which touted button push, end-to-end secure web site deployments. Not only was it supposed to be super simple to setup, it is free until you reach a certain user base.  The claim to fame for Firebase is their Realtime Database.  Originally designed for mobile backends, it allows developers to listen for changes in the database without having to code this complexity yourself.  You essentially setup your evens and you're done.  And it was that easy to get going.  What took me days to get going in AWS took mere minutes using Firebase.  For the first few days I was upset that I had spent so much time in AWS trying to figure out all there many services.  Firebase has everything I need right there in one dashboard.  I use one SDK in my client up to access the database, storage, and auth.  

##### Auth
Auth was amazingly simple!  I just flip a switch, add a few lines to my client code, and BAM! I'm using OAuth to authenticate users from third-parties.  I didn't even have to write a line of code to store the user information in my database, it does that for you.  Not to mention automatically signing the user back in when they visit the site again.  

##### Realtime Database
But, some things are too good to be true.  Their database is pretty amazing if you truly need realtime updates.  However, my web site does not need realtime updates.  In fact realtime updates for my app would end up confusing and possibly causing a bad user experience for my users.  Since I'm using React and Redux, I don't want state changes to interrupt what the user is doing.  I didn't think this would be a problem first.  You are not forced to use the realtime feature.  You can do what is called a "once" operation which does not hook up a listener to the table.  This is where things started falling apart.  Realtime database does not support atomic operations on the server.  So if you want to increment a counter you need to create a transaction that reads the data first then issues an update.  This shouldn't be a big deal except I couldn't figure out how to do it in "once" mode.  I spent hours trying to figure out how to get one value updated in a transaction (and still haven't figured it out).  This made me question, do I really want to struggle with this realtime database if I'm not using the primary feature... realtime.  The answer for me is no, I don't want to use a database I will end up fighting because my use case isn't what this database was designed for. Time to find a database that fits my needs.  

### Final Solution: Back to AWS (ROUND 2)
We spent another 3 days reviewing our options again after deciding to leave Firebase.  Here is what we ended up with.

- AWS Elastic Beanstalk for the API, a container host where you don't own the cluster
- DynamoDB
- S3, Cloudfront, Route 53, Certificate Manager for the web site
- Lambda, for DynamoDB stream tasks

AWS Elastic Beanstalk offers 750 hours per month free for certain instance types which is perfect for us.  Even after you use up the free time, the nano tiers are less than $5/mo to operate (assuming I understand the pricing). Elastic Beanstalk was easy to get up and running using their CLI and git integration.  I would prefer to host the web site in Elastic Beanstalk as well for isomorphic server-side page preloading but S3 and Cloudfront is cheaper in the 12 month free plan.  We still have some more experimenting to do but I feel pretty happy where we landed and AWS has a starup program called [AWS Spark](https://aws.amazon.com/start-ups/) I'm looking to take advantage of.

I'm planning to do a follow-up post after a few weeks to share my experience in AWS-land.