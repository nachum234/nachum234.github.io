---
layout: default
title: "Automatic Backup of AWS instances"
date: 2016-05-14
---

There is no builtin option in AWS to backup instances automatically, so I created a ruby script that can run from crontab and create automatic AMI images from ec2 instances.

aws_ami_autobackup.rb works with ec2 tags, the script get tag and value and create AMI from all instances that has this tag and value.

Here is how to install and use the script

## Prerequisite
* Install ruby (I use ruby 2.2) with aws-sdk-resources gem.
* Create IAM account with privileges to create and remove ec2 snapshots and AMI and save his access key and secret key. The quickest way is to use AmazonEC2FullAccess policy.
* Create credentials file for the user that will run the tool in ~/.aws/credentials

```
vi ~/.aws/credentials
```

```
[default]
aws_access_key_id = XXXXXXXXXXXXXXXXXXXX
aws_secret_access_key = YYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYY
```

# How To Use aws_ami_autobackup.rb

Here are few examples on how to use the tool:

* To take an ami on all instances that contain the tag daily-backup with value of true in us-east-1 region and keep them for 7 days:

```
/usr/local/bin/aws_ami_autobackup.rb -t daily_backup -v true -r us-east-1 -x 7
```

* To take an ami on all instances that contain the tag daily-backup with value of true in us-east-1 region from multiple profiles (aws accounts):

```
for i in dev qa test; do /usr/local/bin/aws_ami_autobackup.rb -t daily_backup -v true -r us-east-1 -x 7 -p ${i}; done
```

* Create cronjobs that take an ami every day at 00:00 and keep them for 30 days:

```
00 00 * * * /usr/local/bin/aws_ami_autobackup.rb -t daily_backup -v true -r us-east-1 -x 7
```

Now you just need to add the right tags to your instances and test it :)
