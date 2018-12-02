# aws_advent_2018_organizations
Code supporting the AWS Organizations article from awsadvent of 2018

# AWS ADVENT 2018 - I GOTS TO GETS ORGANIZIZED 

AWS Organizations are an amazing way to do a few existentially important things:
* Consolidate Payment for multiple AWS Accounts.
* Group AWS Accounts
* Provide policies for a Group of AWS Accounts.
* Control access to AWS Services, by Group or Individual Account
* Centralize CloudTrail Logging ( released at Re:Invent 2018 )

Sooner or later, every business grows to need policies. You may have some policies that should trickle down to your whole organization. You may wish to declare and enforce that **S3 buckets should never be public**, and **every S3 bucket should be using server side encryption**. 

jOther policies may end up being domain specific. PCI-DSS doesn't apply to non-financial business domains. Rights related to data retention may only apply in certain groups of countries, but not others. 

AWS Organizations brings you technical controls for declaring, enforcing, and ( when paired with AWS Config ) reporting on compliance directives.


# Step Zero: Get your plan together. 
<<<IMG>>>
For many organizations having their ... AWS Organization ...  setup as a tree structure is a great option. 

## The Organizational Concept
At the root of the tree, you have a single account ( the same aws account from which we will begin working ). This single account runs _no code_. This account is **exclusively** for payment and policy management. 

## The Policy Layout
As you move out from the root and to the first layer of subordinate accounts ( children of the root account ), one policy may apply. Say this policy is "You can run anything except HSM". 

One of those child accounts may have its own children who process credit card payments and are subject to PCI-DSS. These grandchildren accounts may be restricted to using an explicit whitelist of services. They may be required to use 2FA when logging in. Maybe they can't use S3, because srsbzns can't happen via S3? 

## The Policy Layout
At my day job, we've been using AWS Payer accounts long before before there were Organizations, and let me tell you... (a) organizations is a big improvement (b) it is totally possible to bring existing accounts into Organizations. However, this write-up isn't going to get into detail on the mechanics of that.  

If you follow this guide, you'll be able to bring your existing accounts into Organizations. You'll have the necessary skills to integrate existing accounts into Organizations, logically group them, and apply policies to those accounts. 

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


# Step Two: Create a subordinate account
The Organizations API has great support via the CLI and the SDKs, but it's not really present in CloudFormation. We're going to use the `aws` cli to interact with the AWS Organizations APIs.  If you don't already have it, [here is the guide to installing the aws cli](https://docs.aws.amazon.com/cli/latest/userguide/installing.html). 

We created an IAM User in Step One that has full Administrator privilege. For this guide, I'm going to assume that's the user that you'll be using. 

To get the credentials to log in as this user
1. `Console`
1. `CloudFormation`
1. `Stacks`
1. `AdventAdmin`
1. `Outputs`
   You can cut and paste those two values into your CLI to build something like

   ```shell
   (aws_advent_2018_organizations) bash-3.2$ AWS_SECRET_ACCESS_KEY=LOL_NO_I_CHANGED_THESE \
   AWS_ACCESS_KEY_ID=THIS_ONE_TOO \
   aws sts get-caller-identity
   {
    "UserId": "AIDAJSVCKK2GOXVWBQ2TW",
    "Account": "411181159725",
    "Arn": "arn:aws:iam::411181159725:user/AdventAdminUser"
    }
   (aws_advent_2018_organizations) bash-3.2$
   ```

You can 
