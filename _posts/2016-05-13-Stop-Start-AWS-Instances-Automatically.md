---
layout: post
title: "Stop Start AWS Instances Automatically"
date: 2016-05-13
---


In order to save money in AWS you can stop dev instances at night and weekends and start them again in the morning.

I created a wrapper script for AWS cli tools (stop_start_aws_instances.sh) that with cronjobs can help you automatically stop aws instances when you don’t use them.

The script is located here:
<https://github.com/nachum234/scripts/blob/master/stop_start_aws_instances.sh>

## Prerequisite

In order to use the script you need to install and configure aws tools.

Here is a quick how to install and configure aws tools:

* Install aws cli tools

```
sudo pip install awscli
```

* Create IAM account with privileges to stop and start ec2 instances and save his access key and secret key. The quickest way is to use AmazonEC2FullAccess policy.
* Configure aws cli tools. You need to enter the user access key and secret key

```
aws configure
```

* or if you want to configure a specific profile

```
aws --profile dev configure
```

For more information use AWS guide: <http://docs.aws.amazon.com/cli/latest/userguide>.

## How To Use stop_start_aws_instances.sh
Here are few examples on how to use the script:

* To stop all instances that contain the tag daily-stop with value of true in us-east-1 region:

```
stop_start_aws_instances.sh -p default -a stop-instances -f Name=tag:daily-stop,Values=true -r us-east-1
```

* To test on which instances the action will apply on, run the script with describe-instances action:

```
stop_start_aws_instances.sh -p default -a describe-instances -f Name=tag:daily-stop,Values=true -r us-east-1
```

* To stop all instances that contain the tag daily-stop with value of true in us-east-1 region from multiple profiles:

```
for i in dev qa test; do stop_start_aws_instances.sh -p $i -a stop-instances -f Name=tag:daily-stop,Values=true -r us-east-1; done
```

* Create cronjobs that start instances every working days at 9:00 and stop instances at every day at 19:00:

```
00 09 * * 1-5 /usr/local/bin/stop_start_aws_instances.sh -p default -a start-instances -f Name=tag:daily-start,Values=true -r us-east-1
00 19 * * * /usr/local/bin/stop_start_aws_instances.sh -p default -a stop-instances -f Name=tag:daily-stop,Values=true -r us-east-1
```

Now you just need to add the right tags to your instances and test it :-)
