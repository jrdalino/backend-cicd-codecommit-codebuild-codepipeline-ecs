# My Project Product Backend REST API CodePipeline

## Step 1: Setup CI/CD for Back End Service

### Step 1.1: Create an S3 Bucket for Pipeline Artifacts
```
$ aws s3 mb s3://jrdalino-myproject-product-restapi-artifacts
```

### Step 1.2: Create Codebuild and Codepipeline Role using CloudFormation
```
$ cd ~/environment/myproject-product-restapi
$ mkdir aws-cfn
$ vi ~/environment/myproject-product-restapi/aws-cfn/myproject-product-restapi-codepipeline-service-role-stack.yml
```
```
---
AWSTemplateFormatVersion: '2010-09-09'
Description: This stack deploys the IAM Role for Codebuild and Codepipeline
Resources:
  MyprojectProductRestapiCodebuildServiceRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: MyprojectProductRestapiCodebuildServiceRole
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          Effect: Allow
          Principal:
            Service: codebuild.amazonaws.com
          Action: sts:AssumeRole
      Policies:
      - PolicyName: "MyprojectProductRestapiCodebuildServicePolicy"
        PolicyDocument:
          Version: "2012-10-17"
          Statement:
          - Effect: "Allow"
            Action:
            - "codecommit:ListBranches"
            - "codecommit:ListRepositories"
            - "codecommit:BatchGetRepositories"
            - "codecommit:Get*"
            - "codecommit:GitPull"
            Resource:
            - Fn::Sub: arn:aws:codecommit:${AWS::Region}:${AWS::AccountId}:myproject-product-restapi
          - Effect: "Allow"
            Action:
            - "logs:CreateLogGroup"
            - "logs:CreateLogStream"
            - "logs:PutLogEvents"
            Resource: "*"
          - Effect: "Allow"
            Action:
            - "s3:PutObject"
            - "s3:GetObject"
            - "s3:GetObjectVersion"
            - "s3:ListBucket"
            Resource: "*"
          - Effect: "Allow"
            Action:
            - "ecr:GetAuthorizationToken"
            - "ecr:InitiateLayerUpload"
            - "ecr:UploadLayerPart"
            - "ecr:CompleteLayerUpload"
            - "ecr:BatchCheckLayerAvailability"
            - "ecr:PutImage"
            Resource: "*"

  MyprojectProductRestapiCodepipelineServiceRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: MyprojectProductRestapiCodepipelineServiceRole
      AssumeRolePolicyDocument:
        Statement:
        - Effect: Allow
          Principal:
            Service:
            - codepipeline.amazonaws.com
          Action:
          - sts:AssumeRole
      Path: "/"
      Policies:
      - PolicyName: MyprojectProductRestapiCodepipelineServicePolicy
        PolicyDocument:
          Statement:
          - Action:
            - codecommit:GetBranch
            - codecommit:GetCommit
            - codecommit:UploadArchive
            - codecommit:GetUploadArchiveStatus
            - codecommit:CancelUploadArchive
            Resource: "*"
            Effect: Allow
          - Action:
            - s3:GetObject
            - s3:GetObjectVersion
            - s3:GetBucketVersioning
            Resource: "*"
            Effect: Allow
          - Action:
            - s3:PutObject
            Resource:
            - arn:aws:s3:::*
            Effect: Allow
          - Action:
            - elasticloadbalancing:*
            - autoscaling:*
            - cloudwatch:*
            - ecs:*
            - eks:*
            - codebuild:*
            - codepipeline:*
            - codedeploy:*
            - iam:ListRoles	    
            - iam:PassRole
            - lambda:*
            - sns:*
            Resource: "*"
            Effect: Allow
          Version: "2012-10-17"

Outputs:
  CodeBuildRole:
    Description: REPLACE_ME_CODEBUILD_ROLE_ARN
    Value: !GetAtt 'MyprojectProductRestapiCodebuildServiceRole.Arn'
    Export:
      Name: !Join [ ':', [ !Ref 'AWS::StackName', 'MyprojectProductRestapiCodebuildServiceRole' ] ]
  CodePipelineRole:
    Description: REPLACE_ME_CODEPIPELINE_ROLE_ARN
    Value: !GetAtt 'MyprojectProductRestapiCodepipelineServiceRole.Arn'
    Export:
      Name: !Join [ ':', [ !Ref 'AWS::StackName', 'MyprojectProductRestapiCodepipelineServiceRole' ] ]       
```

