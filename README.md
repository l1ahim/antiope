# Antiope
AWS Inventory &amp; Compliance Framework


# Purpose
Antiope (PRONO An-Tie-Oh-Pee) is intended to be an open sourced framework for managing resources across hundreds of AWS Accounts. From a trusted Security Account, Antiope will leverage Cross Account Assume Roles to gather up resource data and store them in an inventory bucket. This bucket can then be index by ELK or your SEIM of choice to provide easy searching of resources across hundreds of AWS accounts.

## What it current collects
Antiope is given a list of AWS Organizational parent accounts, and will inventory all of the AWS accounts under those parents. For each of the parent & child accounts it will then gather:

1. S3 Buckets, and associated attributes of the bucket
1. VPCs, and the number of EC2 Instances in each VPC
1. Route53 Hosted Zones
1. Route53 Registered Domains
1. EC2 Instances
1. EC2 Security Groups
1. IAM Users
1. IAM Roles (and the AWS accounts that are trusted by the roles)
1. All Elastic Network Interfaces (ENIs) in each VPC, and any PublicIP addresses associated to the ENIs

All resources are dropped as individual json files into the S3 Bucket of your choosing under `/Resources/<type>-<resource_id>.json`

## What you can do with what it collects
Right now, the primary function of the collection is to solve the needle-in-the-haystack problem. My aggregating all the resources across all accounts & regions into a single place finding resources like IP addresses becomes much easier. Antiope is also starting to track where inter-account trust is occurring and creating a record of accounts outside your organization that are trusted by one or more accounts in your organization.

Finally, the elastic search cluster is being used as a data repository for threat hunting scripts to find things like open Elastic Search or de-referenced CloudFront origins. Those hunting scripts can be found in antiope/search-cluster/scripts/hunt-*


# Modular Structure
Antiope is somewhat modular in that there are currently three separate CloudFormation stacks that comprise the core functionality. These are intended to be used if needed and ignore if your enterprise has a better solution. The current three modules are:

## Cognito Stack
This stack creates the UserPool and IDPool for Cognito authentication to the Kibana endpoint in the Search Cluster and as a way to authenticate access to the reports stored in the S3 bucket. It's a fairly bare-bones Cognito install and can be extended to leverage SAML federation with your enterprise identity store.

## AWS Inventory Stack
This stack creates the Lambda, DynamoDB, StepFunctions, and associated glue required to collect the resource data across all of the accounts under all of your organizational parents.

## Search Cluster Stack
This stack creates the (optional) Amazon Elastic Search cluster for searching the resource objects gathered by the inventory stack. This stack also creates the pipeline for SQS & Lambda to detect when new objects are added to the bucket and make sure those objects are indexed.

## GCP Inventory Stack
Currently a work-in-progress, this stack replicates the aws-inventory stack functionality for GCP Projects.

## Compliance Stack
This will be a future addition to Antiope where Turner open-sources the Cloud Security Scorecards we've built for creating executive and technical owner visibility into security issues in each account.

## Local Customizations
Because the trigger if inventory and account detection is based on SNS Topics and state machines, it is easy to add your own enterprises customizations into the Antiope fold. We're just implementing this now so more details will be here soon.


## Structure of the Bucket

<pre>
    /Reports/ - Reports of AWS accounts generated by the inventory phase
    /Resources/ - All the json files collected in the Inventory Phase
    /Health/ - All the Personal Health Events
    /lambda-packages/ - location of the zip files hosting the lambda
</pre>

### Resource Prefix:
Most resources use the normal resource prefix (vpc- for VPC, i- for Instances, etc). Where the unique identifier for the resource didn't have a prefix, or where the resource name can be duplicated across accounts, Antiope prepends a resource prefix. The following prefixes are inventoried:

* bucket
* domain - Domains Registered via Route53 Domains. Each domain is globally unique, so AWS accounts aren't part of the object key
* hostedzone - Domains hosted in Route53. There can be multiple hosted zones with the same domain name, so the HostedZone ID is used
* role - IAM Roles. These are not globally unique, so the account_id is part of the object name
* user - IAM Users. These are not globally unique, so the account_id is part of the object name
