# Real World AWS CLI Examples


## ECS

### Get the latest task definition for a task family

```
$ aws ecs list-task-definitions --family-prefix task-family-prefix --query 'taskDefinitionArns[-1:]' --output text
arn:aws:ecs:eu-central-1:123456789012:task-definition/task-family-prefix:47
```

### Get the image used by a container in a task definition

For a given task definiton, I want to know the image used the container named `my-container`.

```
$ aws ecs describe-task-definition \
    --task-definition arn:aws:ecs:eu-central-1:123456789012:task-definition/task-family-prefix:47 \
    --query "taskDefinition.containerDefinitions[?name=='my-container'].[image][]" --output text
234567890121.dkr.ecr.eu-central-1.amazonaws.com/my-image:my-tag
```


## IAM

### Assume a role on another account and set environment

```bash
eval $(
        aws sts assume-role \
          --role-arn arn:aws:iam::123456789012:role/role_to_assume \
          --role-session-name lskjfdsljfsldfkj | \
        jq  -r '"export AWS_ACCESS_KEY_ID=\"" + .Credentials.AccessKeyId + "\"",
                "export AWS_SECRET_ACCESS_KEY=\"" + .Credentials.SecretAccessKey + "\"",
                "export AWS_SESSION_TOKEN=\"" + .Credentials.SessionToken + "\""'
)
```

After the above command succesfully executes, the environment variables
`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` and `AWS_SESSION_TOKEN`
are set and the AWS CLI can be used for actions covered by the permissions
granted by the role on the remote account.

```bash
$ set | grep AWS_
AWS_ACCESS_KEY_ID=ASIAZZZZZZZZZZZZ
AWS_SECRET_ACCESS_KEY=sflkdjfsldkfjsdlkfsjdflksjdflsdj
AWS_SESSION_TOKEN=dfskjldjs....slkdjfsldjfs
```

## Route 53

### Dump all records for a _Zone_ in a readable format

```bash
aws route53 list-resource-record-sets \
  --hosted-zone-id Z2L52Z1C7UYKOW \
  --query 'ResourceRecordSets[*].[Name, Type, TTL, ResourceRecords[0].Value]' \
  --output text
```

The result will look like this:

```bash
woodpecker.acme.com.        CNAME   300     woody.acme.com
bugs.acme.com.       CNAME   300     bunny.acme.com
daffy.acme.com.     A       300     12.13.14.15
duck.acme.com.       A       300     11.22.33.44
sylvester.acme.com.       A       300     55.66.77.88
```

## AWS CloudWatch

### CloudWatch events

#### Disable

```
for rule in $(aws events list-rules --query "Rules[*]|[?starts_with(Name,'MyRules')].Name" --output text)
do
  aws events disable-rule ${rule}
done
```

#### Enable

```
for rule in $(aws events list-rules --query "Rules[*]|[?starts_with(Name,'MyRules')].Name" --output text)
do
  aws events enable-rule ${rule}
done
```

### Disable all CloudWatch alarms that start with a given string

#### Build the list of alarms

```
$ aws cloudwatch describe-alarms \
  --query "MetricAlarms[*]|[?starts_with(AlarmName, 'MyAlarm')].AlarmName"
[
    "MyAlarm-001-G1OKLD8WV8AX",
    "MyAlarm-002-AE2QOMTSW0CL",
    "MyAlarm-003-1VXVS2R470KSH",
    "MyAlarm-004-1LSTE90T9SV8P",
    "MyAlarm-005-1TG83Q2W9WWFH",
    "MyAlarm-006-1RINGKR9BW8RX",
    "MyAlarm-007-14Y6GIY66F88C",
    "MyAlarm-008-M8QBJWRE19HT"
]
```

Next, add the `--output text` option to have all _AlarmNames_ as a string on a singel line of output:

```
$ aws cloudwatch describe-alarms \
  --output text \
  --query "MetricAlarms[*]|[?starts_with(AlarmName, 'MyAlarm')].AlarmName"
MyAlarm-001-G1OKLD8WV8AX MyAlarm-002-AE2QOMTSW0CL MyAlarm-003-1VXVS2R470KSH MyAlarm-004-1LSTE90T9SV8P MyAlarm-005-1TG83Q2W9WWFH MyAlarm-006-1RINGKR9BW8RX MyAlarm-007-14Y6GIY66F88C MyAlarm-008-M8QBJWRE19HT
```

#### Disable the alarms

Bringing the above into the command `aws cloudwatch disable-alarm-actions` yields in:

```
$ MY_ALARMS=$(aws cloudwatch describe-alarms \
  --output text \
  --query "MetricAlarms[*]|[?starts_with(AlarmName, 'MyAlarm')].AlarmName")
$ aws cloudwatch disable-alarm-actions --alarm-names ${MY_ALARMS}
```

## AWS Certificate Manager

### List the DNS verification records for a given certificate

```bash
aws acm describe-certificate \
  --certificate-arn arn:aws:acm:eu-central-1:123456789012:certificate/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx \
  --query '[Certificate.DomainValidationOptions][0][*].ResourceRecord.[Name, Value]' \
  --output text
_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx.example.com. _yyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyy.acm-validations.aws.
_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx.example.be. _yyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyy.acm-validations.aws.
_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx.example.eu. _yyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyy.acm-validations.aws.

```
## AWS CloudFormation

### List CloudFormation Stacks

#### List all stacks:

```bash
$ aws cloudformation list-stacks
{
    "StackSummaries": [
        {
            "StackId": "arn:aws:cloudformation:eu-central-1:12345678912:stack/myStack/fdd91790-2922-11e8-a8de-500c44f62262",
            "StackName": "myStack",
            "TemplateDescription": "My Stack",
            "CreationTime": "2018-03-16T14:04:49.978Z",
            "StackStatus": "CREATE_COMPLETE"
        },
        ...
    ]
}
```

