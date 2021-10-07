# How to find the latest AMI deployed by EC2 Image Builder

## A solution to automate AWS Lambda functions as Custom Resources to allow lookup of AMIs deployed (or shared) using Image Builder.

[EC2 Image Builder](https://aws.amazon.com/image-builder/) is a service available on AWS to automate the building and deploying of Custom AMIs across AWS regions and AWS accounts. It allows an organisation to build *approved* AMIs (Amazon Machine images) in a central account and push them out to be consumed by other AWS accounts and other regions.

The EC2 Image Builder has the concept of a **Distribution Configuration** that is used to define which Regions and AWS accounts the newly built AMIs should be deployed to. The deployment can be achieved in one of two ways:

1) By **copying** the AMI to the target accounts/regions, this means multiple copies of the AMI are produced and the target accounts/regions now have their own copy separate to the source account.
2) By **sharing** the AMI with the target accounts/regions, this means only one copy of the AMI exists per region and the other accounts use the AMI as a shared resource.

Due to this great flexibility EC2 Image Builder is a very popular AWS Service. However, if the target accounts use [AWS CloudFormation](https://aws.amazon.com/cloudformation/) to deploy their solutions, they often struggle to programatically determine the latest *approved* AMI to use. This results in different solutions being built in different accounts to solve the same problem.

This article supplies a solution to this problem. It does so by making use of [CloudFormation Custom Resources](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-custom-resources.html) in the target accounts to be able to *lookup* the latest AMI whenever needed. However, Custom Resources can be tricky to create and maintain since they need someone to create the code/logic to perform the lookup. In this solution we will be showing you how to deploy this code centrally once (per region) in the AWS account that runs the Image Builder and give access to the target accounts to use it. Therefore, no additional resources need to be deployed into the target accounts they just need the correct syntax to be able to make use of it. Another part of this solution will show how we *link* the **Distribution Configuration** to this solution so that when any changes are made to the target accounts/regions the resources are updated automatically without any human intervention.

So, lets' get started with an overview of the components in this solution:

- AWS CloudWatch Event
  - This event will be triggered (using CloudTrail) when the `UpdateDistributionConfiguration` API is called. This allows us to track any changes to a predefined Distribution Configuration(s).
- AWS Lambda function
  - This function is the "brains" of the solution, it makes sure that the resources required are deployed correctly and sync'd with the Distribution Configuration.
- CloudFormation StackSet
  - This StackSet will be used to deploy the regional components of the solution.
  - The Stacks within the StackSet will consist of a further regional Lambda function that is the function called by the CloudFormation Custom Resource in the target accounts/regions.

The entire solution will be deployed with a single CloudFormation Stack. Here is a diagram of the solution.

![Diagram](./images/ImageBuilder-Automation.png)

## 1 - Deploying the solution into an account.

We will deploy the solution using the [AWS CLI](https://aws.amazon.com/cli/)

First you will need to download the correct template from this repository for your use case.
- [`template-copied-amis.yaml`](https://github.com/danjhd/imagebuilder-customresource/raw/main/template-copied-amis.yaml) for where your Distribution Configuration is setup to **copy** AMIs into the target accounts/regions
- [`template-shared-amis.yaml`](https://github.com/danjhd/imagebuilder-customresource/raw/main/template-shared-amis.yaml) for where your Distribution Configuration is setup to **share** AMIs into the target accounts/regions

Download the corresponding file above and save it somewhere locally. You will need it a little later.

Next we need to collect the ARN(s) of the **Distribution Configuration(s)** we wish to automate with this solution. Execute the following (replacing <samp><mark>REGION</mark></samp> with your region of choice):

<pre>
aws imagebuilder list-distribution-configurations --query "distributionConfigurationSummaryList[*].[name,arn]" --output table --region <mark>REGION</mark>
</pre>

You should see an output that lists all the **Distribution Configurations** in your account in this region. Make a note of the ARN(s) of the required Distribution Configurations as we will need these in the next command.

<pre>
-----------------------------------------------------------------------------------------</br>|                            ListDistributionConfigurations                             |</br>+------+--------------------------------------------------------------------------------+</br>|  Test|  <b>arn:aws:imagebuilder:eu-west-1:111122223333:distribution-configuration/test</b>   |</br>+------+--------------------------------------------------------------------------------+
</pre>

We will now create a new CloudFormation stack in the same region as Image Builder. Again we will use the AWS CLI to do this. The command is shown below. You will need to change the <samp><mark>filename</mark></samp> of the yaml template to the one you downloaded, replace <samp><mark>REGION</mark></samp> again and also supply the <samp><mark>ARNs</mark></samp> for your Distribution Configuration(s). Use commas to separate multiple ARNs.

<pre>
aws cloudformation deploy --template-file <mark>template-shared-amis.yaml</mark> --stack-name ImageBuilderAmiLookupCustomResource --parameter-overrides DistributionArns=<mark>arn:aws:imagebuilder:eu-west-1:111122223333:distribution-configuration/test</mark> --capabilities CAPABILITY_IAM --region <mark>REGION</mark>
</pre>

The stack creation should take a couple of minutes. Once the stack is created we now have the base infrastructure in place.

## 2 - Initial population

It is assumed this solution will be deployed into an account with an existing Image Builder in use. In this scenario we need to trigger the automation for the first time to create all the required Custom Resource Lambda functions and permissions to match the current in use Image Builder.

To do this we will use the CLI again to describe the existing Distribution Configuration(s) and then update it to the same values. This will make no changes to the Distribution Configuration(s) but will result in a call to the `UpdateDistributionConfiguration` API which in turn will trigger the automation.

Excute the following command, replacing the <samp><mark>ARNs</mark></samp> and <samp><mark>REGION</mark></samp> as required:

---
**NOTE: Repeat the following command for each Distribution Configurations**

---

<pre>
aws imagebuilder get-distribution-configuration --distribution-configuration-arn <mark>arn:aws:imagebuilder:eu-west-1:111122223333:distribution-configuration/test</mark> --query "distributionConfiguration.distributions" --output json --region <mark>REGION</mark> > distributions.json

aws imagebuilder update-distribution-configuration --distribution-configuration-arn <mark>arn:aws:imagebuilder:eu-west-1:111122223333:distribution-configuration/test</mark> --distributions file://distributions.json --region <mark>REGION</mark>
</pre>

## 3 - Review the automation

We are now able to view the logs of the Lambda function that was executed to show what action was taken. The easiest way to find the logs is to get the outputs from the CloudFormation stack that we created with the following command:

<pre>
aws cloudformation describe-stacks --stack-name ImageBuilderAmiLookupCustomResource --query "Stacks[0].Outputs[0].OutputValue"  --output text --region <mark>REGION</mark>
</pre>

This will return a URL similar to this:
<pre>
https://<mark>REGION</mark>.console.aws.amazon.com/cloudwatch/home?region=<mark>REGION</mark>#logsV2:log-groups/log-group/$252Faws$252Flambda$252FImageBuilderAmiLookupCustomReso-AutomationFunction
</pre>

Put this URL into a browser to go to the CloudWatch Logs page. Once in the CloudWatch console locate the log streams towards the bottom of the page and click the top one to view the details.

![CloudWatch Log Group](./images/screenshot01.png)

This view now shows you the logs from the automation Lambda function. It is the place you can come to view the changes that were made to the resources as a result of changes to a Distribution Configuration. These logs show:
- The input event, this gives the raw detail of what update event triggered the function to run, useful in seeing what data the automation function was acting on.
- The regions that were added (or removed) from the StackSet so that the AMI Lookup Lambda functions were added or removed.
- The details of the accounts that were given (or revoked) permissions to call that AMI lookup Lambda function.

Now that you can see the Lambda function executed you can navigate to the [CloudFormation StackSets Console](https://console.aws.amazon.com/cloudformation/home#/stacksets) in the same account in the same region. You will see that we have a StackSet called *CustomResouceAmiLookup*. Take a look at the Stack instances and you will see there is one for each region that is defined in your Distribution Configurations. The Automation Lambda function did this for us.

The StackSet has deployed a Lambda function with a hardcoded name `CustomResourceAmiLookup` into each relevant region in this account (the account that contains the Image Builder).

## 5 - Review the CustomResourceAmiLookup Lambda function

If you wish to review the actual code for this Lambda function feel free to do so. It is written in Python and should be pretty easy to follow if you "speak Python". In this section we will describe exactly what it does so that it can be understood how it works. This function is written to use the standard format used by a CloudFormation Custom Resource Lambda function. As such it can take input parameters directly from a CloudFormation Template.
In this case the only parameter used is a `Tags` parameter. This parameter is a list of AWS tags. An example of how to format this is shown later.
The Lambda function uses the `DescribeImages` API to list the AMIs that have matching tags to those provided in the Parameters. The way it calls the `DescribeImages` API differs slightly depending on which template you deployed above:
  - `template-copied-amis.yaml` The Lambda function first assumes the IAM Role called `EC2ImageBuilderDistributionCrossAccountRole` [link](https://docs.aws.amazon.com/imagebuilder/latest/userguide/cross-account-dist.html) in the target account before it calls `DescribeImages`. This means that the AMIs the Lambda function returns are ones available in the target account.
  - `template-shared-amis.yaml` In this scenario an assume role is not needed instead it calls `DescribeImages` in its own account. This is because these AMIs are only shared and so cross account access is not required. This Lambda function also verifies the AMIs returned have been shared with the account to ensure no non-shared AMIs matching the tags will be returned.
Finally, once it has a list of AMIs the Lambda function finds the most recent one (using `CreationDate`) and returns just that AMI as a JSON object. This in turns gets returned to CloudFormation for us.

### 6 - Consuming the Custom Resource

In order to make use of this solution in a target account the user needs to add a new resource to their existing CloudFormation template. This resource is known as a Custom Resource and takes the format of:

```yaml
  Ami1:
    Type: Custom::AmiLookup
    Properties:
      ServiceToken: !Sub 'arn:aws:lambda:${AWS::Region}:111122223333:function:CustomResourceAmiLookup'
      Tags:
        - Key: CreatedBy
          Value: EC2 Image Builder
        - Key: Type
          Value: Hardened OS
```

The name of the resource (`Ami1`) can be chosen as anything the user requires just liek any other CloudFormation resource. The Custom Resource type (`AmiLookup`) can also be any allowed name. The value in the `ServiceToken` is important as this is what tells CloudFormation which Lambda function to invoke. The only part that should be changed from this example is that `111122223333` should be changed to match the **account id where the Image Builder is running**. Finally, the Tags property should be modified as required. With as many tags as required to match the AMI the user requires. The ability to retrieve an AMI based upon tags it has is the key to this entire solution. It is therefore expected that tags are being assign appropriately by Image Builder to make the AMI simple for the user to define here.

This resource type can be used as many times as required in the same template to retrieve different AMIs as required.

Once the resource is added to a template it then needs to be consumed by another resource, for example an EC2 instance:

```yaml
  MyEC2Instance:
    Type: AWS::EC2::Instance
    Properties: 
      ImageId: !GetAtt Ami1.ImageId
      KeyName: "testkey"
      BlockDeviceMappings: 
      - DeviceName: "/dev/sdm"
        Ebs: 
          VolumeType: "io1"
          Iops: "200"
          DeleteOnTermination: "false"
          VolumeSize: "20"
      - DeviceName: "/dev/sdk"
        NoDevice: {}
```

Here you can see the ImageId property has been set to get the ImageId from the attributes of the `Ami1` resource. It's that simple! In case more expansive use cases are required the object returned by the Custom Resource contains more than just the ImageId property its structure matches the full details of the AMI as defined [here](https://docs.aws.amazon.com/AWSEC2/latest/APIReference/API_Image.html).

So, for example if the user wanted to know the `CreationDate` of the AMI returned they would use `!GetAtt Ami1.CreationDate`
