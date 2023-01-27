---
layout: post
title:  "Key AWS Services for Python API Development"
date:   2022-12-27 11:44:08 +0000
---

AWS offers over 200 different services, making it overwhelming for people new to it to know where to start when building something like a serverless Python API.

However, understanding just 12 key services can make the process much simpler. These services include:

1. CloudFormation: This service allows you to use templates to provision and manage your AWS infrastructure (IaC). The docs for this are great.
2. IAM (Identity and Access Management): IAM allows you to create and manage users, groups, and permissions for your AWS resources. This is what allows you to control who (and what) can access your resources.
3. Lambda: Lambda is a serverless compute service that allows you to run code without provisioning or managing servers. It's where your code lives, executes (and scales).
4. API Gateway: API Gateway is a fully managed service that makes it easy to create, publish, maintain, and secure APIs. This is your endpoints, i.e. `apigateway.com/your/api/endpoint` - requests made to this endpoint will be routed to your Lambda function.
5. S3 (Simple Storage Service): S3 is an object storage service that allows you to store and retrieve data. This is where any static files (e.g. images, videos, etc.) will be stored
6. CloudWatch: CloudWatch allows you to monitor your AWS resources and applications. This is your logging, your alarms for when things go wrong, your monitoring of your Lambda functions, etc. Your eyes into your application.
7. Secrets Manager: Secrets Manager is a service that helps you protect access to your applications, services, and IT resources. This is where you'll store any secrets (e.g. API keys, passwords, etc.) that you don't want to be stored in your code.
8. CodePipeline: CodePipeline is a service that allows you to automate your release process. This is where you'll define your CI/CD pipeline (e.g. when to run tests, when to deploy, etc.).
9. CodeBuild: CodeBuild is a service that allows you to build, test and deploy your code. This is where you'll define and run your build process (e.g. running tests, linting, etc.).
10. Route53: Route53 is a service that allows you to route traffic to your application. This is where you'll define your DNS records (e.g. `yourdomain.com` -> `apigateway.com/your/api/endpoint`).
11. EC2 (Security Groups): EC2 security groups are a way to control inbound and outbound traffic for your instances. This is where you'll define your firewall rules i.e which functiosn can access your database.
12. VPC (Virtual Private Cloud): VPC allows you to provision a logically-isolated section of the AWS Cloud where you can launch AWS resources in a virtual network. This is where you'll define your network (e.g. subnets, routing tables, etc.).
13. DynamoDB (or RDS): DynamoDB is a NoSQL document-based database service while RDS is a relational database service, both are fully managed services that allows you to create and manage databases.

By understanding these 12 services (have a flick through the service FAQs - that's always a great place to start), you'll be well on your way to building a serverless Python API in AWS. Keep in mind that as you build your applications and gain more experience, you'll likely find that you need to use other services as well.
