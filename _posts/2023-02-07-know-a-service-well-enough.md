---
layout: post
title: "AWS Insight: Know the service you're using well enough"
date: 2023-02-07 11:44:08 +0000
# categories: aws, services, insight
---

AWS offers over 200 different services, making it overwhelming for people new to it to know where to start when building web applications.

If you're going to use a service, I strongly believe you should know it well enough to be able to answer the following 10 questions:

1. What is it?
2. What does it do?
3. How do I use it?
4. What are the limitations?
5. What are the costs?
6. What are the alternatives?
7. How do I monitor this service?
8. What are the key metrics?
9. How do I makesure it's secure?
10. What does this service look like in IAC?

Getting the anwers to these questions is simple enough, but does require a little bit of effort.

To be clear: I am not saying that you shouldn't use the service if you do not know the answers (however, if you don't know the answer to questions 1 and 5, then I would be very careful). What I am saying here is that the better you understand the service, the better you'll utilise it's capabilities, the more you'll get out of it and less you'll spend on it.

So - how are you going to answer these questions? Here's what I do:

## What is it?

Someones just mentioned this new (...to you) service and you've totally agreed with them that it sounds like the best solution to your problem. They end the call and you're left thinking "what the hell did I just agree to?" - now what? First - go and read the FAQ. This is nearly always the best place to start, it'll tell you exactly what it is and what it isn't. Skim it, read all the questions first then go back and read all the answers, start to digest what they are saying about the service slowly. 

## What does it do?

Go and watch a youtube video about the service - there will be hundreds. Find a channel you like that describes the service in a way you understand.

Now - go and use it. Try out it's most simple features, and do it all in the console. Be mindful of the drop downs, the options, what you can and can't do with it - are there things missing from the setup or creation flows that you thought would be there? Good! you're learning. Your assumptions are being broken down one by one. You're learning what the service can and can't do. You're learning what it's limitations are.

This isn't the last time you're going to use the service in the console, but realise that when you're actually using this service for real, you're going to be using infrastructer as code - Cloudformation, CDK, SAM etc.

## How do I use it?

The next step is to dig deeper and start to understand how to use the service. This is where the official AWS documentation comes in handy. Start with the user guides and tutorials, they will give you step-by-step instructions on how to use the service. Read through the documentation carefully, making sure you understand each step before moving on to the next one.

## What are the limitations?

Once you have a basic understanding of how to use the service, it's time to look at the limitations. This is where you should pay close attention to the fine print. Read the AWS documentation and look for sections that mention limitations or restrictions. If you're unsure about anything, don't hesitate to ask in the AWS forums.

## What are the costs?

The costs of using an AWS service can vary greatly, so it's important to understand what you're getting into. Start by using the AWS pricing calculator to get an estimate of the costs for your specific use case. Then, keep an eye on your costs using the AWS billing dashboard. Make sure you understand what you're paying for and why.

## What are the alternatives?

Before you commit to using an AWS service, it's always a good idea to look at the alternatives. Research other solutions that offer similar features and compare their costs, limitations and ease of use. This will help you make an informed decision and ensure you're using the right solution for your needs.

## How do I monitor this service?

Monitoring an AWS service is critical to ensure it's running smoothly and efficiently. Start by setting up monitoring and logging using AWS services like CloudWatch and CloudTrail. Make sure you understand what metrics are important and how to view them.

## What are the key metrics?

The key metrics will vary depending on the service you're using, but some common metrics to keep an eye on include performance, error rates, resource utilization and costs. Make sure you understand what each metric means and how it affects your service.

## How do I makesure it's secure?

Security is a top priority for AWS and they offer a wide range of security features and services. Start by reading the AWS security documentation and make sure you understand how to secure your service. Pay close attention to the recommendations for configuring and managing access control, encryption and network security.

## What does this service look like in IAC?

Infrastructure as code (IAC) is a best practice for managing AWS services. Look for examples of how to use this service with IAC tools like CloudFormation, CDK and SAM. Make sure you understand how to provision and manage the service using code, rather than manual configuration.

---

Understanding an AWS service requires a little bit of effort but the benefits are well worth it. The better you understand the service, the better you'll be able to utilize its capabilities and get the most out of it. So, take the time to answer these 10 questions and make sure you're using the right service for your needs.
