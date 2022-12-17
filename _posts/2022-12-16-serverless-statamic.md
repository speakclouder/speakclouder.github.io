---
layout: post
title:  "Serverless Laravel Statamic using CDK"
date:   2022-12-16 11:44:08 +0000
categories: serverless, laravel, lambda, AWS, CDK
---

## Imagine if you could have a markdown blog, that's based on Laravel, that's entirely serverless... That's the dream right?

This post describes how I used AWS CDK (Python) to create a serverless Statamic app.

"Dynamic" content is served by a Lambda function (using Bref) over an API Gateway HTTPApi, static content (images, css & js) is served by Cloudfront. It's got some fairly decent monitoring on it too.

## Why?

I like writing in markdown. Perhaps it's not as pretty as some editors, and there is a ton of syntax I can never remember - but still, it's lightweight and quick.

I like Laravel, it's probably the best documented MVC out there, plus the fact it's popular as all hell means that [stackoverflow](https://stackoverflow.com/questions/tagged/laravel) definitely has 5 or 6 answers for that weird issue you've been having.

I like serverless, it means I don't need to run updates on servers or worry that things are going to go down at 3am - my cloud provider (AWS) can handle that.

So imagine if I could have a markdown blog, that is based on Laravel, that is entirely serverless? That's the dream right?

## Prerequisites

* A decent terminal (I am using [Warp](https://www.warp.dev/) right now and it's cool)
* A decent IDE/Text Editor (I am using VSCode + [CoPilot](https://copilot.github.com/) and I don't know how I survived without it)
* You have [AWS CDK](https://aws.amazon.com/cdk/) installed
* You have [Docker desktop](https://www.docker.com/products/docker-desktop/) installed and running
* You have [Laravel](https://laravel.com/) (valet or sail)
* PHP and a little Python experience
* You have an AWS account you can mess around with - don't worry, it's [cheap as chips](https://aws.amazon.com/lambda/pricing/)!

> Right..... That's the ramble done, let's get down to business.

## Steps

Install [Statamic](https://statamic.dev/) and kick off a new project using the least offensive theme there is...

```shell
composer global require statamic/cli
statamic new laravel-statamic-serverless
```

This is going to give you a ton of prompts about the install, here's what I did:

```shell
1: Starter Kit # Because blank slates are no fun
Name of starter kit: statamic/starter-kit-cool-writings
Create Superuser: no # we're good for now
```

It looks like this:

![Statamic Install](/assets/img/statamic-install.png)

We're then going to do some Bref specific stuff, so run these commands in your terminal:

```shell
cd laravel-statamic-serverless # Let's go into the Statamic app we've just created
composer require bref/bref bref/laravel-bridge
touch .dockerignore
touch Dockerfile
```

Your `.dockerignore` file should look like this:

```.dockerignore
./cdk
```

Your `Dockerfile` should look like this:

```Dockerfile
FROM bref/php-81-fpm

ADD  . /var/task

CMD ["public/index.php"]
```

Let's get this into Git, in your terminal:

```shell
git init
git add .
git commit -m 'initial commit'
```

Before we get to the good stuff - I can't find a way of telling [Stache (statamics caching)](https://statamic.dev/stache) to respect the Lambda storage path (`/tmp/storage`), **if you can figure it out, please let me know!**

Change Stache locks to false here in the Statamic config file `./config/statamic/stache.php`

```php
...
    /**

    | https://statamic.dev/stache#locks
    |
    */

    'lock' => [
        'enabled' => false,
        'timeout' => 30,
    ],
```

Almost done here, let's build the styles and JS:

```shell
npm i
npm run prod
```

OK, one last thing before we move onto CDK - Because we're running this in a Lambda function we'll need to change the storage directory.

Inside your AppServiceProvider, let's force it to use the writable `/tmp` directory - let's also set the APP_URL as the root url - otherwise Cloudfront and API Gateway start arguing about who's who - in `./app/Providers/AppServiceProvider.php`'s boot function:

```php
    public function boot()
    {
        app()->useStoragePath("/tmp/storage");

        url()->forceRootUrl(env('APP_URL'));

        if (! is_dir(config('view.compiled'))) { 
            mkdir(config('view.compiled'), 0755, true); 
        }

        if (! is_dir("/tmp/storage/framework/cache")) {
            mkdir("/tmp/storage/framework/cache", 0777, true);
        }
    }
```

## Local test

Check your site works locally, if you're running valet, try [http://laravel-statamic-serverless.test/](http://laravel-statamic-serverless.test/)

You should get something like this:

![Local](/assets/img/local.png)

## CDK Part One

In your terminal, let's init a new CDK instance:

```shell
mkdir cdk && cd cdk
cdk init app -l=python
```

Install a couple of dependencies that we'll need down the line, update your `./cdk/requirements.txt` so it looks like this:

```txt
aws-cdk-lib==2.26.0
constructs>=10.0.0,<11.0.0
aws-cdk.aws-apigatewayv2-integrations-alpha
aws-cdk.aws-apigatewayv2-alpha
cdk-monitoring-constructs
```

**NOTE: Notice how I don't pin down these additional requirements to a specific version - I probably should be doing, but I haven't. Don't be like me, be better - pin yours down to the latest version**

Pop into your virtual environment in python and install these requirements (you'll need to be in the `./cdk` directory in your terminal to do this):

```shell
source .venv/bin/activate
pip install -r requirements.txt -r requirements-dev.txt
```

Nice, we're installed and our virtual env is activated.

Open up the CDK stack that's been generated for us, and let's get the bare bones in place:

_File: `./cdk/cdk/cdk_stack.py`_

```python
from aws_cdk.aws_apigatewayv2_integrations_alpha import HttpLambdaIntegration
from cdk_monitoring_constructs import MonitoringFacade
from aws_cdk.aws_apigatewayv2_alpha import HttpApi
from aws_cdk import (
    CfnOutput,
    Duration,
    Stack,
    aws_lambda as _lambda,
    aws_s3 as s3,
)
from constructs import Construct


class CdkStack(Stack):

    def __init__(self, scope: Construct, construct_id: str, **kwargs) -> None:
        super().__init__(scope, construct_id, **kwargs)

        # We create a Docker Image function to get around the 250mb limit
        # This is where Laravel lives
        laravel_web = _lambda.DockerImageFunction(self, "LaravelWeb",
            code=_lambda.DockerImageCode.from_image_asset("../"),
            memory_size=1024,
            timeout=Duration.seconds(20),
        )

        # This is integrating our Lambda function with our HTTP Api
        web_integration = HttpLambdaIntegration("LaravelWebIntegration", laravel_web)

        # This creates our HttpApi and sets the integration we've just created above
        endpoint = HttpApi(self, "ApiGateway",
            default_integration=web_integration
        )

        # Let's add some monitoring around our resources!
        monitoring = MonitoringFacade(self, "LaravelServerlessMonitoringFacade")
        monitoring.add_large_header("Serverless Laravel Blog")
        monitoring.monitor_api_gateway_v2_http_api(api=endpoint)
        monitoring.monitor_lambda_function(lambda_function=laravel_web)

        # This will output into the terminal after a successful deploy
        CfnOutput(self, "ApiGatewayURL", value=endpoint.url)
```

Here we're creating the Docker Image Lambda function and pointing to our `Dockerfile` (the `.dockerignore` file will stop CDK trying to include the `cdk` directory, stopping it from getting stuck in a loop).

CDK will create the ECR for us and upload the image to it, which is awesome.

We then create the API Gateway HTTP API and attach it to the function, any calls to this API (to any endpoint) will be forwarded to it.

We then add a little bit of monitoring around the API and function, so we can have some insight into how things are running.

Let's deploy this and see what we get:

```shell
cdk deploy 
```

Say yes to the security prompt (it's making a couple of roles and giving those roles least priviledge access - i.e. API Gateway can invoke your Lambda) - you'll now have an API and Lambda function running. Once complete, the console should output your API's endpoint. Let's visit it:

![No Style](/assets/img/statamic-no-styles.png)

Would you look at that. It's like, kinda working huh. The thing is, it's trying to load our assets via Laravel, but that's static content. Let's sort this out

## Static Content

Let's update our `cdk_stack.py` file to look like this:

```python
from aws_cdk.aws_apigatewayv2_integrations_alpha import HttpLambdaIntegration
from cdk_monitoring_constructs import MonitoringFacade
from aws_cdk.aws_apigatewayv2_alpha import HttpApi
from aws_cdk import (
    CfnOutput,
    Duration,
    Stack,
    Fn,
    aws_lambda as _lambda,
    aws_s3 as s3,
    aws_s3_deployment as deployment,
    aws_cloudfront as cloudfront,
    aws_cloudfront_origins as origins
)
from constructs import Construct


class CdkStack(Stack):

    def __init__(self, scope: Construct, construct_id: str, **kwargs) -> None:
        super().__init__(scope, construct_id, **kwargs)

        # This is where all our static assets live. 
        bucket = s3.Bucket(self, "StorageBucket",
            block_public_access=s3.BlockPublicAccess.BLOCK_ALL
        )

        laravel_web = _lambda.DockerImageFunction(self, "LaravelWeb",
            code=_lambda.DockerImageCode.from_image_asset("../"),
            memory_size=1024,
            timeout=Duration.seconds(20),
        )

        # Later on, we can have Laravel us the S3 Filesystem, so this lets
        # Laravel read and write objects in the bucket
        bucket.grant_read_write(laravel_web)

        web_integration = HttpLambdaIntegration("LaravelWebIntegration", laravel_web)

        endpoint = HttpApi(self, "ApiGateway",
            default_integration=web_integration
        )

        # This allows Cloudfront access to our assets bucket
        oai = cloudfront.OriginAccessIdentity(self, "CloudfrontOAI")

        endpoint_uri = Fn.select(2, Fn.split("/", endpoint.url))

        # We need to stop the HOST header getting through to API Gateway
        # This also passes all cookies and query strings through too
        laravel_cache_policy = cloudfront.CachePolicy(self, "LaravelCachePolicy",
            cookie_behavior=cloudfront.CacheCookieBehavior.all(),
            header_behavior=cloudfront.CacheHeaderBehavior.allow_list(
                'Accept',
                'Accept-Language',
                'Origin',
                'Referer'),
            query_string_behavior=cloudfront.CacheQueryStringBehavior.all()
        )

        # Ths primary behavior of our cloudfront distro is to route traffic to our
        # API gateway. We're allowing ALL methods.
        distro = cloudfront.Distribution(self, "CloudfrontDistro",
            enable_logging=True,
            default_behavior=cloudfront.BehaviorOptions(
                origin=origins.HttpOrigin(endpoint_uri),
                allowed_methods=cloudfront.AllowedMethods.ALLOW_ALL,
                viewer_protocol_policy=cloudfront.ViewerProtocolPolicy.REDIRECT_TO_HTTPS,
                cache_policy=laravel_cache_policy
            ),
            price_class=cloudfront.PriceClass.PRICE_CLASS_100
        )

        # This additional behavior diverts all traffic with the path starting
        # /assets to our S3 bucket
        distro.add_behavior("/assets/*",
            origin=origins.S3Origin(
                origin_access_identity=oai,
                bucket=bucket
            ),
            allowed_methods=cloudfront.AllowedMethods.ALLOW_GET_HEAD_OPTIONS,
            viewer_protocol_policy=cloudfront.ViewerProtocolPolicy.REDIRECT_TO_HTTPS,
            origin_request_policy=cloudfront.OriginRequestPolicy.CORS_S3_ORIGIN
        )

        # Let's pass some important environment variables to Laravel
        laravel_web.add_environment("MIX_ASSET_URL", "/assets")
        laravel_web.add_environment("ASSET_URL", "/assets")
        laravel_web.add_environment("AWS_BUCKET", bucket.bucket_name)
        laravel_web.add_environment("FILESYSTEM_DISK", "s3")
        laravel_web.add_environment("APP_URL", f"https://{distro.domain_name}")

        # This is a tasty feature of CDK - we can get the `cdk deploy` command 
        # to lift our assets from the public folder into our S3 bucket
        # notice how I move both files from public and files from public/assets here
        # otherwise the asset files would be in /assets/assets....
        deployment.BucketDeployment(self, "StaticAssetsDeployment", 
            sources=[
                deployment.Source.asset("../public"),
                deployment.Source.asset("../public/assets")
            ],
            destination_bucket=bucket,
            destination_key_prefix="assets",
            exclude=["*.php"]
        )

        monitoring = MonitoringFacade(self, "ServerlessLaravelMonitoringFacade")
        monitoring.add_large_header("Serverless Laravel Blog")
        monitoring.monitor_api_gateway_v2_http_api(api=endpoint)
        monitoring.monitor_lambda_function(lambda_function=laravel_web)
        # Add the new resources to the dashboard
        monitoring.monitor_cloud_front_distribution(distribution=distro)
        monitoring.monitor_s3_bucket(bucket=bucket)

        CfnOutput(self, "DistributionURL", value=distro.domain_name)
```

OK - So this is getting a little tastier. We've now:

* Added a cloudfront distribution which will direct all requests with the path `/assets` to an S3 bucket, the rest of the traffic goes directly to the API Gateway HTTP API (with a nice cache policy)
* Created an S3 bucket for our assets and an OAI so Cloudfront can get objects from it
* An S3 Deployment construct puts all our `public/` content onto the S3 inside an `/assets` object
* Created environment variables in our Laravel function that adds `./assets` to the uri's.
* Changes the output from the API Gateway to the Cloudfront distro domain

Let's run `cdk deploy` again and see where this gets us...

![Feature complete](/assets/img/finished.png)

OK. That's pretty neat-o huh.

Try a couple of things:

* Create a user locally and deploy with that new YAML file (`php please make:user`)
* Visit the `/cp` control panel and have a play around there - note, you'll only be able to edit stuff on your local - for now!

## Monitoring

Now - the best bit - go and check out your CloudWatch dashboard

![Feature complete](/assets/img/statamic-dashboard.png)

It's a little busy, but it's awesome to see it's all in place for us from just a couple lines of python.

## Tests

Let's make sure each of our resources are at least in the template and resemble what we're expecting.

Change your unit test file `./cdk/tests/unit/test_cdk_stack.py` look like this:

```python
import aws_cdk as core
import aws_cdk.assertions as assertions

from cdk.cdk_stack import CdkStack

def test_core_resources_created():
    app = core.App()
    stack = CdkStack(app, "cdk")
    template = assertions.Template.from_stack(stack)

    template.has_resource_properties("AWS::Lambda::Function", {
        "MemorySize": 1024,
        "Timeout": 20,
        "PackageType": "Image"
    })

    template.has_resource_properties("AWS::ApiGatewayV2::Api", {
        "ProtocolType": "HTTP"
    })

    template.has_resource_properties("AWS::ApiGatewayV2::Route", {
        "RouteKey": "$default",
        "AuthorizationType": "NONE",
    })

    template.has_resource_properties("AWS::ApiGatewayV2::Integration", {
        "IntegrationType": "AWS_PROXY"
    })

    template.has_resource_properties("AWS::ApiGatewayV2::Stage", {
        "StageName": "$default",
        "AutoDeploy": True
    })

    template.has_resource_properties("AWS::S3::Bucket", {
        "PublicAccessBlockConfiguration": {
            "BlockPublicAcls": True,
            "BlockPublicPolicy": True,
            "IgnorePublicAcls": True,
            "RestrictPublicBuckets": True
        },
    })

    template.has_resource_properties("AWS::CloudFront::Distribution", {
        "PriceClass": "PriceClass_100"
    })
```

Run this in the terminal with `pytest` - you should get all greens!

> Why didn't you TDD?

I'll tell you why I didn't use a TDD flow - because my personal knowledge Cloudformation isn't that great, and to make assertions when writing tests - I assert against Cloudformation domain specific language, but I am not sure what that would be without reading the docs (which means reading the CDK docs and the CFn docs then writing tests, which is too long a process for me)

## How are you editing the content?

I am editing the content locally in Markdown then running:

```shell
php please stache:clear
php please stache:warm
php please stache:refresh
```

in my terminal, then deploying those changes when I am happy with them.

## What's next?

There are many many directions we can go with this. I think next up has to be a deployment pipeline right? Let's see if we can get Github deploying it

I hope you enjoyed this journey

> Good luck everybody, Stu