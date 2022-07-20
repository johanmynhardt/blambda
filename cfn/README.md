# Simplifying Resource Management

The current project works well if you have to create one or two functions and
managing the resources by hand is manageable. When you have to start scaling
beyond a few, setting up the roles and permissions each time becomes tedious.

This is a proposal to make use of CloudFormation templates, instead of managing
a policy and trust JSON and then doing the boilerplate wiring up IAM. When
tearing down the resources, it can be done so cleanly because of it being tied
into a stack.

Through trial-and-error it was discovered that a full stack can't be done in
CloudFormation only, and the dependency of having a deployment bucket first
starts showing.

## Manual first attempts

Below are the summary steps for the initial CloudFormation templates:

``` bash
# Build bb.zip and handler.zip first.
# Deploy "base"
aws cloudformation deploy --stack-name blambda-base --template-file blambda-base.template
aws cloudformation list-stack-resources --stack-name blambda-base

# Retrieve created bucket value
aws s3 cp --region af-south-1 ../blambda/target/bb.zip s3://blambda-deploymentbucket-1cjdcff3hfh94
aws s3 cp --region af-south-1 ../blambda/target/handler.zip s3://blambda-deploymentbucket-1cjdcff3hfh94

aws --profile jo \
  cloudformation deploy \
  --stack-name blambda-app \
  --template-file blambda-app.template \
  --capabilities CAPABILITY_IAM \
  --parameter-overrides DeploymentBucket=${AWS_BUCKET}
```

## aws-api

Steps to "initialise" blambda foundation and deploy layered function:

- Deploy foundation template to create deployment bucket
    - `blambda-base.template`
- Retrieve bucket value via `DescribeStackResources`
    - Maybe store this in a `.blambda/config.edn`?
- Deploy the function/app stack template `blambda-app.template`
    - Requires:
        - DeployBucket
        - layer zip copied into bucket
        - function zip copied into bucket
    - Results in:
        - CloudFormation Stack with permissions wired up
        - which can be torn down in one click/command (`{:op :DeleteStack :request {:StackName "the-name"}}`)

**Two templates are simpler.**
The downside is that it binds the layer and function resources.

- `blambda-base.template`
- ~~`blambda-app.template`~~

**Three templates are more flexible.**
Manage layer in 2nd template.
Manage function in 3rd template.

- `blambda-base.template`
- `blambda-layer.template`
- `blambda-app.template`

# REPL exploration

```clojure
(aws/invoke cfn
 {:op :CreateStack
  :request {:StackName "blambda-base"
            :TemplateBody (slurp "cfn/blambda-base.template")}})

=> {:StackId "arn:aws:cloudformation:${AWS::Region}:${AWS::AccountId}:stack/blambda-base/24c915d0-03b7-11ed-a4ae-0a4d90c5e978"}
```

```clojure
(pprint (aws/invoke cfn {:op :DescribeStackResources :request {:StackName "blambda-base"}}))

=> 
{:StackResources
 [{:ResourceType "AWS::S3::Bucket",
   :StackId
   "arn:aws:cloudformation:${AWS::Region}:${AWS::AccountId}:stack/blambda-base/24c915d0-03b7-11ed-a4ae-0a4d90c5e978",
   :PhysicalResourceId "bl-base-deploymentbucket-uge2t2o06acd",
   :StackName "blambda-base",
   :LogicalResourceId "DeploymentBucket",
   :DriftInformation {:StackResourceDriftStatus "NOT_CHECKED"},
   :Timestamp #inst "2022-07-14T20:54:49.190-00:00",
   :ResourceStatus "CREATE_COMPLETE"}]}

;; (:PhysicalResourceId (first (filter (comp #(= % "DeploymentBucket") :LogicalResourceId) (:StackResources describe-stack-resources))))
```

Copy zips to Deploy Bucket here...

Then deploy layer + function.

```clojure
(aws/invoke cfn
  {:op :CreateStack
   :request {:StackName "blambda-app"
             :TemplateBody (slurp "cfn/blambda-app.template")
             :Capabilities ["CAPABILITY_IAM"]
             :Parameters [{:ParameterKey "DeploymentBucket"
                           :ParameterValue "bl-base-deploymentbucket-uge2t2o06acd"}]}})

```
