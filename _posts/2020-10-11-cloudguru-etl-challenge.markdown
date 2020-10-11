---
layout: post
title:  "Event-Driven Python ETL: ACloudGuru September 2020 Challenge"
date:   2020-10-11 12:00:00 -0400
categories: posts
---


Hi there! This is my first blog post on this site, so I wanted to make it a fun one. I've just completed ACloudGuru's first monthly challenge, to build an event-driven application using Python on AWS, and I am going to share some details of my approach to architecting and building this pipeline.

The challenge is well-designed and clearly written, but still gives a lot of room for creativity. I was especially excited by this challenge because completing it involves applying several concepts I have been learning and using every day. Specifically: serverless, event-driven cloud processes, and infrastructure-as-code (IaC).

There is a reason that serverless, IaC, and other similar concepts have become such hot buzzwords over the past several years - they are really useful! Running the pipeline I built costs almost nothing, it's highly scalable, there are no servers to maintain, and the whole thing deploys with the push of a button. These benefits really matter when it comes to building resilient applications and a high-performing team. Although this challenge is simpler than most real-world data pipelines, the tools and concepts are extremely relevant and practical for a broad range of applications.

#### The Challenge

In brief, this is the challenge:

> Automate an ETL processing pipeline for COVID-19 data using Python and cloud services.

