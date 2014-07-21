CloudFormation template for a [Jenkins](https://jenkins-ci.org/) server with automatic backup and recovery.

Prerequisites:
* Route 53 hosted zone for the desired DNS address (e.g., `mycompany.com` for `jenkins.mycompany.com`)

## Overview

This template bootstraps a Jenkins server.

The Jenkins server is launched in an auto scaling group using public AMIs running Ubuntu 14.04 LTS and pre-reloaded with Docker and Runit.  If you wish to use your own image, simply modify `RegionMap` in the template file.

The server registers with an Elastic Load Balancer, a CNAME for which is created in Route 53. Use this CNAME as your bookmarked address.

The auto scaling group is pinned at a capacity of one. If your Jenkins server is terminated, a new one will come back and load the most recent backup.

The Jenkins daemon is run via a Docker image specified as a Parameter. You can use the default or provide your own.

Jenkins will be backed up daily to S3. At server boot, Jenkins will be restored from the latest S3 backup (if one exists).

Note that this template must be used with Amazon VPC. New AWS accounts automatically use VPC, but if you have an old account and are still using EC2-Classic, you'll need to modify this template or make the switch.

## Usage

### 1. Clone the repository
```bash
git clone git@github.com:thefactory/cloudformation-jenkins.git
```

### 2. Create an Admin security group
This is a VPC security group containing access rules for cluster administration, and should be locked down to your IP range, a bastion host, or similar. This security group will be associated with the Jenkins server.

Inbound rules are at your discretion, but you may want to include access to:
* `22 [tcp]` - SSH port
* `80 [tcp]` - ELB HTTP port

### 5. Launch the stack
Launch the stack via the AWS console, a script, or [aws-cli](https://github.com/aws/aws-cli).

See `jenkins.json` for the full list of parameters, descriptions, and default values.

Example using `aws-cli`:
```bash
aws cloudformation create-stack \
    --template-body file://jenkins.json \
    --stack-name <stack> \
    --capabilities CAPABILITY_IAM \
    --parameters \
        ParameterKey=KeyName,ParameterValue=<key> \
        ParameterKey=S3Bucket,ParameterValue=<bucket> \
        ParameterKey=VpcId,ParameterValue=<vpc_id> \
        ParameterKey=Subnets,ParameterValue='<subnet_id_1>\,<subnet_id_2>' \
        ParameterKey=AdminSecurityGroup,ParameterValue=<sg_id> \
        ParameterKey=DnsZone,ParameterValue=<zone>
```

### 4. Watch the stack come up
Once the stack has been provisioned, visit `http://<route53_address>/`. You will need to do this from a location granted access by the specified `AdminSecurityGroup`.

_Note the Docker image for Jenkins may take several minutes to retrieve. This can be improved with the use of a private Docker registry._

You should see the Jenkins UI. You can now use it as you would any other Jenkins server. Daily backups will automatically be generated and uploaded to the specified S3 location.