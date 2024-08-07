# Deploy Lambda function in Localstack with AWS CDK

## Overview

1. Create custom image of localstack to save a few steps
2. Start docker container
3. Initial setup
4. Edit cdk file
5. Deploy
6. Test 

-----------------------------------------
## Create custom image

This is to save the steps 
* install aws-cdk
* set aliases
* create work directory

I used a help from ChatGPT.

```
# Start from the official localstack image
FROM localstack/localstack

# Set environment variables for the AWS region and default output format
ENV AWS_DEFAULT_REGION=eu-central-1
ENV AWS_DEFAULT_OUTPUT=yaml

# Install necessary packages
RUN apt-get update && \
    apt-get install -y npm python3-pip && \
    apt-get clean

# Install aws-cdk-local and aws-cdk globally using npm
RUN npm install -g aws-cdk-local aws-cdk

# Create an alias for cdklocal
RUN echo 'alias c="cdklocal"' >> ~/.bashrc
RUN echo 'alias a="awslocal"' >> ~/.bashrc

# Create the /aws directory
RUN mkdir -p /aws
RUN echo 'cd /aws' >> ~/.bashrc

# Expose the port that LocalStack listens on (default is 4566)
EXPOSE 4566

# Set the entrypoint to LocalStack
ENTRYPOINT ["docker-entrypoint.sh"]

# Command to start LocalStack
#
```
Build the image.
```
> docker build -t localstack-cdk .
```

Check if the image is built.
```
> docker images
REPOSITORY                     TAG       IMAGE ID       CREATED         SIZE
localstack-cdk                 latest    d72434532e1a   2 days ago      1.89GB
```
-----------------------------------------
## Start image

Move to your work directory. If this is the first time to start the work, make 
sure the directory is empty.

```
> mkdir e5 # for instance
> cd e5
```


Start the image.  
```
> docker run \
  --rm  -d \
  --name cdk \
  -p 127.0.0.1:4566:4566 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v ~/.aws:/root/.aws \
  -v $(pwd):/aws \
  localstack-cdk
```

This command 
* runs container in the background
* will remove the container when you stop it
* names the container 'cdk'
* expose the port 4566 to the local (=your laptop) 4566
* gives access to docker server to the container (insecure)
* mounts .aws in your home directory to /root/ of the container
* mounts the current directory to /aws of the container 

Let us check if the container is running.
```
> docker ps
CONTAINER ID   IMAGE            COMMAND                  CREATED        STATUS                  PORTS                                               NAMES
c24f179ec9fa   localstack-cdk   "docker-entrypoint.sh"   15 hours ago   Up 15 hours (healthy)   4510-4559/tcp, 5678/tcp, 127.0.0.1:4566->4566/tcp   cdk
```

Get into the container, and move to the working directory /aws.

```
> docker exec -it cdk bash
...
> cd /aws
root@c24f179ec9fa:/aws#
```

-----------------------------------------
## Initial setup

If this is the first time to start that container, do the following.

```
c init -l typescript
```
c is aliased to cdklocal
 
This will set up the working directory as follows. 
```
> tree -L  1
.
├── README.md
├── aws-stack-cdk.yaml
├── bin
├── cdk.json
├── cdk.out
├── jest.config.js
├── lib
├── node_modules
├── package-lock.json
├── package.json
├── test
└── tsconfig.json
```

__NOTE__ 
* As your work directory is mounted to the container, all the rest, 
like editing the code, checking the directory, you can do on your laptop.
* Make sure you have '.git' directory as well.

```
 ls -1 .git
description
branches
info
hooks
HEAD
index
COMMIT_EDITMSG
logs
objects
config
refs
```

Then create a (mock) S3 storage that is required for (mock) AWS CloudFront.

```
c bootstrap
```

Now we are ready to work with the code.

-----------------------------------------
## Edit cdk file

We will follow the official [quick tutorial](https://docs.aws.amazon.com/cdk/v2/guide/hello_world.html) of AWS.

The codes are split in ./lib and ./bin.

```./bin``` stores 'App' which is the upper level grouping of 'Stack'. 
'Stack' is stored in ```./lib```. We will work with ```./lib/aws-stack.ts```. 

Make sure the codes ```./bin/aws.ts``` and ```./lib/aws-stack.ts``` are consistent.
For instance, 

``` ./lib/aws-stack.ts
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as lambda from 'aws-cdk-lib/aws-lambda';
// import * as sqs from 'aws-cdk-lib/aws-sqs';


export class AwsStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);
...
```
Here you cannot change the name ```AwsStack``` as it is defined in ```./bin/aws.ts```.


We will 

1. Create a construct for Lambda function. (Step 4. of [quick tutorial](https://docs.aws.amazon.com/cdk/v2/guide/hello_world.html))
2. Create a URL for Lambda function. (Step 5)
3. Write the URL to stdout when the code is deployed. (So that we can execute the Lambda function)

See the code available on the AWS tutorial. 

-----------------------------------------
## Deploy
```
# c deploy
✨  Synthesis time: 8.43s

AwsStack:  start: Building 
...
AwsStack:  success: Built 
...
AwsStack:  start: Publishing 
...
AwsStack:  success: Published 
...
AwsStack: deploying... [1/1]
AwsStack: creating CloudFormation changeset...

 ✅  AwsStack

✨  Deployment time: 10.12s

Outputs:
AwsStack.myFunctionUrlOutput = http://XXXX.....XX:4566
Stack ARN:
arn:aws:cloudformation:XXXXX
✨  Total time: 18.55s
```

Check if the stack is deployed.
```
# c list
AwsStack
```
Check the resources deployed.

```
# AWS_DEFAULT_OUTPUT=json a cloudformation describe-stack-resources --stack-name AwsStack 
```

Need ```jq``` (Note to myself -> put it in Dockerfile).

```
apt install jq
```
(Somehow ```jq``` does not work well)

```
# AWS_DEFAULT_OUTPUT=json a cloudformation describe-stack-resources --stack-name AwsStack | grep ResourceType

            "ResourceType": "AWS::IAM::Role",
            "ResourceType": "AWS::CDK::Metadata",
            "ResourceType": "AWS::Lambda::Function",
            "ResourceType": "AWS::Lambda::Url",
            "ResourceType": "AWS::Lambda::Permission",
```

-----------------------------------------
## Test 
```curl``` the URL.

```
> curl  http://URL_you_got
HelloWorld
```

Change the message and reploy.

```
c deploy
```

Destroy the stack
```
c destroy
```

-----------------------------------------
# Appendix (Original README.md)
## Welcome to your CDK TypeScript project 

This is a blank project for CDK development with TypeScript.

The `cdk.json` file tells the CDK Toolkit how to execute your app.

## Useful commands

* `npm run build`   compile typescript to js
* `npm run watch`   watch for changes and compile
* `npm run test`    perform the jest unit tests
* `npx cdk deploy`  deploy this stack to your default AWS account/region
* `npx cdk diff`    compare deployed stack with current state
* `npx cdk synth`   emits the synthesized CloudFormation template

-----------------------------------------
# END