#### List stacks with status `CREATE_COMPLETE`:

```bash
$ aws cloudformation list-stacks \
    --query "StackSummaries[?StackStatus=='CREATE_COMPLETE'].[StackName, StackStatus]" \
    --output text
DevECS	CREATE_COMPLETE
DevLBALBInt	CREATE_COMPLETE
SecuritySubaccount	CREATE_COMPLETE
BastionForDev	CREATE_COMPLETE
DevRoute53	CREATE_COMPLETE
```

#### List stacks with status `CREATE_COMPLETE` and name starting with a certain string:

```bash
$ aws cloudformation list-stacks \
    --query "StackSummaries[?StackStatus=='CREATE_COMPLETE']|[?(starts_with(StackName, 'TstSand'))].[StackName, StackStatus]" \
    --output text
TstSandCloudFront       CREATE_COMPLETE
TstSandS3       CREATE_COMPLETE
TstSandLBALBExt CREATE_COMPLETE
TstSandCW       CREATE_COMPLETE
TstSandLambda   CREATE_COMPLETE
TstSandIAM      CREATE_COMPLETE
TstSandECSMgmt  CREATE_COMPLETE
TstSandBastion  CREATE_COMPLETE
TstSandVPCEndpoint      CREATE_COMPLETE
```

## S3

### Get the _canonical id_ for an AWS account

```bash
$ aws s3api list-buckets --query Owner.ID
"lkdfjgiewoirlkqejfflkjdlkfjsdlkfasjdlfkjasdfktkbhdkkfkjhfkasjdhf"
```

### Get a list of all S3 buckets in an account

```bash
$ aws s3api list-buckets --query 'Buckets[*].Name' --output text
bucket_01 bucket_02 ....
```

### Enable AES-256 encryption on a bucket

```bash
$ aws s3api put-bucket-encryption --bucket bucketname \
                                  --server-side-encryption-configuration '{ "Rules": [ { "ApplyServerSideEncryptionByDefault": { "SSEAlgorithm": "AES256" } } ] }'
```


### Enable AES-256 encryption on all buckets in an account

Remember that this not change existing object. Only newly created objects n the bucket will be encrypted.

```bash
for bucket in $(aws s3api list-buckets --query 'Buckets[*].Name' --output text)
do
  aws s3api put-bucket-encryption --bucket ${bucket} \
                                  --server-side-encryption-configuration '{ "Rules": [ { "ApplyServerSideEncryptionByDefault": { "SSEAlgorithm": "AES256" } } ] }'
done
```

### Retrieve the value of a named tag on a S3 object

```bash
aws s3api get-object-tagging \
  --bucket ${S3_BUCKET} \
  --key ${S3_KEY} --query "TagSet[*]|[?Key == 'KEYNAME'].Value" --output text
```

## ELBv2

### Show listener rules

#### Step 1: Find loadbalancer arn

```bash
$ aws elbv2 describe-load-balancers --query 'LoadBalancers[*].LoadBalancerArn'
[
    "arn:aws:elasticloadbalancing:eu-central-1:123456789012:loadbalancer/app/lb-dev-ALBExt/2976f0dc67ff78d2",
    "arn:aws:elasticloadbalancing:eu-central-1:123456789012:loadbalancer/app/lb-dev-ALBInt/66b7f2f18789362f"
]
```

#### Step 2: Find the listener arn


```bash
$ aws elbv2 describe-listeners --query 'Listeners[*].ListenerArn' \
     --load-balancer-arn arn:aws:elasticloadbalancing:eu-central-1:123456789012:loadbalancer/app/lb-dev-ALBExt/2976f0dc67ff78d2
[
    "arn:aws:elasticloadbalancing:eu-central-1:123456789012:listener/app/lb-dev-ALBExt/2976f0dc67ff78d2/6536d2cca73d2452",
    "arn:aws:elasticloadbalancing:eu-central-1:123456789012:listener/app/lb-dev-ALBExt/2976f0dc67ff78d2/b8fef364af91e52c"
]
```

#### Step 3: List the listener rules and print the priority

```bash
$ aws elbv2 describe-rules --query 'Rules[*].Priority' \
   --listener-arn arn:aws:elasticloadbalancing:eu-central-1:123456789012:listener/app/lb-dev-ALBExt/2976f0dc67ff78d2/6536d2cca73d2452`
[
    "100",
    "101",
    "102",
    "103",
    "210",
    "211",
    "212",
    "215",
    "216",
    "220",
    "221",
    "default"
]
```

## RDS

### RDS Snapshots

#### Retrieve the ARN of the most recent DB Snapshot

```bash
$ aws rds describe-db-snapshots \
    --query 'reverse(sort_by(DBSnapshots,&SnapshotCreateTime))[0].[DBSnapshotArn][0]' \
    --output text
arn:aws:rds:eu-central-1:123456789012:snapshot:rds:my-db-id-2018-08-26-02-35
```

## SSM

### Retrieve the AMI `image_id` for the recommended ECS AMI

```bash
$ aws ssm get-parameters-by-path \
    --path /aws/service/ecs/optimized-ami/amazon-linux-2/recommended \
    --query "Parameters[*]|[?contains(Name, 'image_id')][Value]" --output text
ami-09577c19fbe1bd7fa
```

## Links and Resources

* [http://jmespath.org/tutorial.html](http://jmespath.org/tutorial.html)
* [Advanced AWS CLI JMESPATH queries](https://opensourceconnections.com/blog/2015/07/27/advanced-aws-cli-jmespath-query/)
