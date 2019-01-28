# CodePipeline-Nested-CFN

This repo contains the CloudFormtaion template which will create a CodePipeline containing multiple stages starting from CodeCommit as source stage, followed by build using CodeBuild. Once finished it proceed to production stage where it creates a CloudFormation ChangeSet for production stack and wait for approval, once approved it will execute the ChangeSet in production stack.

![CodePipeline Design](images/Pipeline_Design.png)

Let's start by creating the repositories and enabling Continuous Delivery pipeline for nested CFN.

## Step 1:

### Create base VPC Stack
In the [nested-templates](nested-templates/) directory there are multiple YAML (*CloudFormation Templates*) & JSON (*CloudFormation Configuration*) files.

**[vpc-stack.yml](nested-templates/vpc-stack.yml):** is the CloudFormation template to create the base VPC, Subnets, NAT Gateways, etc which will be used.
**[vpc-params.json](nested-templates/vpc-params.json):** is the parameters file which contains the parameter values for the CFN template. Update the *ProdApprovalEmail* values to provide the appropriate email address.

Go to `nested-templates` directory and execute the following AWS CLI command to create CloudFormation stack.

```bash
cd nested-templates
aws cloudformation create-stack --stack-name NestedCF-VPC --template-body file://vpc-stack.yml --parameters file://vpc-params.json
```

## Step 2:

### Update CloudFormation parameters configuration files
In the [nested-templates](nested-templates/) directory there is (*CloudFormation Configuration*) files.

**[config-prod.json](nested-templates/config-prod.json):** - CloudFormation parameter configuration file for Prod stack

Update this configuration file with appropriate values for *VPCID, PrivateSubnet1, PrivateSubnet2, PublicSubnet1, PublicSubnet2, S3BucketName & DBSubnetGroup* based on the values in the output section of the base VPC stack created in Step 1. Update *KeyPair* value with an existing key pair or create a new key pair and use it.

## Step 3:

### Creating CodeCommit repositories
Create CodeCommit repositories as mentioned below.

```bash
aws codecommit create-repository --repository-name nested-templates --repository-description "Repository for CloudFormation templates"
```

Once the repositories are create, clone those repositories and upload the content of directories `nested-templates`

## Step 4:

### Creating CodePipeline using CloudFormation

Update the **[template-params.json](template-params.json)** file with the appropriate values for *ArtifactS3Location & ProdTopic* based on the values from output section of main stack created in Step 1 and update the values for *SourceRepoName* with appropriate values based on the repositories created in Step 3.

Once the configuration file has been updated, execute the following command to create the CloudFormation stack which will create the required CodePipeline.

```bash
aws cloudformation create-stack --stack-name NestedCF-CodePipeline --template-body file://codepipeline-cfn-codebuild.yml --parameters file://codepipeline-cfn-codebuild.json --capabilities CAPABILITY_NAMED_IAM
```

Once the CloudFormation successfully creates the stack, it would have created a CodePipeline with similar stages as shown below.

![CodePipeline Stages](images/Pipeline_Flow.png)

_Note: While removing the resources, delete the Prod & UAT stacks created by pipeline before deleting the pipeline since those CloudFormations stacks uses the role created by pipeline stack._
