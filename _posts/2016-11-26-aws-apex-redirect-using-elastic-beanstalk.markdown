---
title:  "AWS Redirect from WWW to Apex using Elastic Beanstalk"
date:   2016-11-14 18:20:36
categories: aws
---
If all you need to do is redirect your www subdomain to your "naked" or apex domain *and* you can create exact matching buckets in Amazon then [this](http://docs.aws.amazon.com/AmazonS3/latest/dev/website-hosting-custom-domain-walkthrough.html) documentation is all you need. However, my situation is unique.  Someone had already created S3 buckets with the name of my domain.  Both the apex domain and the www subdomain were not available for me to create.  To create the redirect using the method linked above you must be able to create S3 buckets using the URL exactly matching.  For example you need to create www.example.com and example.com.  Amazon will not release those bucket names if they are already taken which will leave you in a bind.

# Attempted Solutions

I didn't want to change hosting my static web site from S3 and CloudFront so I had to find a better solution to redirect www to my apex domain.  I was able to continue hosting my apex domain statically using S3 simply by using CloudFront.  You do not need an exact bucket name match to use CloudFront.  The problem was with the redirect S3 bucket from www to apex.  When I set this up using CloudFront I received an Unauthorized response when I browsed to www.domain.com.  No matter what permissions I gave my bucket this simply would not work. 

The solution is to use a web server to rewrite the URL.  I was already using Elastic Beanstalk (EBS) for my API so I decided to leverage that existing application to not only be my API but also serve the redirect.   

# Final Solution

The solution isn't elegant but it works.  Use my nginx server in Elastic Beanstalk to rewrite the www.domain.com address to domain.com.  No being familiar with nginx or Elastic Beanstalk this took me hours to figure out (which is why I'm posting this).

### Update DNS using Route 53

The first step is to update your DNS settings to route all requests to www.domain.com to your Elastic Beanstalk instance using an A record Alias.  This is simple, when you select A and "alias: yes" you will see your EBS instance in the dropdown.  Select it.

### NGINX configuration

Next, I had to get the NGINX configuration in my EBS solution.  To do this I setup SSH so I could login to one of my instances and copy out the EBS conf in the conf.d directory.  The idea was to replace the default EBS conf with my custom version.
Rewrite rules in nginx are well-documented [here](https://www.nginx.com/blog/creating-nginx-rewrite-rules/).  EBS uses a proxy to forward all requests to node which complicated things for me.  I didn't know how to incorporate server blocks to isolate just www requests to my web site so I opted for a route I knew would work using condition logic to parse the request and rewrite if the address was an exactl match on www.domain.com.  After a few minutes of Googling this is what I came up with:

{% highlight json %}
  ...
  location / {
    if ($host ~* www\.domain\.io) {
          return 301 https://domain.io$request_uri;
        }
        proxy_pass  http://nodejs;
        proxy_set_header   Connection "";
        proxy_http_version 1.1;
        proxy_set_header        Host            $host;
        proxy_set_header        X-Real-IP       $remote_addr;
        proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
    }
  ...
{% endhighlight %}

I know this isn't perfect and I could probably use server_name to get this done but I had already spent hours researching this and I knew this would work.  I directly modified the file 00_elastic_beanstalk_proxy.conf to add my rewrite rule and it worked fine.  The problem with this is that when you redeploy your EBS application, your updates will be overwritten.  To solve this we need to create a custom configuration file in your .ebextensions folder to handle the replace of the default conf file before it is copied to its production location.   

### Update your EBS source
If you are working from a sample application as your starting point, you don't have a .ebextension folder, you only have a .elasticbeanstalk folder.  *These folders are not the same.* Create a config file in the newly created .ebextensions folder.  It can be named anything, I chose 0001_copy_nginx_config.config.  What we want this config to do is pull down our custom nginx config file, replace the default config file in the staging directory before it is copied to production.

I found [this](http://stackoverflow.com/questions/23709841/how-to-change-nginx-config-in-amazon-elastic-beanstalk-running-a-docker-instance) post on Stackoverflow that pointed me in the right direction.
{% highlight yaml %}
commands:
  01-get-nginx-conf-file:
    command: aws s3 cp s3://{your0-s3-bucket}}/custom-nginx.conf /home/ec2-user

container_commands:
  01-replace-default-nginx-config:
    command: mv -f /home/ec2-user/custom-nginx.conf /tmp/deployment/config/#etc#nginx#conf.d#00_elastic_beanstalk_proxy.conf
{% endhighlight %}

Granted I could have accomplished this inline but I wanted to use the solution exactly as it was posted to minimize mistakes.  To do this I created a new S3 bucket and uploaded my custom conf file to it then granted the AWS permissions to Allow GetObject.  After I deployed this setup I verified my changes were still in place after an eb deploy and they were!  I was also able to hit my domain at www.domain.com and it was redirected to my apex site. 

A few notes about the script: ec2-user is a default user AWS creates when it stands up an EC2 environment for EBS. /tmp/deploy/config is a staging directory EBS uses before files are copied to their production locations. #etc#nginx#conf.d#00_elastic_beanstalk_proxy.conf is the conf.d file that is copied to /etc/nginx/conf.d/00_elastic_beanstalk_proxy.conf. 

# HTTPS Considerations

One thing to note, I am using a wildcard and apex certificate to host my web site and my API therefore https://www.domain.com redirecting to https://domain.com worked with no errors. My API is hosted as a subdomain at api.domain.com. I'm not sure if this is required but I figured I would mention it just in case you get warnings if you are not using the same cert.

Hopefully this will help someone out that is in my situation where someone took your domain name buckets which causes a huge pain if you are planning on hosting your site out of S3.
