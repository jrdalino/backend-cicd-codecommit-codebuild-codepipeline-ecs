# Backend CICD using CodeCommit CodeBuild CodePipeline ECS

## Step 1: Setup CI/CD for Back End Service

### Step 1.1: Create Codebuild and Codepipeline Role (eks-calculator-codebuild-codepipeline-iam-role)
```
$ cd ~/environment/calculator-backend
$ mkdir aws-cli
$ vi ~/environment/calculator-backend/aws-cli/eks-calculator-codebuild-codepipeline-iam-role.yaml
```
```
---
AWSTemplateFormatVersion: '2010-09-09'
Resources:

  # An IAM role that allows the AWS CodeBuild service to perform the actions
  # required to complete a build of our source code retrieved from CodeCommit,
  # and push the created image to ECR.

  CalculatorServiceCodeBuildServiceRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: CalculatorServiceCodeBuildServiceRole
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          Effect: Allow
          Principal:
            Service: codebuild.amazonaws.com
          Action: sts:AssumeRole
      Policies:
      - PolicyName: "CalculatorService-CodeBuildServicePolicy"
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
            - Fn::Sub: arn:aws:codecommit:${AWS::Region}:${AWS::AccountId}:CalculatorServiceRepository
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

  # An IAM role that allows the AWS CodePipeline service to perform it's
  # necessary actions. 

  CalculatorServiceCodePipelineServiceRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: CalculatorServiceCodePipelineServiceRole
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
      - PolicyName: CalculatorService-codepipeline-service-policy
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
```

```
$ aws cloudformation create-stack \
--stack-name eks-calculator-codebuild-codepipeline-iam-role \
--capabilities CAPABILITY_NAMED_IAM \
--template-body file://~/environment/calculator-backend/aws-cli/eks-calculator-codebuild-codepipeline-iam-role.yaml
```

### Step 1.2: Create an S3 Bucket for Pipeline Artifacts
```
$ aws s3 mb s3://jrdalino-calculator-backend-artifacts
```

### Step 1.3: Modify S3 Bucket Policy
```
$ vi ~/environment/calculator-backend/aws-cli/artifacts-bucket-policy.json
```

```
{
    "Statement": [
      {
        "Sid": "WhitelistedGet",
        "Effect": "Allow",
        "Principal": {
          "AWS": [
            "arn:aws:iam::707538076348:role/CalculatorServiceCodeBuildServiceRole",
            "arn:aws:iam::707538076348:role/CalculatorServiceCodePipelineServiceRole"
          ]
        },
        "Action": [
          "s3:GetObject",
          "s3:GetObjectVersion",
          "s3:GetBucketVersioning"
        ],
        "Resource": [
          "arn:aws:s3:::jrdalino-calculator-backend-artifacts/*",
          "arn:aws:s3:::jrdalino-calculator-backend-artifacts"
        ]
      },
      {
        "Sid": "WhitelistedPut",
        "Effect": "Allow",
        "Principal": {
          "AWS": [
            "arn:aws:iam::707538076348:role/CalculatorServiceCodeBuildServiceRole",
            "arn:aws:iam::707538076348:role/CalculatorServiceCodePipelineServiceRole"
          ]
        },
        "Action": "s3:PutObject",
        "Resource": [
          "arn:aws:s3:::jrdalino-calculator-backend-artifacts/*",
          "arn:aws:s3:::jrdalino-calculator-backend-artifacts"
        ]
      }
    ]
}
```

### Step 1.4: Grant S3 Bucket access to your CI/CD Pipeline
```
$ aws s3api put-bucket-policy \
--bucket jrdalino-calculator-backend-artifacts \
--policy file://~/environment/calculator-backend/aws-cli/artifacts-bucket-policy.json
```

### Step 1.5: View/Modify Buildspec file
```
$ cd ~/environment/calculator-backend
$ vi ~/environment/calculator-backend/buildspec.yml
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
      - printf '[{"name":"calculator-backend","imageUri":"%s"}]' $REPOSITORY_URI:$TAG > imagedefinitions.json
artifacts:
  files: imagedefinitions.json
```

### Step 1.6: View/Modify CodeBuild Project Input File
```
$ vi ~/environment/calculator-backend/aws-cli/code-build-project.json
```

