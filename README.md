# Real World AWS CLI Examples

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
$ aws cloudformation list-stacks --query "StackSummaries[?StackStatus=='CREATE_COMPLETE'].[StackName, StackStatus]" --output text
DevECS	CREATE_COMPLETE
DevLBALBInt	CREATE_COMPLETE
SecuritySubaccount	CREATE_COMPLETE
BastionForDev	CREATE_COMPLETE
DevRoute53	CREATE_COMPLETE
```

## S3

### Get a list of all S3 buckets in an account

```
$ aws s3api list-buckets --query 'Buckets[*].Name' --output text
bucket_01 bucket_02 ....
```

### Enable AES-256 encryption on a bucket

```
$ aws s3api put-bucket-encryption --bucket bucketname \
                                  --server-side-encryption-configuration '{ "Rules": [ { "ApplyServerSideEncryptionByDefault": { "SSEAlgorithm": "AES256" } } ] }'
```


### Enable AES-256 encryption on all buckets in an account

Remember that this not change existing object. Only newly created objects n the bucket will be encrypted.

```
for bucket in $(aws s3api list-buckets --query 'Buckets[*].Name' --output text)
do
  aws s3api put-bucket-encryption --bucket ${bucket} \
                                  --server-side-encryption-configuration '{ "Rules": [ { "ApplyServerSideEncryptionByDefault": { "SSEAlgorithm": "AES256" } } ] }'
done
```

## ELBv2

### Show listener rules

#### Step 1: Find loadbalancer arn

```
$ aws elbv2 describe-load-balancers --query 'LoadBalancers[*].LoadBalancerArn'
[
    "arn:aws:elasticloadbalancing:eu-central-1:123456789012:loadbalancer/app/lb-dev-ALBExt/2976f0dc67ff78d2",
    "arn:aws:elasticloadbalancing:eu-central-1:123456789012:loadbalancer/app/lb-dev-ALBInt/66b7f2f18789362f"
]
```

#### Step 2: Find the listener arn


```
$ aws elbv2 describe-listeners --query 'Listeners[*].ListenerArn' \
     --load-balancer-arn arn:aws:elasticloadbalancing:eu-central-1:123456789012:loadbalancer/app/lb-dev-ALBExt/2976f0dc67ff78d2
[
    "arn:aws:elasticloadbalancing:eu-central-1:123456789012:listener/app/lb-dev-ALBExt/2976f0dc67ff78d2/6536d2cca73d2452",
    "arn:aws:elasticloadbalancing:eu-central-1:123456789012:listener/app/lb-dev-ALBExt/2976f0dc67ff78d2/b8fef364af91e52c"
]
```

#### Step 3: List the listener rules and print the priority

```
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

## Links and Resources

* [http://jmespath.org/tutorial.html](http://jmespath.org/tutorial.html)