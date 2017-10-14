# Launch PHP and MySQL containers in ECS

This is a method to deploy 2 environments (PROD and UAT) in ECS with MySQL and PHP containers, along with proper monitoring, logging, backup and data persistency.

The CloudFormation template and explanation is also posted on the [NETBEARS](https://github.com/NETBEARS/php-mysql-EFS-ECS) company blog. You might want to check the website out for more tutorials like this.

## Run the CloudFormation template with AWS CLI

```
git clone https://github.com/NETBEARS/php-mysql-ecs.git

aws cloudformation create-stack \
  --stack-name php-mysql-prod-and-uat \
  --template-body file://cloudformation-template.yaml \
  --parameters \
    ParameterKey=Ami,ParameterValue=ami-29f80351 \
    ParameterKey=AsgMaxSize,ParameterValue=6 \
    ParameterKey=AsgMinSize,ParameterValue=2 \
    ParameterKey=CadvisorImage,ParameterValue=google/cadvisor:latest \
    ParameterKey=EmailAlerts,ParameterValue=email_for_alerts@domain.com \
    ParameterKey=InstanceType,ParameterValue=m4.large \
    ParameterKey=KeyName,ParameterValue=YOUR_INSTANCE_KEY \
    ParameterKey=MYSQLDATABASE,ParameterValue=wordpress \
    ParameterKey=MYSQLUSER,ParameterValue=wordpress \
    ParameterKey=MYSQLPASSWORD,ParameterValue=SECRET_PASSWORD \
    ParameterKey=NodeExporterImage,ParameterValue=quay.io/prometheus/node-exporter:latest \
    ParameterKey=PcpImage,ParameterValue=mmitrofan/pcp:latest \
    ParameterKey=ProdWordpressImage,ParameterValue=wordpress:latest \
    ParameterKey=UatWordpressImage,ParameterValue=wordpress:latest \
    ParameterKey=VpcId,ParameterValue=VPC_ID \
    ParameterKey=SubnetID1,ParameterValue=SUBNET_IN_VPC_ID_1 \
    ParameterKey=SubnetID2,ParameterValue=SUBNET_IN_VPC_ID_2 \
  --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM

```

## Run the CloudFormation template in the AWS Console
* Login to the AWS console and browse to the CloudFormation section
8 Select the "cloudformation-template.yaml” file
* Before clicking "Create", make sure that you scroll down and tick the “I acknowledge that AWS CloudFormation might create IAM resources” checkbox
* ...drink coffee...
* Go to the URL in the output section for the environment that you want to access

## Resources created
* 1 ECS cluster
* 1 AutoScaling Group
* 2 ECS services for PHP (one for each environment)
* 2 ECS services for MySQL (one for each environment)
* 2 Application Load Balancers (1 for each PHP service in each container)
* 2 Elastic Load Balancers (1 for each MySQL service in each container)
* 4 Elastic File Systems (1 for each container in each environment)
* 1 S3 bucket (for database backup)
* 1 SNS topic (send monitoring alerts)
          

## Autoscaling
The autoscaling groups uses the following alarms to autoscale automatically based on load:
* CpuUtilization
* MemoryUtilization
            
Because of this, you wouldn't have to bother making sure that your hosts can sustain the load.

## Alarms
In order to be sure that you have set up the proper limits for your containers, the following alerts have been but into place:            
* NetworkInAlarm
* RAMAlarmHigh
* NetworkOutAlarm
* IOWaitAlarmHigh
* StatusAlarm
            
These CloudWatch alarms will send an email each time the limits are hit so that you will always be in control of what happens with your stack.

## Monitoring
The stack launches 3 monitoring tools on each ECS host inside the cluster:
            
* [PCP](http://vectoross.io/) - An on-host performance monitoring framework which exposes hand picked high resolution metrics to every engineer’s browser
* [NodeExporter](https://github.com/prometheus/node_exporter) - Prometheus exporter for hardware and OS metrics exposed by NIX kernels, written in Go with pluggable metric collectors
* [cAdvisor](https://github.com/google/cadvisor) - cAdvisor (Container Advisor) provides container users an understanding of the resource usage and performance characteristics of their running containers.
             
To view the monitoring data, all you need to setup is a Prometheus host and a Grafana dashboard and you're all set.

## Logging
The stack makes use of AWS [LogGroups](http://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CloudWatchLogsConcepts.html) and offers you the possibility to store all logs in CloudWatch and view them at any point in time by simply accessing any running Task and clicking on "View Logs in CloudWatch".

## Data persistency
The stack makes use of AWS [EFS](https://aws.amazon.com/efs/) system to ensure that even though any container fails it will have access to any data that was previously being handled by the previous container.

This also ensures that any PHP container (in this case WordPress) that uses a specific storage system for its user-uploaded files, they will be available on any container, regardless of how many are currently running.

Due to its high-throughput and NFS connectivity mechanisms, it has been successfully used to store data for the (single) MySQL container running for each environment.

## Backup
Although the data persistency is full-proof in case of container or host failure, we should still have a backup for the MySQL data, just to be sure.

Because of this, a cronjob has been set up to run every 3 days on the ECS hosts that connect to each MySQL database and dump the data in an S3 bucket that is created inside the template.
            
## Final notes
Need help implementing this?

Feel free to contact us using [this form](https://netbears.com/#contact-form).
