---
title: Deploying Resources Into Shared API Gateway - Part 2 
author: gareth
date: 2023-12-09 07:00:00 +1000
excerpt: Deploying Resources Into Shared API Gateway - Updating the stage in apigateway
categories: [Development]
tags: [development, aws, cloud]
---

The previous post on this topic didn't quite work, the resources were deployed into the API however the deployment didn't work correctly and update the stage. As we hadn't gone far into the dev process I didn't test the API calls outside of the API Gateway console.

### CDK

Currently in CDK there's [no way to reuse a stage when deploying into a stack](https://github.com/aws/aws-cdk/issues/25582). So this code doesn't work if the `dev` stage is already created in another stack.

```typescript
// production stage
const prodLogGroup = new logs.LogGroup(this, "PrdLogs");
const api = new apigateway.RestApi(this, 'books', {
  deployOptions: {
    accessLogDestination: new apigateway.LogGroupLogDestination(prodLogGroup),
    accessLogFormat: apigateway.AccessLogFormat.jsonWithStandardFields(),
  },
});
const deployment = new apigateway.Deployment(this, 'Deployment', {api});

// development stage
const devLogGroup = new logs.LogGroup(this, "DevLogs");
new apigateway.Stage(this, 'dev', {
  deployment,
  accessLogDestination: new apigateway.LogGroupLogDestination(devLogGroup),
  accessLogFormat: apigateway.AccessLogFormat.jsonWithStandardFields({
    caller: false,
    httpMethod: true,
    ip: true,
    protocol: true,
    requestTime: true,
    resourcePath: true,
    responseLength: true,
    status: true,
    user: true,
  }),
});
```

### Solution 

So the workaround for now, I wrote up a deployment script that sits in `bin/deploy.sh`. Thie script does the following.

1. npm install
2. cdk deploy
3. Uses AWS CLI to `describe-stacks` and pull out the shared `restApiId`
4. Creates a deployment with that `restApiId` and stage name
5. Uses the AWS CLI to `update-stage` with the deployment ID and `restApiId`

The script I threw together to do this looks something like this

```bash
#!/bin/sh
# This script will deploy the stack and then trigger a manual deployment on the API Gateway stage
# Updating the stage isn't currently supported by CDK: https://github.com/aws/aws-cdk/issues/25582
NODE_ENV="${1:-sandbox}"
REQUIRE_APPROVAL="${2:-broadening}"
echo "Deploying to $NODE_ENV"

# npm install
npm i

# deploy
NODE_ENV="${1:-sandbox}" npx aws-cdk deploy --ci --require-approval $REQUIRE_APPROVAL

# grab details from common stack
aws cloudformation describe-stacks --stack-name 'StackNameHere' | jq '.Stacks | .[] | .Outputs | reduce .[] as $i ({}; .[$i.OutputKey] = $i.OutputValue)' > build/common.json

# get the common api gateway from the stack details
RESI_ID=$(jq -r '.restApiId' 'build/common.json')

# Create a deploymnt manually and output to json file
echo "Creating deployment on $RESI_ID on $NODE_ENV stage"
aws apigateway create-deployment --rest-api-id $RESI_ID --stage-name $NODE_ENV > build/deployment.json

# Grab the deployment Id from the json file 
DEPLOYMENT_ID=$(jq -r '.id' 'build/deployment.json')

# Update the stage with the deployment ID
echo "Updating stage with deployment $DEPLOYMENT_ID on $RESI_ID on $NODE_ENV stage"
aws apigateway update-stage --rest-api-id $RESI_ID --stage-name $NODE_ENV --patch-operations op='replace',path='/deploymentId',value="$DEPLOYMENT_ID" > build/update.json

echo 'Deployment complete'
```