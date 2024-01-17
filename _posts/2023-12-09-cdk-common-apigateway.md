---
title: Deploying Resources Into Shared API Gateway
author: gareth
date: 2023-12-09 07:00:00 +1000
excerpt: Deploying Resources Into Shared API Gateway
categories: [Development]
tags: [development, aws, cloud]
---

> This post only gets you part way, it doesn't update the stage when deploying - [Updating the stage in ApiGateway]({% post_url 2024-01-18-updating-stage-apigateway %})
{: .prompt-info }

We wanted to have a shared Cloudfront Distribution and API Gateway and deploy multiple resources from potentially different stacks as endpoints on this API Gateway, this way we can control access and authenication at a centralised location.

This assumes you already have an API Gateway and a VPC deployed and `CommonVpcId`, `CommonApiGatewayId` and `CommonRootResorceId` available in SSM.

This first class looks up the shared Vpc and ApiGateway details based on these SSM parameters. And provides a `createLambda` function. This function takes a url path for the lambda, a file path for the lambda and creates a `NodeJsFunction`. It then creates resources for the path and an http method then finally a deployment and this when deployed will add that resource to your API Gateway.

### CommonApiGateway.ts
```typescript
export class CommonApiGateway {
  private Stack: cdk.Stack;
  private Vpc: ec2.IVpc;
  private ApiGateway: apigateway.IRestApi;

  constructor(stack: cdk.Stack) {
    this.Stack = stack;

    const vpcId = ssm.StringParameter.valueFromLookup(
      stack,
      'CommonVpcId'
    );

    const apiGatewayId = ssm.StringParameter.valueFromLookup(
      stack,
      'CommonApiGatewayId'
    );

    const apiGatewayRootResourceId = ssm.StringParameter.valueFromLookup(
      stack,
      'CommonRootResourceId'
    );

    this.Vpc = ec2.Vpc.fromLookup(stack, 'CommonVPC', {
      vpcId: vpcId,
    });

    this.ApiGateway = apigateway.RestApi.fromRestApiAttributes(
      stack,
      'CommonApiGateway',
      {
        restApiId: apiGatewayId,
        rootResourceId: apiGatewayRootResourceId,
      }
    );
  }

  public createLambda(
    name: string,
    config: LambdaConfig,
    props?: NodejsFunctionProps
  ): NodejsFunction {
    const lambdaConfig = {
      runtime: lambda.Runtime.NODEJS_LATEST,
      handler: 'index.main',
      entry: config.src,
      vpc: this.Vpc,
      vpcSubnets: {
        subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS,
      },
    };

    // add in other lambda options & allow overides if required
    const mergedProps = {...lambdaConfig, ...props};

    const lambdaInstance = new NodejsFunction(
      this.Stack,
      `Lambda${name}`,
      mergedProps
    );

    const resource = this.ApiGateway.root.resourceForPath(config.path);

    if (typeof config.corsOptions !== 'undefined') {
      resource.addCorsPreflight(config.corsOptions);
    }

    new apigateway.Method(
      this.Stack,
      `${name}${config.httpMethod}GatewayMethod`,
      {
        httpMethod: config.httpMethod,
        resource: resource,
        integration: new apigateway.LambdaIntegration(lambdaInstance, {
          proxy: true,
        }),
      }
    );

    new apigateway.Deployment(this.Stack, `${name}Deployment`, {
      api: this.ApiGateway,
    });

    return lambdaInstance;
  }
}

```

The following shows how the `CommonApiGateway` is used. The main point to note here is that the root resource of the path (`'secondarystack/v1/test'`), in this case `secondarystack` has to be unique to this stack. So each stack you deploy into the shared API Gateway has to have a unique root resource otherwise it will fail to create it.

But this actually works quite nicely as you would probably want to group lambdas with related functionality into the same repository/stack.

So in a pet store example you could have a stack that handles stock. Which would have the endpoints `stock/v1/list` and `stock/v1/add` then a secondary stack to manage orders with the endpoints `order/v1/list` and `order/v1/create`. Or even use `order/list` and `order/create`.


### ASecondaryStack.ts
```typescript
export class ASecondaryStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const gateway = new CommonApiGateway(this);

    const corsOptions = {
      allowHeaders: [
        'Content-Type',
        'X-Amz-Date',
        'Authorization',
        'X-Api-Key',
      ],
      allowMethods: ['OPTIONS', 'GET'],
      allowCredentials: true,
      allowOrigins: ['http://localhost:3000'],
    };

    gateway.createLambda(
      'V1Test',
      {
        path: 'secondarystack/v1/test',
        src: path.join(__dirname, '../lambdas/test/index.ts'),
        httpMethod: 'GET',
        corsOptions,
      },
      {timeout: cdk.Duration.seconds(5), runtime: lambda.Runtime.NODEJS_20_X}
    );

    gateway.createLambda(
      'V2Test',
      {
        path: 'secondarystack/v2/test',
        src: path.join(__dirname, '../lambdas/test/index.ts'),
        httpMethod: 'GET',
        corsOptions,
      },
      {runtime: lambda.Runtime.NODEJS_20_X}
    );
  }
}

```
