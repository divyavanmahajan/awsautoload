# awsautoload
Cloud formation and instructions to setup AWS Redshift loader

+ Clone this repo
```bash
    git clone https://github.com/divyavanmahajan/awsautoload.git
```
+ Setup the command line variables. 
```bash
    unset AWS_REGION
    unset AWS_PROFILE
    export AWS_DEFAULT_PROFILE=myprofile
    export AWS_DEFAULT_REGION=us-west-2
```
+ Create two buckets (if you don't already have them) in the same region as where you will create the stack. 
The Stack and Data bucket must be in the same region as the Redshift cluster.
```bash
    export CODEBUCKET=testbucket-mycode1
    export DATABUCKET=testbucket-mydata1
    cd awsautoload
    aws s3api create-bucket --bucket $CODEBUCKET --region $AWS_DEFAULT_REGION
    aws s3api create-bucket --bucket $DATABUCKET --region $AWS_DEFAULT_REGION
    aws s3 cp dist/AWS*zip s3://$CODEBUCKET/AWSLambdaRedshiftLoader.zip
    aws s3 cp dist/SFDC*zip s3://$CODEBUCKET/SFDCExtractorLambda.zip
```
+ Execute the cloudformation script to create the functions. 
  This creates an IAM policy, IAM role, 2 Lambda functions, 2 SNS topics
   and permissions to let S3 notify the Lambda function.

```bash
    export STACKNAME=mystack
    export TOPICPREFIX=load
    aws s3 cp cf-templates/createfunctions.template s3://$CODEBUCKET/createfunctions.template
    aws cloudformation create-stack --stack-name $STACKNAME \
      --template-url  https://s3.amazonaws.com/$CODEBUCKET/createfunctions.template \
      --capabilities CAPABILITY_IAM \
      --parameters ParameterKey=LambdaZipBucket,ParameterValue=$CODEBUCKET ParameterKey=TopicPrefix,ParameterValue=$TOPICPREFIX  
    aws cloudformation describe-stacks --stack-name $STACKNAME
```
+ When the StackStatus is "CREATE_COMPLETE", note down the output of the stack.
```json
    "Outputs": [
        {
            "OutputKey": "SNSOkayTopicARN", 
            "OutputValue": "arn:aws:sns:us-west-2:404268877227:load-ok"
        }, 
        {
            "OutputKey": "SNSFailTopicARN", 
            "OutputValue": "arn:aws:sns:us-west-2:404268877227:load-fail"
        }, 
        {
            "OutputKey": "LambdaLoaderFunction", 
            "OutputValue": "arn:aws:lambda:us-west-2:404268877227:function:lambdaAWSLoader"
        }, 
        {
            "OutputKey": "SFDCExtractorFunction", 
            "OutputValue": "arn:aws:lambda:us-west-2:404268877227:function:lambdaSFDCExtractor"
        }
    ]
```
+ If your Lambda Loader functions need to access resources in a VPC 
(e.g. your Redshift cluster is inside a VPC and not accessible on the Internet); manually edit the 
VPC configuration to select the VPC and subnets. The execution role has the rights needed to execute in a VPC.