This is a really broad instruction! Fortunately, there are plenty of more details provided at the [ACloudGuru blog post](https://acloudguru.com/blog/engineering/cloudguruchallenge-python-aws-etl).
The post outlines several required components:

- Scheduling
- Compute
- Data Store
- Notification
- Infrastructure Definition
- Visualization (`#TODO`)

Below, I will discuss each of these in sequence, followed by the  Python code that does the actual work.

![A generic architecture diagram from ACloudGuru](https://acg-wordpress-content-production.s3.us-west-2.amazonaws.com/app/uploads/2020/09/ETL.png)

#### A Note on Infrastructure & Orchestration

For this exercise, I used a Ruby gem called [`stax`](https://github.com/rlister/stax) to define and orchestrate everything.
`stax` is written by our head of devops, Ric Lister, and is a highly-opinionated tool informed by his many years of experience. Ultimately, everything here is used to generate Cloudformation template, but with a series of abstractions that I find to be really useful and fun. The templates themselves utilize another Ruby gem called ['cfer'](https://github.com/seanedwards/cfer). Hopefully, the Ruby code is presented in a lightweight-enough way that the underlying concepts are accessible.

Each component presents an opportunity to make an interesting design decision. There are certain choices I would have made differently if this was a real, production-grade application, but each of the services and tools I used has its place in your real-world toolkit.

#### Schedule & Compute

There are several ways to schedule things on AWS, but the simplest and best in my opinion is to
use a Cloudwatch (now EventBridge) rule. My rule looks like this:

```ruby
# event_rule.rb
resource :DailyEtlRule, 'AWS::Events::Rule' do
  schedule_expression 'cron(0 0 * * ? *)'
  state :ENABLED
  targets [
    {
      Id: Fn::ref('AWS::StackName'),
      Arn: Fn::get_att(:EtlLambdaFunction, :Arn),
    }
  ]
end
```

For lightweight, event-driven compute, Lambda is the go-to choice. The Python that I wrote
is relatively straight-forward, is contained in a single module, and has no dependencies
outside of the standard library. Additionally, Lambda's current limits - runtime of 15 minutes,
and memory allocation of 3 GB - should be more than enough.

```ruby
# lambda.rb
code = File.read(File.join(File.dirname(__FILE__), 'handler.py'))

resource :EtlLambdaFunction, 'AWS::Lambda::Function', DependsOn: :IamRoleLambdaFunction do
  handler 'index.main'
  role Fn::get_att(:IamRoleLambdaFunction, :Arn)
  code(
    ZipFile: code
  )
  runtime 'python3.7'
  self["Properties"]["Timeout"] = 60
  environment do
    variables(
      DYNAMO_DB_TABLE_NAME: Fn.import_value(Fn.sub('${dynamo}-DynamoDbTableName')),
      SNS_TOPIC_ARN: Fn.ref(:SnsTopic)
    )
  end
end
```

Some other possible schedule + compute options were:

##### Fargate Scheduled Task
Fargate provides a serverless, on-demand container service. It's
a great service, and worth considering for more complex pipelines, especially if you need
to build your own container image. For this task, I didn't see any benefit to defining my own image.

##### Glue
Glue is AWS' managed ETL service. You can choose to run Python Shell jobs (i.e., scripts) or PySpark jobs. I have some experience with Glue Spark jobs, which are decent if the task at hand requires some heavy duty compute power, but the serverless Spark clusters can take 10+ minutes to provision, and can get costly very fast. I haven't used Shell Jobs, but it looks like a good option for small-to-medium sized tasks - the runtime comes with useful libraries like NumPy and Pandas.


#### Data Store

AWS offers a wide variety of ways to store data. I chose to use DynamoDB, AWS' serverless NoSQL database.  I made this choice not because it was necessarily the best tool for the job, but because it's an interesting service. DynamoDB *is* well-suited to the task at hand - it is super cheap, easy to setup, and easy to query - but only if you assume that the requirements won't change over time.

The benefits of DynamoDB are that there is no infrastructure to maintain, it scales very well, and it's super flexible.  The cost of that flexibility is that it can be difficult to query, and it helps a great deal to know your data model really well up front so that you can
choose the right keys.

My DynamoDB table looks like this:

```ruby
# table.rb
resource :DynamoDbTable, 'AWS::DynamoDB::Table' do
  table_name Fn::sub('${app}-${branch}-covid-data')
  attribute_definitions [
    { AttributeName: :date, AttributeType: :S },
  ]
  key_schema [
    { AttributeName: :date, KeyType: :HASH },
  ]
  billing_mode :PAY_PER_REQUEST
end
```

Some other possible data-store options were:

##### Timestream
Timestream just became Generally Available earlier this month, and is Amazon's answer to the increasingly popular concept of a Time Series Database. The data in question here is not a "time series" per se, but the primary key is a date, so I really _wanted_ to take the opportunity to use Timestream.

The service is very similar to DynamoDB in its functionality, but with even less configuration. There is no need to define primary or secondary keys in advance: in fact, there is no schema at all. You simply write a record with a given timestamp, measures, and dimensions. It is append-only, and you can only define a single measure per record.

An extremely interesting feature is that the latest data is always stored in-memory, according to a retention policy that you define. Beyond that retention policy - which can be up to 365 days - the data is moved to magnetic storage (again, for a configurable amount of time). This is really cool, and likely provides enormous performance benefits for Timestream's intended applications (smart devices, infrastructure monitoring, etc), but makes the service completely unusable for run-of-the-mill time series analysis that your friendly local grad student might want to do. The reason for this is that you simply _cannot_ write data directly to magnetic storage - any records older than one year can _never_ be loaded into a Timestream table.

The reason I ultimately chose not to use Timestream for this application, however, is because the available documentation is incomplete, and Cloudformation support is not yet polished. When I repeatedly got an error message reading, in its entirety, `null`, I decided to give up on Timestream until it is a little bit more mature. Too bad!


##### RDS
Amazon's Relational Database Service is the most obvious choice. RDS is great, and provides numerous flavors to choose from (MySQL, PostgreSQL, SQL Server for some reason), but it generally requires that you provision an RDS database instance, which runs up an hourly cost. Additionally, using RDS requires setting up a bunch of networking that I simply don't want to bother with for a simple project - a VPC, subnets, security group, route table, etc.

##### Aurora Serverless
Technically, this is a part of RDS, but it's quite different in that there is no need to provision an instance. Saving on the hourly cost is very nice! This seems to work in a very similar fashion to DynamoDB, but you still need to setup all of those fun bits of network stuff.

##### S3
S3 is AWS' very well known object storage service. S3 would be a great choice here, and AWS provides a lot of interesting tools to query data on S3 that is stored in a flat-file format (CSV, JSON, Parquet). If you're already using a Redshift data warehouse, it's very convenient to `COPY` data from S3 or `UNLOAD` your results back. The only downside I can think of to using S3 here is you'd have to do a bit more work to make the data available to analysts who may want to query it from their preferred database client.

##### Redshift
Redshift is AWS' data warehouse offering, a massively scaleable relational database built on top of an ancient version of the PostgreSQL engine. It's great, and I use it every day, but it's extreme overkill for this challenge.

In a real-world situation, assuming I already had an RDS or Redshift database up and running,
I likely would have chosen that as the data store. The ability to create staging tables and
perform joins would actually have been quite useful for the task at hand, especially for determining day-over-day differences while loading the data.

#### Notifications

One of the challenge requirements is to send a notification upon completion of the ETL job. In order to send notifications, the obvious choice is to use SNS - the _Simple Notification Service_. SNS uses a "pub/sub" model, meaning that you must "publish" a message to a "topic", and then you must "subscribe" to that topic using one of several available protocols. SNS is an extremely flexible service, and can be used to build wildly complex workflows. Fortunately, the present requirement is quite simple, and I was able to configure it with very little hassle. My SNS configuration looks like this:

```ruby
# sns.rb
resource :SnsTopic, 'AWS::SNS::Topic'

resource :SnsSubscription, 'AWS::SNS::Subscription' do
  Protocol :email
  Endpoint Fn.ref(:email)
  TopicArn Fn.ref(:SnsTopic)
end
```

Not bad! The only other service I can think of that would be worthy of consideration would be SES, which is Amazon's _Simple Email Service_. I've used it before, and like it, but it's a little bit fiddly to configure using Cloudformation, and there is no obvious benefit. It is much nicer to be able to make a direct call to `sns.publish()` than it is to deal with using secrets, setting up an SMTP connection, and building an email object from scratch.

#### Python: The Actual Code

Now that I've finished describing the various AWS services, it's time to talk about what the Python code I wrote actually does. You can find the code on [Github](https://github.com/speedyturkey/cloudguruchallenge-python-aws-etl/blob/main/ops/cf/etl/handler.py).

This code is structured as a single-module Lambda function, invoked by a scheduled Cloudwatch event. The instructions advised:

> Abstract your data manipulation work into a Python module. This module should only perform transformations. It should not care where the CSV files are stored and it should not know anything about the database in the next step.


I disagree with part of this instruction, because it feels like a premature abstraction for the sake of abstraction - I'd rather have no abstraction than the wrong one. However, I did choose to set the data source URLs as global variables, and to set my resource identifiers as environment variables within the Lambda, so that this code need not know anything about the database.

Here is my `main` function, which is intentionally kept very simple:

```python3
# handler.py
def main(event, context) -> None:
    """
    Perform extract, transform, and load of NYT Covid Data.
    If no errors, send success notification.
    Otherwise, catch error and send failure notification.
    """
    try:
        new, updated = load(transform(extract()))
        notify_success(new, updated)
    except Exception as exc:
        notify_failure(str(exc))
```

Because I don't have a lot of prior experience working with DynamoDB, the `load` function was the most interesting to me. I chose to do a full table scan, loading all of the existing data into memory for comparison against the "new" data. For both performance and cost reasons, this is only a good idea to do with small datasets - and with 262 records at time of writing, this certainly qualifies. Additionally, if you write a record to a DynamoDB table where the primary key already exists (and secondary, if one is defined), it simply overwrites the existing item with new whatever new attributes were provided. If I had chosen a relational database, I would have instead used a temporary or staging table to perform this step, joining it to the existing data. A slightly abridged `load` function is shown below:

```python3
# handler.py
def load(new_data: List[Dict]) -> Tuple:
    """
    Load items to DynamoDB, and return count of new and changed records.
    """
    dynamodb = boto3.resource("dynamodb")
    table = dynamodb.Table(DYNAMO_DB_TABLE_NAME)
    existing_data = {row["date"]: row for row in scan_existing_data(table)}
    new_count = 0
    updated_count = 0
    with table.batch_writer() as batch:
        for row in new_data:
            row["date"] = str(row["date"])
            exists = existing_data.get(row["date"])
            if not exists:
                new_count += 1
            if exists and row != exists:
                updated_count += 1
            batch.put_item(
                Item={
                    "date": row["date"],
                    "cases": row["cases"],
                    "deaths": row["deaths"],
                    "recovered": row["recovered"],
                }
            )
    return new_count, updated_count
```

#### Remaining Tasks
I have not yet written tests, and I have not built any kind of dashboard. I am planning to update this post once I have finished these important pieces!

#### Closing Thoughts
I really enjoyed working on this exercise. There were some interesting choices to make, but without the stress of designing a production-grade application and having to really focus on the design trade-offs, cost implications, and all of the other important details. Kudos to Forrest Brazeal and ACloudGuru for coming up with such a fun concept - I'm excited to see what next month has in store!

I don't yet have any grand designs for what to write about on this site, so if I've mentioned anything interesting that you might like more detail about, please drop me a line!

