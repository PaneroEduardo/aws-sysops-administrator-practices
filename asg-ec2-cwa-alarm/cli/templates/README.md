# Practice 1: Autoscaling group with a Cloudwatch alarm.

## Creation a new launch template
### Get the image id of the Amazon Linux OS (Free tier)
```sh
aws ec2 describe-images --owners amazon --filters 'Name=name,Values=al2023-ami-2023.2.20231113.0-kernel-6.1-x86_64' --query 'Images[0].ImageId' --output text

> ami-*****************
```

### Create a new launch template
```yaml
LaunchTemplateName: asg-ec2-cwa-template
LaunchTemplateData:
  ImageId: 'ami-*****************'
  InstanceType: t2.micro
  TagSpecifications:
    - ResourceType: instance
      Tags:
        - Key: 'name'
          Value: 'asg-ec2-cwa-instance'
        - Key: 'type'
          Value: 'practice'
    - ResourceType: volume
      Tags:
        - Key: 'name'
          Value: 'asg-ec2-cwa-volume'
        - Key: 'type'
          Value: 'practice'
TagSpecifications:
  - ResourceType: launch-template
    Tags:
      - Key: 'name'
        Value: 'asg-ec2-cwa-template'
      - Key: 'type'
        Value: 'practice'
```
```sh
aws ec2 create-launch-template --cli-input-yaml "file://templates/launch-template.yml"
```

## Create a new autoscaling group
### Get the VpcId of the default VPC
```sh
aws ec2 describe-vpcs --filters 'Name=is-default,Values=true' --query 'Vpcs[*].[VpcId]' --output text

> vpc-*****************
```

### Get the subnets of the current default VPC
```sh
aws ec2 describe-subnets --filters 'Name=vpc-id,Values=vpc-0c789c24647d3eba6' --query 'Subnets[*].[SubnetId]' --output text

> subnet-*****************
> subnet-*****************
> subnet-*****************
```

### Implement the autoscaling group with the Vpc Subnets
```yaml
AutoScalingGroupName: 'asg-ec2-cwa-asgroup'
LaunchTemplate:
  LaunchTemplateName: 'asg-ec2-cwa-template' 
  Version: '$Latest'
MinSize: 1
MaxSize: 2
DesiredCapacity: 2
VPCZoneIdentifier: 'subnet-*****************,subnet-*****************,subnet-*****************'
Tags:
  - Key: 'name'
    Value: 'asg-ec2-cwa-asgroup'
    PropagateAtLaunch: false
  - Key: 'type'
    Value: 'practice'
    PropagateAtLaunch: false
```

```sh
aws autoscaling create-auto-scaling-group --cli-input-yaml "file://templates/launch-autoscaling-group.yml"
```

## Create a new autoscaling policy
```yaml
AutoScalingGroupName: 'asg-ec2-cwa-asgroup'
PolicyName: 'asg-ec2-cwa-scaling-policy'
PolicyType: 'SimpleScaling'
AdjustmentType: 'ChangeInCapacity'
ScalingAdjustment: 1
Cooldown: 300
Enabled: true
```

```sh
aws autoscaling put-scaling-policy --cli-input-yaml file://templates/launch-scaling-policy.yml --query 'PolicyARN' --output text

> arn:aws:autoscaling:eu-west-1:************:scalingPolicy:********-****-****-****-************:autoScalingGroupName/asg-ec2-cwa-asgroup:policyName/asg-ec2-cwa-scaling-policy
```


## Create a new Cloudwatch Alarm

### Create a new SNS Notification

```sh
aws sns create-topic --name asg-ec2-cwa-sns --tags 'Key=name,Value=asg-ec2-cwa-sns' 'Key=type,Value=practice' --query 'TopicArn' --output text

> arn:aws:sns:eu-west-1:************:asg-ec2-cwa-sns
```

```sh
aws sns subscribe --topic-arn arn:aws:sns:eu-west-1:************:asg-ec2-cwa-sns --protocol email --notification-endpoint panero.eduardo@gmail.com

> SubscriptionArn: pending confirmation
```

```sh
aws sns publish --topic-arn arn:aws:sns:eu-west-1:************:asg-ec2-cwa-sns --subject "WARNING: CPUUtilization 70%" --message "XXXXXX"

> MessageId: ********-****-****-****-************
```

### Create a new Cloudwatch Alarm

```yaml
AlarmName: 'asg-ec2-cwa-alarm'
AlarmDescription: ''
ActionsEnabled: true
AlarmActions:
- 'arn:aws:sns:eu-west-1:************:asg-ec2-cwa-sns'
- 'arn:aws:autoscaling:eu-west-1:************:scalingPolicy:********-****-****-****-************:autoScalingGroupName/asg-ec2-cwa-asgroup:policyName/asg-ec2-cwa-scaling-policy'
MetricName: 'CPUUtilization'
Namespace: 'AWS/EC2'
Statistic: Average
Dimensions:
- Name: 'AutoScalingGroupName'
  Value: 'asg-ec2-cwa-asgroup'
Unit: Seconds
Period: 300
EvaluationPeriods: 3
Threshold: 70.0
ComparisonOperator: GreaterThanOrEqualToThreshold
Tags:
- Key: 'name'
  Value: 'asg-ec2-cwa-alarm'
- Key: 'type'
  Value: 'practice'
```

```sh
aws cloudwatch put-metric-alarm --cli-input-yaml "file://templates/launch-cloudwatch-alarm.yml"
```

## Test the autoscaling group
With the following command, we can test when the alarm is active. If the alarm is active, the autoscaling group should created a new instance.
```sh
aws cloudwatch set-alarm-state --alarm-name "asg-ec2-cwa-alarm" --state-value ALARM --state-reason "testing purpose"
```