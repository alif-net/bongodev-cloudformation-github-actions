# IaC with AWS CloudFormation and GitHub Actions

## Part 1: CloudFormation lab (console)
I uploaded one template (cloudformation.yaml) twice in the CloudFormation console
and created two independent stacks in eu-central-1 (Frankfurt):
bongodev-ubuntu-nginx-dev (Environment=dev) and bongodev-ubuntu-nginx-test
(Environment=test). Each stack created its own VPC, public subnet, internet
gateway, route table, security group (HTTP 80 only) and Ubuntu 24.04 t3.micro
instance. EC2 User Data installed Nginx and wrote the page.

| Check | Development | Testing |
|---|---|---|
| EC2 instance ID | i-___ | i-___ |
| VPC ID | vpc-___ | vpc-___ |
| Website shows | Environment: dev | Environment: test |
| Stack status | CREATE_COMPLETE | CREATE_COMPLETE |

Both stacks were deleted afterwards (screenshot L05).

## Part 2: Homework (GitHub Actions + OIDC)
Repository: https://github.com/alif-net/bongodev-cloudformation-github-actions
Second-push commit: https://github.com/alif-net/bongodev-cloudformation-github-actions/commit/___
Region: eu-central-1. (The README table pairs us-west-2 with N. Virginia;
us-west-2 is Oregon and N. Virginia is us-east-1, so I used my usual region.)

- IAM OIDC provider token.actions.githubusercontent.com, audience sts.amazonaws.com
- Role bongodev-github-iac-deploy-role with policy bongodev-github-iac-lab-policy
- Trust policy restricted to this repository's main branch, using the new
  immutable subject format (owner ID and repository ID)
- Repository variables AWS_ROLE_ARN and AWS_REGION; no access keys stored anywhere
- First push: workflow validated the template and created stack
  bongodev-github-iac-dev (CREATE_COMPLETE); website shows "Hello from bongoDev!"
- Second push: changed only the Course tag to bongoDev-DevOps-Homework-V2;
  the stack went to UPDATE_COMPLETE and the instance ID stayed the same (i-___)
- Cleanup: workflow disabled, stack deleted (DELETE_COMPLETE)

## GitHub Actions vs CloudFormation, and why OIDC
GitHub Actions is the automation engine: on a push to main it checks out the
repository, gets temporary AWS credentials and runs the AWS CLI commands.
CloudFormation is the AWS service that reads the template and creates, updates
or deletes the actual resources so they match it. OIDC lets AWS trust a signed,
short-lived token that GitHub issues for this exact repository and branch, and
exchange it for credentials that expire in about an hour. There are no
long-lived access keys that could leak from the repository, logs or a laptop,
and nothing has to be rotated.

## Answers: CloudFormation lab
1. IaC replaces manual clicking with a version-controlled definition, so
   environments are repeatable, reviewable, documented and easy to delete.
2. Parameters are inputs (Environment, AMI ID), Resources are what gets created
   (VPC, subnet, security group, EC2), Outputs are values shown after creation
   (WebsiteURL, InstanceId, PublicIPAddress, VpcId).
3. A stack is one deployed instance of a template: all its resources are
   created, updated and deleted together as one unit.
4. Each stack creates its own resources. Same template, two stacks, two
   independent sets of resources, so two instance IDs.
5. Ubuntu is Debian-based and uses apt/apt-get. dnf is the package manager of
   Fedora-based systems such as Amazon Linux 2023.
6. Canonical is the company behind Ubuntu. It publishes the current official
   Ubuntu AMI ID in a public SSM parameter in every region, so the template
   needs no hard-coded, region-specific AMI ID.
7. No. CREATE_COMPLETE means AWS created the resources. User Data then still
   runs apt-get and starts Nginx, which can take a few more minutes.
8. Both stacks (which removes the EC2 instances, EBS volumes, public IPs, VPCs,
   subnets, gateways and security groups), because they cost money and an open
   HTTP server should not run without a reason.

## Answers: Homework
1. GitHub Actions runs the deployment steps; CloudFormation provisions and
   reconciles the AWS resources to the template.
2. The push triggers the workflow, the workflow assumes an IAM role through
   OIDC and calls CloudFormation with the AWS CLI. Nobody needs the console.
3. OIDC lets AWS verify a GitHub-signed token for a specific repo and branch
   and return temporary credentials. Committed access keys are long-lived,
   can leak and work for anyone who has them.
4. aws cloudformation deploy creates a change set against the existing stack
   with the same name. A new stack is only created when no stack has that name.
5. No. CloudFormation finishes when the EC2 instance exists; Nginx is installed
   later by User Data on first boot.
6. User Data runs only on first boot, so changing the HTML would not change the
   running site. A tag change is safe, visible right away and keeps the instance.
7. Delete the stack, confirm the instance and network resources are gone,
   disable the workflow so a push cannot recreate them, and later remove the
   IAM role and policy when they are no longer needed.