```
{
  "name": "CalculatorBackendServiceCodeBuildProject",
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
        "value": "us-east-1"
      },      
      {
        "name": "IMAGE_REPO_NAME",
        "value": "calculator-backend"
      }
    ],
    "type": "LINUX_CONTAINER"
  },
  "serviceRole": "arn:aws:iam::707538076348:role/CalculatorServiceCodeBuildServiceRole",
  "source": {
    "type": "CODECOMMIT",
    "location": "https://git-codecommit.us-east-1.amazonaws.com/v1/repos/calculator-backend"
  }
}
```

### Step 1.7: Create the CodeBuild Project
```
$ aws codebuild create-project \
--cli-input-json file://~/environment/calculator-backend/aws-cli/code-build-project.json
```

### Step 1.8:  Modify CodePipeline Input File
Replace:
- roleArn = REPLACE_ME_CODEPIPELINE_ROLE_ARN
- location = REPLACE_ME_ARTIFACTS_BUCKET_NAME

```
$ vi ~/environment/calculator-backend/aws-cli/code-pipeline.json
```

```
{
  "pipeline": {
      "name": "CalculatorBackendServiceCICDPipeline",
      "roleArn": "arn:aws:iam::707538076348:role/CalculatorServiceCodePipelineServiceRole",
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
                  "name": "CalculatorBackendService-SourceArtifact"
                }
              ],
              "configuration": {
                "BranchName": "master",
                "RepositoryName": "calculator-backend"
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
                  "name": "CalculatorBackendService-BuildArtifact"
                }
              ],
              "inputArtifacts": [
                {
                  "name": "CalculatorBackendService-SourceArtifact"
                }
              ],
              "configuration": {
                "ProjectName": "CalculatorBackendServiceCodeBuildProject"
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
                            "name": "CalculatorBackendService-BuildArtifact"
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
                        "ClusterName": "MythicalMysfits-Cluster", 
                        "ServiceName": "MythicalMysfits-Service", 
                        "FileName": "imagedefinitions.json"
                    }, 
                    "runOrder": 1
                }
            ]
        }
      ],
      "artifactStore": {
        "type": "S3",
        "location": "jrdalino-calculator-backend-artifacts"
      }
  }
}
```

### Step 1.13: Create a pipeline in CodePipeline
```
$ aws codepipeline create-pipeline \
--cli-input-json file://~/environment/calculator-backend/aws-cli/code-pipeline.json
```

### Step 1.14: Create ECR Policy File and Enable automated Access to the ECR Image Repository
Replace:
- REPLACE_ME_CODEBUILD_ROLE_ARN = arn:aws:iam::707538076348:role/CalculatorServiceCodeBuildServiceRole
```
$ vi ~/environment/calculator-backend/aws-cli/ecr-policy.json
```

```
{
  "Statement": [
    {
      "Sid": "AllowPushPull",
      "Effect": "Allow",
      "Principal": {
        "AWS": [
         "REPLACE_ME_CODEBUILD_ROLE_ARN"
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
--repository-name mythicalmysfits/service \
--policy-text file://~/environment/calculator-backend/aws-cli/ecr-policy.json
```

### Step 1.15: Make a small code change, push and validate changes

### (Optional) Clean up
```
$ aws codepipeline delete-pipeline --name CalculatorBackendServiceCICDPipeline
$ rm ~/environment/calculator-backend/aws-cli/code-pipeline.json
$ aws lambda delete-function --function-name LambdaKubeClient
$ rm ~/environment/lambda-package_v1.zip
$ aws codebuild delete-project --name CalculatorBackendServiceCodeBuildProject
$ Manually delete codebuild history
$ aws s3api delete-bucket-policy --bucket jrdalino-calculator-backend-artifacts
$ rm ~/environment/calculator-backend/aws-cli/artifacts-bucket-policy.json
$ aws s3 rm s3://jrdalino-calculator-backend-artifacts --recursive
$ aws s3 rb s3://jrdalino-calculator-backend-artifacts --force
$ aws lambda delete-function --function-name LogsToElasticsearch_kubernetes-logs
$ aws logs delete-log-group --log-group-name /aws/lambda/LambdaKubeClient
$ aws logs delete-log-group --log-group-name /aws/codebuild/CalculatorBackendServiceCodeBuildProject
```