## Step 1.3: Create the Stack
```
$ aws cloudformation create-stack \
--stack-name MyprojectProductRestapiCodepipelineServiceRoleStack \
--capabilities CAPABILITY_NAMED_IAM \
--template-body file://~/environment/myproject-product-restapi/aws-cfn/myproject-product-restapi-codepipeline-service-role-stack.yml
```

### Step 1.4: Create S3 Bucket Policy File
```
$ cd ~/environment/myproject-product-restapi
$ mkdir aws-cli
$ vi ~/environment/myproject-product-restapi/aws-cli/artifacts-bucket-policy.json
```

```
{
    "Statement": [
      {
        "Sid": "WhitelistedGet",
        "Effect": "Allow",
        "Principal": {
          "AWS": [
            "arn:aws:iam::707538076348:role/MyprojectProductRestapiCodebuildServiceRole",
            "arn:aws:iam::707538076348:role/MyprojectProductRestapiCodepipelineServiceRole"
          ]
        },
        "Action": [
          "s3:GetObject",
          "s3:GetObjectVersion",
          "s3:GetBucketVersioning"
        ],
        "Resource": [
          "arn:aws:s3:::jrdalino-myproject-product-restapi-artifacts/*",
          "arn:aws:s3:::jrdalino-myproject-product-restapi-artifacts"
        ]
      },
      {
        "Sid": "WhitelistedPut",
        "Effect": "Allow",
        "Principal": {
          "AWS": [
            "arn:aws:iam::707538076348:role/MyprojectProductRestapiCodebuildServiceRole",
            "arn:aws:iam::707538076348:role/MyprojectProductRestapiCodepipelineServiceRole"
          ]
        },
        "Action": "s3:PutObject",
        "Resource": [
          "arn:aws:s3:::jrdalino-myproject-product-restapi-artifacts/*",
          "arn:aws:s3:::jrdalino-myproject-product-restapi-artifacts"
        ]
      }
    ]
}
```

### Step 1.5: Grant S3 Bucket access to your CI/CD Pipeline
```
$ aws s3api put-bucket-policy \
--bucket jrdalino-myproject-product-restapi-artifacts \
--policy file://~/environment/myproject-product-restapi/aws-cli/artifacts-bucket-policy.json
```

### Step 1.6: Create Buildspec file
```
$ cd ~/environment/myproject-product-restapi
$ vi ~/environment/myproject-product-restapi/buildspec.yml
```

```
version: 0.2
phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - $(aws ecr get-login --no-include-email --region $AWS_DEFAULT_REGION)
      - TAG="$(date +%s)"
      - REPOSITORY_URI="$AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME"
  build:
    commands:
      - echo Build started on `date`
      - echo Building the Docker image...
      - docker build -t $REPOSITORY_URI:$TAG .
  post_build:
    commands:
      - echo Uploading the Docker image...
      - docker push $REPOSITORY_URI:$TAG
      - printf '[{"name":"myproject-product-restapi","imageUri":"%s"}]' $REPOSITORY_URI:$TAG > imagedefinitions.json
artifacts:
  files: imagedefinitions.json
```

### Step 1.7: Create CodeBuild Project Input File
```
$ vi ~/environment/myproject-product-restapi/aws-cli/codebuild.json
```

```
{
  "name": "MyprojectProductRestapiCodebuild",
  "artifacts": {
    "type": "no_artifacts"
  },
  "environment": {
    "computeType": "BUILD_GENERAL1_SMALL",
    "image": "aws/codebuild/python:3.5.2",
    "privilegedMode": true,
    "environmentVariables": [
      {
        "name": "AWS_ACCOUNT_ID",
        "value": "707538076348"
      },
      {
        "name": "AWS_DEFAULT_REGION",
        "value": "ap-southeast-1"
      },      
      {
        "name": "IMAGE_REPO_NAME",
        "value": "myproject-product-restapi"
      }
    ],
    "type": "LINUX_CONTAINER"
  },
  "serviceRole": "arn:aws:iam::707538076348:role/MyprojectProductRestapiCodebuildServiceRole",
  "source": {
    "type": "CODECOMMIT",
    "location": "https://git-codecommit.ap-southeast-1.amazonaws.com/v1/repos/myproject-product-restapi"
  }
}
```

### Step 1.8: Create the CodeBuild Project
```
$ aws codebuild create-project \
--cli-input-json file://~/environment/myproject-product-restapi/aws-cli/codebuild.json
```

