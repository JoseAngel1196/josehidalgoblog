---
title: "Application performance using Amazon Elasticache"
description: "My approach to improve application performance using Amazon Elasticache #CloudGuruChallenge"
date: 2022-05-01T14:10:09-04:00
draft: false
cover:
    image: "https://res.cloudinary.com/acloud-guru/image/fetch/c_thumb,f_auto,q_auto,w_1200/https://acg-wordpress-content-production.s3.us-west-2.amazonaws.com/app/uploads/2020/09/CloudGuruChallenge-min.jpg"
    alt: "#CloudGuruChallenge"
    caption: "#CloudGuruChallenge"
tags: ["cloudskills", "terraform", "redis", "python"]
---

On June 7, 2021, [A Cloud Guru](https://acloudguru.com) released a challenge about improving the performance of an application. After a very bumpy ride, I was able to complete this challenge. One of the things that made me work on this challenge was the need to learn terraform, so I found existing building the infrastructure of this project using terraform.

Check it out! Let's connect?
[Github](https://github.com/JoseAngel1196/elastic-cache-challenge) 
[LinkedIn](https://www.linkedin.com/in/jose-hidalgo-rosa/) 

## My learnings

When I started this challenge, there were a couple of things that I didn't know about AWS and one of the reasons that made me work on this. One of them was that I thought that it was possible to access a Redis cluster from the public web, after some investigation I quickly realized that is not safe to expose my Redis to the internet. I have to say that doing the networking on AWS was challenging and is something that I will spend more time practicting now that I successfully completed this project from the ground up, creating a public VPC that allows my resources to connect to that VPC and all subsequent resources that I wanted. It took me around 4 days without sleep to set up the whole infrastructure on AWS, I cannot remember how many times I had to run: `terraform destroy` to start again because there were some misconfigurations on the server.

## My approach to the challenge

Before I even started the challenge, I had an idea of what was terraform but did not know where to start. I saw a course on [Udemy](https://www.udemy.com/course/terraform-certified/) that gave me a good understanding on how to start this challenge using terraform. I was very eager to start this project and after having a good knowledge of terraform and watching the course, I moved up to create my first EC2 Instance. The things I learned on terraform are how I can reuse the same line of code called modules to create multiple resources on AWS and the way I can share variables across the code. I found this very interesting and just could stop myself from learning it.

## Why I took the challenge?

At my workplace terraform is what we use to deploy our infrastructure to AWS and I want to keep leveling up on my career by expanding my knowledge and taking on challenges that put me to test.

## The Project

Once I was able to set up my infrastructure on terraform and seeing I was able to connect to my EC2 and ping to my RDS ad Redis without any problem. The only thing I was missing was installing nginx on my server. I chose the amazon linux ami that use centos. I had to investigate how to install nginx on centos as it was my first-time using centos and I did not know that I had to use yum instead of apt, the good thing was that the set up was not complicated at all. After having all the set up in place, I had to make some modifications to the code to cache our query. The result was:

![redis-app](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/9c9s0nvfu88k50w3fzwf.png)
 
Having that in place, once we visit the application and because there is a timer in the sql function, the application takes some time to get the response from the server and including the time to connect to the database and return it back to the server. 

![slow-application](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/9fxksh3m5kkyhja3z9gp.png)

As we can see on the image above, it takes around 5sec to get the results back to the server, which is a long time, if we compare it by caching the result with redis, time would be around: 0.023sec the timedelta is a huge difference:

![fast-application](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/uvb21wxpx7sssed5mb6p.png)

## Conclusion

I want to thank #acloudguru for this amazing challenge and I am really waiting to see more like this on the following months. I always appreciate any and all feedback.

Thanks for reading! 😀