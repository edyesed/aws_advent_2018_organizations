# AWS ADVENT 2018 - I GOTS TO GETS ORGANIZIZED 
![Travis Bickle Clapping](https://media.giphy.com/media/ulAzlijlNnKggJFDLl/giphy.gif "I GOTS TO GET ORGANIZIZED")

AWS Organizations are an amazing way to do a few existentially important things:
* Consolidate Payment for multiple AWS Accounts.
* Group AWS Accounts
* Provide policies for a Group of AWS Accounts.
* Control access to AWS Services, by Group or Individual Account
* Centralize CloudTrail Logging ( released at Re:Invent 2018 )

Sooner or later, every business grows to need policies. You may have some policies that should trickle down to your whole organization. You may wish to declare and enforce concepts like **S3 buckets should never be deleted**, **IAM Users should not be able to generate access keys**, or perhaps **CloudTrail logging should never be stopped**. Whether or not these concepts resonate with you, they are the types of ideas that Organizations can declare and enforce.

Some policies may end up being domain specific. PCI-DSS doesn't apply to non-financial business domains. Rights related to data retention may only apply in certain groups of countries, but not others. AWS Organizations can be leveraged to manifest these ideas.

AWS Organizations brings you technical controls for declaring, enforcing, and ( when paired with AWS Config ) reporting on compliance directives.

# Step Zero: Get your plan together. 
<<<IMG>>>
For many organizations having their ... AWS Organization ...  setup as a tree structure is a great option. 

## The Organizational Concept
At the root of the tree, you have a single account ( the same aws account from which we will begin working ). This single account runs _no code_. This account is **exclusively** for payment and policy management. 

In this article, we're going to:
* **Create a new account** that will be the root of our organization
* Create a SCP that declares **cloudhsm should not be used**
* Create a SCP that declares **cloudtrail:StopLogging cannot be called**
* Attach those SCPs into our new organization.

## The Policy Layout
As you move out from the root and to the first layer of subordinate accounts ( children of the root account ), one policy may apply. Say this policy is "You can run anything except HSM". 

One of those child accounts may have its own children who process credit card payments and are subject to PCI-DSS. These grandchildren accounts may be restricted to using an explicit whitelist of services. They may be required to use 2FA when logging in. Maybe they can't use S3, because srsbzns can't happen via S3? 

## But what about existing accounts
You can invite accounts into your organziation via one of two ways:
1. You invite by AWS Account ID
2. You invite by email, which uses the AWS Console's root user login's email address.

I'm going to assume you've already got at least one AWS Account, if not, create one and invite it via email. If you do already have an account, find its account id. 

## The Organizational Layout
The image below illustrates the organizational structure that we're going to be creating as a part of this writeup. [Service Control Policies](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scp.html) ( called SCPs hereafter ) will be used to enforce our policies. If you follow the link to the SCPs page, you'll notice a few important caveats to how SCPs work. 
In short 
1. SCPs only **Deny** access 
2. SCPs don't apply to Service Linked Accounts
3. SCPs only apply to Principles **inside your organization**, 
4. if you disable the SCP policy type in a root you can expect to spend the next several days re-enabling it with much tender loving care. 
<<<images/Amazing_Organization.png>>>


# Step One: Prep your soon-to-be-root account
AWS Requires you to verify your email address before you can begin ~~summoning~~ creating subordinate accounts. Choose an account that will be your new root account, and verify the email address on it by logging into the AWS Console, and visiting [https://console.aws.amazon.com/organizations/home](https://console.aws.amazon.com/organizations/home)

## Build a fresh account to be the new root account
If you have an existing AWS Account that is not itself the root of an organization, you may want to create a new account for this purpose. This writeup's screencaps will continue with a fresh account that's the new designated root

<<FreshAccount_IMG1>>
<<FreshAccount_IMG2>>
<<FreshAccount_IMG3>>

it should look like this : <<INSERT PIC OF ORGANIZATION Console>>

## Create an admin user 
We will need to use the `aws` CLI just a bit, because there isn't super amazing cloudformation support to _generate organizational children_. 

This user can be created via cloudformation, however, and that's doable via the console.

1. As your root user, naviagate to `Services` -> `CloudFormation`
1. `Create Stack`
1. In the **Choose a Template** section, put the radio button onto `Specify an Amazon S3 URL` 
1. In that URL place [https://s3-us-west-2.amazonaws.com/awsadvent-2018-organizations/StepOneCFNs/phase_0_user_and_accesskey.yml](https://s3-us-west-2.amazonaws.com/awsadvent-2018-organizations/StepOneCFNs/phase_0_user_and_accesskey.yml)
1. `Next`
1. **Stack Name** `AdventAdmin`
1. `Next`
1. Nothing needed here on the Options screen
1. `Next`
1. Check the box acknowledging that AWS CloudFormation might create IAM resources with custom names.
1. `Create`
1. Wait for the stack to reach a `Create Completed` state <<IMG OF CFn1 COMPLETE>>


# Step Two: Create Some SCPs
The Organizations API has great support via the CLI and the SDKs, but it's not really present in CloudFormation. We're going to use the `aws` cli to interact with the AWS Organizations APIs.  If you don't already have it, [here is the guide to installing the aws cli](https://docs.aws.amazon.com/cli/latest/userguide/installing.html). 

## Get your credentials together
We created an IAM User in Step One that has full Administrator privilege. For this guide, I'm going to assume that's the user that you'll be using. 

To get the credentials to use the cli as this user
1. `Services` -> `CloudFormation`
1. `Stacks`
1. `AdventAdmin`
1. `Outputs`
   You can cut and paste those two values into your CLI to build something like

    ```shell
    (aws_advent_2018_organizations) bash-3.2$ AWS_SECRET_ACCESS_KEY=LOL_NO_I_CHANGED_THESE \
    AWS_ACCESS_KEY_ID=THIS_ONE_TOO \
    aws sts get-caller-identity
    ```
    ```json
   {
    "UserId": "AIDAJSVCKK2GOXVWBQ2TW",
    "Account": "411181159725",
    "Arn": "arn:aws:iam::411181159725:user/AdventAdminUser"
    }
   (aws_advent_2018_organizations) bash-3.2$
   ```

## Make your first SCP
We're going to first make a Service Control Policy for our entire organzation that states **"No one anywhere can run [CloudHSM](https://aws.amazon.com/cloudhsm/)"**. CloudHSM was chosen for this example because it tends to not be used, and it's relatively expensive. There's nothing wrong with CloudHSM! **If your business needs CloudHSM, these SCPs should be considered for demonstration purposes only.**


1. Check to be sure that your organization is functional:

    ```shell
    (aws_advent_2018_organizations) bash-3.2$ AWS_SECRET_ACCESS_KEY=LOL_NO_I_CHANGED_THESE \
    AWS_ACCESS_KEY_ID=THIS_ONE_TOO \
    aws organizations describe-organization
    ```
    ```json
    {
        "Organization": {
            "Id": "o-8a8xjmie9h",
            "Arn": "arn:aws:organizations::411181159725:organization/o-8a8xjmie9h",
            "FeatureSet": "ALL",
            "MasterAccountArn": "arn:aws:organizations::411181159725:account/o-8a8xjmie9h/411181159725",
            "MasterAccountId": "411181159725",
            "MasterAccountEmail": "edyesed+rootawsaccount@gmail.com",
            "AvailablePolicyTypes": [
                {
                    "Type": "SERVICE_CONTROL_POLICY",
                    "Status": "ENABLED"
                }
            ]
        }
    }
    ```

2. Check your existing SCPs. Amazon created one for you when you built your organization. It says **"All Accounts in this organization can use all services"**. Check it with this command:
    ```shell
    (aws_advent_2018_organizations) bash-3.2$ AWS_SECRET_ACCESS_KEY=LOL_NO_I_CHANGED_THESE \
    AWS_ACCESS_KEY_ID=THIS_ONE_TOO \
    aws organizations list-policies --filter SERVICE_CONTROL_POLICY
    ```
    ```json
    {
        "Policies": [
            {
                "Id": "p-FullAWSAccess",
                "Arn": "arn:aws:organizations::aws:policy/service_control_policy/p-FullAWSAccess",
                "Name": "FullAWSAccess",
                "Description": "Allows access to every operation",
                "Type": "SERVICE_CONTROL_POLICY",
                "AwsManaged": true
            }
        ]
    }
    ```

1. Now we'll create our new SCP, which will state **Deny all use of cloudhsm**. Notice how the SCP language is almost an IAM policy? 
    ```shell
    (aws_advent_2018_organizations) bash-3.2$ AWS_SECRET_ACCESS_KEY=LOL_NO_I_CHANGED_THESE \
    AWS_ACCESS_KEY_ID=THIS_ONE_TOO \
    aws organizations create-policy \
    --description "Disables cloudhsm" \
    --name "Deny cloudhsm:*"  \
    --type SERVICE_CONTROL_POLICY  \
    --content '{"Version": "2012-10-17", "Statement": [{"Effect": "Deny", "Action": ["cloudhsm:*","cloudhsmv2:*"], "Resource":["*"]}]}'
    ```

1. List the policies again, to notice that there are two
    ```shell
    (aws_advent_2018_organizations) bash-3.2$ AWS_SECRET_ACCESS_KEY=LOL_NO_I_CHANGED_THESE \
    AWS_ACCESS_KEY_ID=THIS_ONE_TOO \
    aws organizations list-policies --filter SERVICE_CONTROL_POLICY
    ```
    ```json
    {    "Policies": [
            {
                "Id": "p-FullAWSAccess",
                "Arn": "arn:aws:organizations::aws:policy/service_control_policy/p-FullAWSAccess",
                "Name": "FullAWSAccess",
                "Description": "Allows access to every operation",
                "Type": "SERVICE_CONTROL_POLICY",
                "AwsManaged": true
            },
            {
                "Id": "p-2pnf8e6y",
                "Arn": "arn:aws:organizations::411181159725:policy/o-8a8xjmie9h/service_control_policy/p-94uf5sa5",
                "Name": "Deny cloudhsm:*",
                "Description": "Disables cloudhsm",
                "Type": "SERVICE_CONTROL_POLICY",
                "AwsManaged": false
            }
        ]
    }
    ```

1. Let's make another SCP, which will state **CloudTrail cannot be disabled**. 
    ```shell
    (aws_advent_2018_organizations) bash-3.2$ AWS_SECRET_ACCESS_KEY=LOL_NO_I_CHANGED_THESE \
    AWS_ACCESS_KEY_ID=THIS_ONE_TOO \
    aws organizations create-policy \
    --description "Keep CloudTrail enabled" \
    --name "Keep CloudTrail enabled"  \
    --type SERVICE_CONTROL_POLICY  \
    --content '{"Version": "2012-10-17", "Statement": [{"Effect": "Deny", "Action": "cloudtrail:StopLogging", "Resource":["*"]}]}'
    ```

1. Let's make another SCP, which will state **AWS Config Rules Cannot Be Disabled**. 
    ```shell
    (aws_advent_2018_organizations) bash-3.2$ AWS_SECRET_ACCESS_KEY=LOL_NO_I_CHANGED_THESE \
    AWS_ACCESS_KEY_ID=THIS_ONE_TOO \
    aws organizations create-policy \
    --description "Keep Config enabled" \
    --name "Keep Config enabled"  \
    --type SERVICE_CONTROL_POLICY  \
    --content '{"Version": "2012-10-17", "Statement": [{"Effect": "Deny", "Action": ["config:DeleteConfigRule","config:DeleteConfigurationRecorder","config:DeleteDeliveryChannel","config:StopConfigurationRecorder"], "Resource":["*"]}]}'
    ```

## Enable SCPs for your organization and attach them
At this point, we have **An Organization** and **some SCPs**, but they aren't attached to one another. 

Our organization does not yet have any structure, and it is not in a state where the SCPs that we created can be attached anywhere.

SCPs have to be explicitly enabled for your Organization. Let us go ahead and do that. 
1. First, we're going to go ahead run a command that will be disabled via SCP later.
    ```shell
    (aws_advent_2018_organizations) bash-3.2$ AWS_SECRET_ACCESS_KEY=LOL_NO_I_CHANGED_THESE \
    AWS_ACCESS_KEY_ID=THIS_ONE_TOO \
    aws cloudhsmv2 describe-clusters
    ```
    ```json
    {
        "Clusters": []
    }
    ```
1. Now, we need to gather the organizational Root id in order to enable SCPs
    ```shell
    (aws_advent_2018_organizations) bash-3.2$ AWS_SECRET_ACCESS_KEY=LOL_NO_I_CHANGED_THESE \
    AWS_ACCESS_KEY_ID=THIS_ONE_TOO \
    aws organizations list-roots
    ```
    ```json
    {
        "Roots": [
            {
                "Id": "r-8a0p",
                "Arn": "arn:aws:organizations::411181159725:root/o-8a8xjmie9h/r-8a0p",
                "Name": "Root",
                "PolicyTypes": []
            }
        ]
    }
    ```
2. We will now enable SCPs in our Organization's Root. Take the Id in the output above
    ```shell
    (aws_advent_2018_organizations) bash-3.2$ AWS_SECRET_ACCESS_KEY=LOL_NO_I_CHANGED_THESE \
    AWS_ACCESS_KEY_ID=THIS_ONE_TOO \
    aws organizations enable-policy-type --root-id r-8a0p \
      --policy-type SERVICE_CONTROL_POLICY
    ```
    ```json
    {
        "Root": {
            "Id": "r-8a0p",
            "Arn": "arn:aws:organizations::411181159725:root/o-8a8xjmie9h/r-8a0p",
            "Name": "Root",
            "PolicyTypes": []
        }
    }
    ```
    **DO NOT PANIC BECAUSE POLICYTYPES IS EMPTY**
2. List the roots of the organization again, and (hopefully) notice that SCPs are enabled
    ```shell
    (aws_advent_2018_organizations) bash-3.2$ AWS_SECRET_ACCESS_KEY=LOL_NO_I_CHANGED_THESE \
    AWS_ACCESS_KEY_ID=THIS_ONE_TOO \
    aws organizations list-roots
    ```
    ```json
    {
        "Roots": [
            {
                "Id": "r-8a0p",
                "Arn": "arn:aws:organizations::411181159725:root/o-8a8xjmie9h/r-8a0p",
                "Name": "Root",
                "PolicyTypes": [
                    {
                        "Type": "SERVICE_CONTROL_POLICY",
                        "Status": "ENABLED"
                    }
                ]
            }
        ]
    }
    ```
2. List out the SCP Policies. You'll need these Ids in the coming commands
    ```shell
    (aws_advent_2018_organizations) bash-3.2$ AWS_SECRET_ACCESS_KEY=LOL_NO_I_CHANGED_THESE \
    AWS_ACCESS_KEY_ID=THIS_ONE_TOO \
    aws organizations list-policies \
      --filter SERVICE_CONTROL_POLICY
    ```
    ```json
    {
        "Policies": [
            {
                "Id": "p-FullAWSAccess",
                "Arn": "arn:aws:organizations::aws:policy/service_control_policy/p-FullAWSAccess",
                "Name": "FullAWSAccess",
                "Description": "Allows access to every operation",
                "Type": "SERVICE_CONTROL_POLICY",
                "AwsManaged": true
            },
            {
                "Id": "p-2v72m3ps",
                "Arn": "arn:aws:organizations::411181159725:policy/o-8a8xjmie9h/service_control_policy/p-2v72m3ps",
                "Name": "Keep Config enabled",
                "Description": "Keep Config enabled",
                "Type": "SERVICE_CONTROL_POLICY",
                "AwsManaged": false
            },
            {
                "Id": "p-2pnf8e6y",
                "Arn": "arn:aws:organizations::411181159725:policy/o-8a8xjmie9h/service_control_policy/p-2pnf8e6y",
                "Name": "Deny cloudhsm:*",
                "Description": "Disables cloudhsm",
                "Type": "SERVICE_CONTROL_POLICY",
                "AwsManaged": false
            },
            {
                "Id": "p-7i3y6l3k",
                "Arn": "arn:aws:organizations::411181159725:policy/o-8a8xjmie9h/service_control_policy/p-7i3y6l3k",
                "Name": "Keep CloudTrail enabled",
                "Description": "Keep CloudTrail enabled",
                "Type": "SERVICE_CONTROL_POLICY",
                "AwsManaged": false
            }
        ]
    }
    ```
2. Attach the **Deny cloudhsm:\*** SCP to the root. Doing this will trickle through the whole organization.
    ```shell
    (aws_advent_2018_organizations) bash-3.2$ AWS_SECRET_ACCESS_KEY=LOL_NO_I_CHANGED_THESE \
    AWS_ACCESS_KEY_ID=THIS_ONE_TOO \
    aws organizations attach-policy \
      --policy-id p-2pnf8e6y \
      --target-id r-8a0p
    ```
    ```json
    ```
2. Attach the **Keep CloudTrail Enabled** SCP to the root. Doing this will trickle through the whole organization.
    ```shell
    (aws_advent_2018_organizations) bash-3.2$ AWS_SECRET_ACCESS_KEY=LOL_NO_I_CHANGED_THESE \
    AWS_ACCESS_KEY_ID=THIS_ONE_TOO \
    aws organizations attach-policy \
      --policy-id p-7i3y6l3k \
      --target-id r-8a0p
    ```
    ```json
    ```
2. Attach the **Keep Config enabled** SCP to the root. Doing this will trickle through the whole organization.
    ```shell
    (aws_advent_2018_organizations) bash-3.2$ AWS_SECRET_ACCESS_KEY=LOL_NO_I_CHANGED_THESE \
    AWS_ACCESS_KEY_ID=THIS_ONE_TOO \
    aws organizations attach-policy \
      --policy-id p-2v72m3ps \
      --target-id r-8a0p
    ```
    ```json
    ```
2. Show which policies are actually attached to our root object.
    ```shell
    (aws_advent_2018_organizations) bash-3.2$ AWS_SECRET_ACCESS_KEY=LOL_NO_I_CHANGED_THESE \
    AWS_ACCESS_KEY_ID=THIS_ONE_TOO \
    aws organizations list-policies-for-target \
      --filter SERVICE_CONTROL_POLICY \
      --target-id r-8a0p
    ```
    ```json
    {
        "Policies": [
            {
                "Id": "p-FullAWSAccess",
                "Arn": "arn:aws:organizations::aws:policy/service_control_policy/p-FullAWSAccess",
                "Name": "FullAWSAccess",
                "Description": "Allows access to every operation",
                "Type": "SERVICE_CONTROL_POLICY",
                "AwsManaged": true
            },
            {
                "Id": "p-7i3y6l3k",
                "Arn": "arn:aws:organizations::411181159725:policy/o-8a8xjmie9h/service_control_policy/p-7i3y6l3k",
                "Name": "Keep CloudTrail enabled",
                "Description": "Keep CloudTrail enabled",
                "Type": "SERVICE_CONTROL_POLICY",
                "AwsManaged": false
            },
            {
                "Id": "p-2v72m3ps",
                "Arn": "arn:aws:organizations::411181159725:policy/o-8a8xjmie9h/service_control_policy/p-2v72m3ps",
                "Name": "Keep Config enabled",
                "Description": "Keep Config enabled",
                "Type": "SERVICE_CONTROL_POLICY",
                "AwsManaged": false
            },
            {
                "Id": "p-2pnf8e6y",
                "Arn": "arn:aws:organizations::411181159725:policy/o-8a8xjmie9h/service_control_policy/p-2pnf8e6y",
                "Name": "Deny cloudhsm:*",
                "Description": "Disables cloudhsm",
                "Type": "SERVICE_CONTROL_POLICY",
                "AwsManaged": false
            }
        ] 
    }
    ```
1. Ok, **Now let's see what we've disabled***
    ```shell
    (aws_advent_2018_organizations) bash-3.2$ AWS_SECRET_ACCESS_KEY=LOL_NO_I_CHANGED_THESE \
    AWS_ACCESS_KEY_ID=THIS_ONE_TOO \
    aws cloudhsmv2 describe-clusters
    ```
    ```json
    {
        "Clusters": []
    }
    ```
    BUT, BUT, we just disabled that!! WHAT?!

    **SCPs don't impact the root of your organization**. The Aristocrats!
    ![Gilbert Gotfried delivering the punchline "The Aristcats!"](https://media.giphy.com/media/69lVlY63AMhoDsvhhu/giphy.gif "The Aristocats!")

    Presumably, this means that if you have an admin/root user in the root of your organization, you can recover. maybe.  with the help of support.  ( don't try this for funsies, folks! )

# Step Three: Build out an Organization CloudTrail
Now that we've laid the groundwork, let's build out this amazing organization that we're so excited to try out! 

## Make a S3 bucket to drop the CloudTrail logs into
We're now ready to create a S3 bucket into which we'll stash our CloudTrail Logs for our entire organization, automatically, as the org grows or shrinks. 

Pretty cool, right? 
Before we can create the S3 bucket that we're gonna drop our cloudtrails into, we need to get the organization's OrgId

1.  Use the CLI to grab your OrgId
    ```shell
    (aws_advent_2018_organizations) bash-3.2$ AWS_SECRET_ACCESS_KEY=LOL_NO_I_CHANGED_THESE \
    AWS_ACCESS_KEY_ID=THIS_ONE_TOO \
    aws organizations describe-organization  | grep '"Id"'
    ```
    ```json
        "Id": "o-8a8xjmie9h",
    ```
Now move over to add the CloudFormation Stack that is going to build out the S3 bucket
with the right bucket policy for our org to log cloudtrail data into it. 

1. As your root user, naviagate to `Services` -> `CloudFormation`
1. `Create Stack`
1. In the **Choose a Template** section, put the radio button onto `Specify an Amazon S3 URL` 
1. In that URL place [https://s3-us-west-2.amazonaws.com/awsadvent-2018-organizations/StepThreeCFNs/phase_3_s3_bucket.yml](https://s3-us-west-2.amazonaws.com/awsadvent-2018-organizations/StepThreeCFNs/phase_3_s3_bucket.yml)
1. `Next`
1. **Stack Name** `CloudTrailS3Bucket`
1. **OrgId** `your-org-id-from-the-cli-command-above`
1. `Next`
1. Nothing needed here on the Options screen
1. `Next`
1. Check the box acknowledging that AWS CloudFormation might create IAM resources with custom names.
1. `Create`
1. Wait for the stack to reach a `Create Completed` state <<IMG OF CFn1 COMPLETE>>


## Create a **Organizational** CloudTrail
Normally, I'd have dropped the CloudTrail creation into CloudFormation, because I'm not a monster... **BUT...** If you aren't already aware, you can consider this my heads up to you that new features frequently get CLI/API support well before they manifest in CloudFormation.

CloudFormation does not yet support organizational CloudTrails.  TO THE CLI!

1.  Gather the S3 bucket name that we created earlier.
    ```shell
    (aws_advent_2018_organizations) bash-3.2$ AWS_SECRET_ACCESS_KEY=LOL_NO_I_CHANGED_THESE \
    AWS_ACCESS_KEY_ID=THIS_ONE_TOO \
    aws cloudformation describe-stack-resources --stack-name CloudTrailS3Bucket | \
      grep Physical | \
      grep s3bucket
    ```
    ```json
                "PhysicalResourceId": "cloudtrails3bucket-s3bucket-78is3f3s2eqj",
    ```
2. Now we have to enable all organizational features
    ```shell
    (aws_advent_2018_organizations) bash-3.2$ AWS_SECRET_ACCESS_KEY=LOL_NO_I_CHANGED_THESE \
    AWS_ACCESS_KEY_ID=THIS_ONE_TOO \
    aws organizations enable-all-features
    ```
    ```json
    An error occurred (HandshakeConstraintViolationException) when calling the EnableAllFeatures operation: All features are already enabled on this organization.
    ```
    Even though this is an error, it's the one that we want. Features are enabled 👍👍

2. Now we have to enable service access for cloudtrail. SCPs don't impact service access.

    ```shell
    (aws_advent_2018_organizations) bash-3.2$ AWS_SECRET_ACCESS_KEY=LOL_NO_I_CHANGED_THESE \
    AWS_ACCESS_KEY_ID=THIS_ONE_TOO \
    aws organizations enable-aws-service-access \
      --service-principal cloudtrail.amazonaws.com
    ```
    ```json
    ```

2. Finally, we create the actual trail

    ```shell
    (aws_advent_2018_organizations) bash-3.2$ AWS_SECRET_ACCESS_KEY=LOL_NO_I_CHANGED_THESE \
    AWS_ACCESS_KEY_ID=THIS_ONE_TOO \
    aws cloudtrail create-trail \
      --name org-trail \
      --s3-bucket-name cloudtrails3bucket-s3bucket-78is3f3s2eqj \
      --is-organization-trail \
      --is-multi-region-trail
    ```
    ```json
    {
        "Name": "org-trail",
        "S3BucketName": "cloudtrails3bucket-s3bucket-78is3f3s2eqj",
        "IncludeGlobalServiceEvents": true,
        "IsMultiRegionTrail": true,
        "TrailARN": "arn:aws:cloudtrail:us-west-2:411181159725:trail/org-trail",
        "LogFileValidationEnabled": false,
        "IsOrganizationTrail": true
    }
    ```

Ok, you've done a lot so far. And you're going to be happy that you laid out all this prep work once you have to start answering questions like "Who in X account built out as many EC2s as their account would allow?" 

Or, "Did Frank from Accounting actually delete the RDS Database in their AWS Account?"

# Step Four: Finally Build Out Some Organization
Let's get to the purpose that you're here.. Building out some Organizations!
<<IMG ORG LAYOUT>>

## Make An Organizational Unit ( OU ) for developers
You work in a progressive organization that wants individual developers to have their own AWS Accounts, huzzah! 

We're going to build an OU to stuff our developer accounts in, and then we'll invite some developers into our org

1. Gather the existing organization's root. Since we don't yet have any OUs, all things are rooted from the root. 
    ```shell
    (aws_advent_2018_organizations) bash-3.2$ AWS_SECRET_ACCESS_KEY=LOL_NO_I_CHANGED_THESE \
    AWS_ACCESS_KEY_ID=THIS_ONE_TOO \
    aws organizations list-roots | grep '"Id"'
    ```
    ```json
        "Id": "r-8a0p",
    ```
1. Now let's create a subordinate OU
    ```shell
    (aws_advent_2018_organizations) bash-3.2$ AWS_SECRET_ACCESS_KEY=LOL_NO_I_CHANGED_THESE \
    AWS_ACCESS_KEY_ID=THIS_ONE_TOO \
    aws organizations \
      create-organizational-unit \
      --parent-id r-8a0p \
      --name "Developer Accounts"
    ```
    ```json
    {
        "OrganizationalUnit": {
            "Id": "ou-8a0p-ejmtbyhz",
            "Arn": "arn:aws:organizations::411181159725:ou/o-8a8xjmie9h/ou-8a0p-ejmtbyhz",
            "Name": "Developer Accounts"
        }
    }
    ```

1. Invite an email address
    ```shell
    (aws_advent_2018_organizations) bash-3.2$ AWS_SECRET_ACCESS_KEY=LOL_NO_I_CHANGED_THESE \
    AWS_ACCESS_KEY_ID=THIS_ONE_TOO \
    aws organizations \
      invite-account-to-organization \
      --target Type=EMAIL,Id=youremail+devaccount1@whatever.com
    ```
    ```json
    {
        "Handshake": {
            ...
        }
    }
    ```
1. Or invite an account id
    ```shell
    (aws_advent_2018_organizations) bash-3.2$ AWS_SECRET_ACCESS_KEY=LOL_NO_I_CHANGED_THESE \
    AWS_ACCESS_KEY_ID=THIS_ONE_TOO \
    aws organizations \
      invite-account-to-organization \
      --target Type=ACCOUNT,Id=488887740717
    ```
    ```json
    {
        "Handshake": {
            ...
        }
    }
 
1. Now go sign in as that account that you just invited. Accept the invitation.
<<SCREENCAP ORGANIZATIONS AS SUBORDINATE>>


1. At this point, our newly invited account needs to be moved to our developers OU. When the account joins our organization, it's parented by the *root* of the org. 
    ```shell
    (aws_advent_2018_organizations) bash-3.2$ AWS_SECRET_ACCESS_KEY=LOL_NO_I_CHANGED_THESE \
    AWS_ACCESS_KEY_ID=THIS_ONE_TOO \
    aws organizations move-account \
      --account-id 488887740717 \
      --source-parent-id r-8a0p \
      --destination-parent-id ou-8a0p-ejmtbyhz
    ```
    ```json
    ```
Deep Breath, We've done it

1. Check your permissions. I have some credentials stashed away in ~/.aws/credentials for this account (488887740717). When I try and run `aws cloudhsmv2 describe-clusters`, I will now expect to get a AccessDeniedExeption.

    ```shell
    (aws_advent_2018_organizations) bash-3.2$ AWS_PROFILE=edyesed_ebooks \
    aws cloudhsmv2 describe-clusters
    ```
    ```json
    An error occurred (AccessDeniedException) when calling the DescribeClusters operation: User: arn:aws:iam::488887740717:user/ed_temp is not authorized to perform: cloudhsm:DescribeClusters with an explicit deny
    ```