### Step 1.9: Create CodePipeline Input File
```
$ vi ~/environment/myproject-product-restapi/aws-cli/codepipeline.json
```

```
{
  "pipeline": {
      "name": "MyprojectProductRestapiCodepipeline",
      "roleArn": "arn:aws:iam::707538076348:role/MyprojectProductRestapiCodepipelineServiceRole",
      "stages": [
        {
          "name": "Source",
          "actions": [
            {
              "inputArtifacts": [
    
              ],
              "name": "Source",
              "actionTypeId": {
                "category": "Source",
                "owner": "AWS",
                "version": "1",
                "provider": "CodeCommit"
              },
              "outputArtifacts": [
                {
                  "name": "myproject-product-restapi-source-artifact"
                }
              ],
              "configuration": {
                "BranchName": "master",
                "RepositoryName": "myproject-product-restapi"
              },
              "runOrder": 1
            }
          ]
        },
        {
          "name": "Build",
          "actions": [
            {
              "name": "Build",
              "actionTypeId": {
                "category": "Build",
                "owner": "AWS",
                "version": "1",
                "provider": "CodeBuild"
              },
              "outputArtifacts": [
                {
                  "name": "myproject-product-restapi-build-artifact"
                }
              ],
              "inputArtifacts": [
                {
                  "name": "myproject-product-restapi-source-artifact"
                }
              ],
              "configuration": {
                "ProjectName": "MyprojectProductRestapiCodebuild"
              },
              "runOrder": 1
            }
          ]
        },
        {
            "name": "Deploy", 
            "actions": [
                {
                    "inputArtifacts": [
                        {
                            "name": "myproject-product-restapi-build-artifact"
                        }
                    ], 
                    "name": "Deploy", 
                    "actionTypeId": {
                        "category": "Deploy", 
                        "owner": "AWS", 
                        "version": "1", 
                        "provider": "ECS"
                    }, 
                    "outputArtifacts": [], 
                    "configuration": {
                        "ClusterName": "Myproject-ECS-Cluster", 
                        "ServiceName": "Myproject-ECS-Service", 
                        "FileName": "imagedefinitions.json"
                    }, 
                    "runOrder": 1
                }
            ]
        }
      ],
      "artifactStore": {
        "type": "S3",
        "location": "jrdalino-myproject-product-restapi-artifacts"
      }
  }
}
```

### Step 1.10: Create the pipeline
```
$ aws codepipeline create-pipeline \
--cli-input-json file://~/environment/myproject-product-restapi/aws-cli/codepipeline.json
```

### Step 1.11: Create ECR Policy File and Enable automated Access to the ECR Image Repository
```
$ vi ~/environment/myproject-product-restapi/aws-cli/ecr-policy.json
```

```
{
  "Statement": [
    {
      "Sid": "AllowPushPull",
      "Effect": "Allow",
      "Principal": {
        "AWS": [
         "arn:aws:iam::707538076348:role/MyprojectProductRestapiCodebuildServiceRole"
        ]
      },
      "Action": [
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:BatchCheckLayerAvailability",
        "ecr:PutImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload"
      ]
    }
  ]
}
```

```
$ aws ecr set-repository-policy \
--repository-name myproject-product-restapi \
--policy-text file://~/environment/myproject-product-restapi/aws-cli/ecr-policy.json
```

### Step 1.12: Make a small code change, push and validate changes

### (Optional) Clean up
```
$ aws codepipeline delete-pipeline --name MyprojectProductRestapiCodepipeline
$ rm ~/environment/myproject-product-restapi/aws-cli/codepipeline.json
$ aws lambda delete-function --function-name LambdaKubeClient
$ rm ~/environment/lambda-package_v1.zip
$ aws codebuild delete-project --name MyprojectProductRestapiCodebuild
Manually delete codebuild history
$ aws s3api delete-bucket-policy --bucket jrdalino-myproject-product-restapi-artifacts
$ rm ~/environment/myproject-product-restapi/aws-cli/artifacts-bucket-policy.json
$ aws s3 rm s3://jrdalino-myproject-product-restapi-artifacts --recursive
$ aws s3 rb s3://jrdalino-myproject-product-restapi-artifacts --force
$ aws lambda delete-function --function-name LogsToElasticsearch_kubernetes-logs
$ aws logs delete-log-group --log-group-name /aws/lambda/LambdaKubeClient
$ aws logs delete-log-group --log-group-name /aws/codebuild/MyprojectProductRestapiCodebuild
```
