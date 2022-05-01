---
title: "Distributed Serverless Workflow"
description: "My approach building a serverless application that notifies me of gasoline prices"
date: 2022-05-01T16:08:18-04:00
draft: false
cover:
  image: "https://dev-to-uploads.s3.amazonaws.com/uploads/articles/3x8h22ap710rnitzu1l1.png"
  alt: "serverless-application"
  caption: "serverless-application"
  relative: true
tags: ["aws", "serverless", "python", "terraform"]
---

AWS offers a variety of products to create any application you can think of. With DynamoDB, you can trigger a Lambda function to perform additional work each time a DynamoDB table is updated.

In this post, I’ll show how these two technologies can work together. Without further ado, let’s get started.

## What I'm building

[NYC Open Data](https://opendata.cityofnewyork.us/) has a public [API](https://data.ny.gov/resource/wuxr-ni2i.json) that details gasoline price information of all the New York Regions. A new gasoline price gets added every week.

I had the idea of building a system that can notify me of the gasoline prices with the condition when the price goes down. I’m going to show you how I did this. You might think the idea sounds a little vague or easy to do but the most important thing about the project is how I did it which is what I want to emphasize here in this blog.

![Architecture](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/3x8h22ap710rnitzu1l1.png)

## Setting up the project

### Getting your repository set up

I enjoy doing side projects with the latest trends in the market. In the project, I use [terraform](https://terraform.io/). For those of you that don’t know terraform is infrastructure as code that lets you build, change, and version cloud and on-premise resources safely and efficiently. In short words, instead of using the AWS UI, we use code to create AWS services.

Another cool technology that I’m using is Serverless. Serverless is a cloud-native development model that allows developers to build and run applications without having to manage servers. It’s really cool because we just need to provide the code and AWS will do all the heavy tasks in the background.

### Building the infrastructure

Something I have learned over time using terraform is that I like to structure the different resources I want to build using modules. For example, if I’m planning to use an EC2 I normally create a compute module and allocate all the logics that falls in EC2. Here’s what I did:

```jsx
module "dynamodb" {
  source = "./dynamodb"
}

module "sns" {
  source = "./sns"
  email  = var.email
}

module "lambda" {
  source                          = "./lambda"
  gasoline_prices_table_arn       = module.dynamodb.gasoline_prices_table_arn
  gasoline_price_table_stream_arn = module.dynamodb.dynamodb_stream_arn
  sns_arn                         = module.sns.sns_arn
}
```

Once we have the modules in place I start building each module separately and export any variable that a given module needs.

In AWS every resource that we want to create is private. If we want to execute a resource either programmatically or get access to it, we need to explicitly tell AWS to do it.

I created an IAM role in the `lambda` module to detail all the permissions that the Lambda needs.

```jsx
resource "aws_iam_policy" "iam_role_policy_for_lambda" {

  name        = "aws_iam_policy_for_terraform_aws_lambda_role"
  description = "AWS IAM Policy for managing aws lambda role"
  policy = jsonencode({
    Statement = [{
      Action   = ["s3:GetObject", "s3:PutObject"],
      Effect   = "Allow",
      Resource = "${aws_s3_bucket.price_fetcher_deployment.arn}"
      },
      {
        Action   = ["dynamodb:Scan", "dynamodb:Query", "dynamodb:PutItem"],
        Effect   = "Allow",
        Resource = "${var.gasoline_prices_table_arn}"
      },
      {
        Action   = ["logs:CreateLogGroup", "logs:CreateLogStream", "logs:PutLogEvents", "logs:DescribeLogStreams", "logs:createExportTask"],
        Effect   = "Allow",
        Resource = "*"
      },
      {
        Action   = ["dynamodb:DescribeStream", "dynamodb:GetRecords", "dynamodb:GetShardIterator", "dynamodb:ListStreams"],
        Effect   = "Allow",
        Resource = "${var.gasoline_price_table_stream_arn}"
      },
      {
        Action = ["sns:Publish"],
        Effect   = "Allow",
        Resource = "${var.sns_arn}"
      }
    ]
    Version = "2012-10-17"
  })
}
```

As you can see above, we have explicitly told AWS which permissions my AWS needs to execute the Lambda. This is something that you have to think about as you continue adding functionalities to your Lambda.

If you wondered before why I was passing some variables to the `lambda` from the `dynamodb` module. In order to grant permission to our Lambda to execute queries in Dynamo DB. We need to get the reference of the table and explicitly add it to the policy.

### Lambda and More

It only takes a few minutes to create a lambda using the serverless framework. Make sure you have the serverless package installed on your machine. Depending on what type of serverless template you want to create (e.g javascript, python) you specify it in the serverless CLI. For example

```jsx
sls create --template aws-python3 --path myService
```

The serverless.yml is the heart of a serverless application. This file describes the entire application infrastructure, all the way from the programming language to resource access.

A few examples of properties we can use:

- The functions we want to run
- Environment variables
- Depending on the cloud provider we’re using, we can specify the role and the region our code will get run on

For more information about this, check out this [documentation](https://www.serverless.com/framework/docs/providers/aws/guide/serverless.yml).

Here’s what my YAML looks like:

```yaml
service: aws-price-fetcher

frameworkVersion: "3"

provider:
  name: aws
  runtime: python3.8
  region: us-east-1
  deploymentBucket:
    name: price-fetcher-serverlessdeploymentbucket
  iam:
    role: arn:aws:iam::008735640664:role/Price-Notifier-Role

functions:
  price_fetcher:
    handler: handler.price_fetcher
    description: Fetches gasoline price from public API
    events:
      - schedule: cron(* * 0 ? * WED *)
  price_publisher:
    handler: handler.price_publisher
    description: Publish gasoline drops

plugins:
  - serverless-python-requirements
```

I won’t go to each one of the properties because some of them are self-explanatory.

`service` is the name of our lambda in AWS.

`provider/region`: Lambas are region-specific. What this means is that each Lambda function lives in a specific AWS region.

`deploymentBucket`: when we create a lambda our lambda has to live somewhere in AWS. This is why in terraform we are creating a bucket for our lambda to live in and by this means we need to specify it in the YAML file.

`role` The role created in terraform along with its permissions to execute resources in AWS.

`functions`: When we create a lambda we can have multiple functions living in one lambda. To make this project easy, I chose to have 1 lambda with 2 functions one for fetching the gasoline price from the public API and another one to publish the gasoline drops.

`plugins` are just the different libraries we are using in serverless. We're using `serverless-python-requirements` because we need some libraries to execute our code, for example, `requests`. If you’re curious to see more libraries, check out this [documentation](https://www.serverless.com/plugins) that details all the plugins serverless has.

In the lambda, I created a function called `price_fetcher` whose purpose is to query the DynamoDB table, if there are any records it gets the most recent gasoline price, calculates against the last record we have in the database if the price has dropped it insert the record to the table otherwise, it doesn’t do the insertion.

```python
 def price_fetcher(event: Dict[str, Any], _: Any) -> None:
    """
    Fetches gasoline price from public API
    Calculate the most recent gasoline price with the last record saved in the DynamoDB table
    if the price has dropped, it inserts that record into the table otherwise, it skips it.
    """
    items = query(GASOLINE_PRICE_TABLE_NAME)
    LOG.info(f"Got items: {items}")

    gasoline_api_response = request()

    gasoline_api_json = gasoline_api_response.json()

    most_recent_gasoline_price: Dict = gasoline_api_json[0]
    LOG.info(f"Got most recent gasoline price: {most_recent_gasoline_price}")

    published_at = most_recent_gasoline_price['date']
    gasoline_price = most_recent_gasoline_price['new_york_state_average_gal']

    if items:
        previous_gasoline_price = float(items[0]['newYorkStateAverageGal'])
        LOG.info(f'Got previous previous_gasoline_price {previous_gasoline_price}')

        recent_gasoline_price = float(gasoline_price)

        price_has_dropped = recent_gasoline_price < previous_gasoline_price

        if not price_has_dropped:
            LOG.info("Skipping insertion because price hasn't dropped")
            return

    response = insert(GASOLINE_PRICE_TABLE_NAME, published_at=published_at, gasoline_price=gasoline_price)
    LOG.info(f"Got response: {response}")

    return None
```

DynamoDB provides a Change Data Capture (CDC) mechanism for each table.

That means, if someone creates a new entry or modifies an item in a table, DynamoDB immediately emits an event containing the information about the change. You can build applications that consume these events and take action based on the contents.

DynamoDB keeps track of every modification to data items in a table and streams them as real-time events.

Here’s how I did it using terraform:

```python
resource "aws_lambda_event_source_mapping" "gasoline_prices_stream" {
  event_source_arn  = aws_dynamodb_table.gasoline_prices_table.stream_arn
  function_name     = var.price_publisher_arn
  starting_position = "LATEST"
}
```

We need to provide a unique ARN that points to our table and indicates the function where we want to send these events.

Next, see how I’m getting the events from the Lambda:

```python
def price_publisher(event: Dict[str, Any], _: Any) -> None:
    """
    Get DynamoDB events and send information to the user.
    """
    LOG.info(f'Got event {event}')
    record = event['Records'][0]
    most_recent_gasoline_price = record['dynamodb']['NewImage']['newYorkStateAverageGal']['S']
    message = f'🛡🛡🛡 Gasoline price {most_recent_gasoline_price} have fell 🛡🛡🛡'
    subject = 'A gasoline price dropped has been detected'
    send_sns(message, subject)

def send_sns(message, subject):
    client = boto3.client("sns")
    client.publish(
        TopicArn=SNS_ARN, Message=message, Subject=subject)

```

Lastly, we use AWS Simple Notification Service (or AWS SNS) a cloud-based web service that delivers messages. In SNS we have a publisher and a subscriber.

The publisher sends out a message to the subscriber immediately instead of storing the message.

The subscriber s an entity that receives the messages to the topics that they have subscribed to. In my case, I’m using my email to receive the notifications.

Here’s the final result when I receive the mail from SNS

![SNS Notification](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/p65do2pltfmh75s7cf9e.png)

## Summary and where to next?

This was a really interesting project that taught me two important things I won’t forget. It taught me how permissions really work on AWS. They can be confusing sometimes. Do I need a role for this project? What policy do I need to run this action? What users, groups, and roles really are, and their differences? These and other questions were the ones I was able to answer during this project. After learning how to spin up a Lambda using the Serverless framework and reading records with DynamoDB streams, it’s my turn to keep digging more into these great technologies to continue creating more projects like this.

If you want to send me feedback, or just want to connect with me, let me know via [Twitter](https://twitter.com/JustCallMeJochi) or [LinkedIn](https://www.linkedin.com/in/jose-hidalgo-rosa/). Also, if you are curious to see the entire project, check it out on my [Github](https://github.com/JoseAngel1196/gas-price-notifier). Till the next one, stay safe and keep learning.
