---
title: 'PDF Generator using Terraform, AWS, Lambda and API Gateway'
author: gareth
date: 2021-07-05 00:00:00 +10:00
featured_image: /assets/img/2021/07/05/header.png
excerpt: First time working with Terraform to make a HTML/CSS to PDF service in AWS using Lambda and API Gateway
categories: [Development]
tags: [development, javascript, aws]
mermaid: true
math: true
---

The application I'm currently working on has a lot of old documents created using a crystal reports editor and references DLLs in a folder in source control instead of using Nuget. As we're trying to upgrade this solution to .NET 5 we need to move away from local DLLs. As part of this we decided to redo how our PDFs are being generated.

The approach: create a micro service that accepts a HTML payload and returns a PDF file. The stack we decided to use for this;

* AWS 
* Terraform
* Lambda
* API Gateway
* S3

The intent was to use Puppeteer and Chrome to save the PDF from the browser directly, while this worked fine locally it obviously wouldn't work in a Lambda because it wouldn't have Chrome available. 

The solution was to use a package called `chrome-aws-lambda` from here: [https://github.com/alixaxel/chrome-aws-lambda](https://github.com/alixaxel/chrome-aws-lambda). Using this we could use the following code to generate a PDF file.

### The Lambda

```typescript
import * as Sentry from "@sentry/serverless";
import * as SentryTracing from "@sentry/tracing";
import { CaptureConsole } from "@sentry/integrations";
import { APIGatewayProxyEvent, APIGatewayProxyResult } from "aws-lambda";
import { generate } from "./services/pdfService";
import { save } from './services/fileService'
import logger from "./utils/logger";

export const handler = Sentry.AWSLambda.wrapHandler(
  async (event: APIGatewayProxyEvent): Promise<APIGatewayProxyResult> => {
    const document = event.body;

    if (!document) {
      return {
        statusCode: 400,
        body: "Bad request, missing document",
      };
    }

    try {
      const pdf = await generate(document);
      const result = await save(pdf);

      return {
        statusCode: 200,
        headers: {
          "Content-Type": "application/json"
        },
        body: JSON.stringify(result)
      };
    } catch (e) {
      logger.error(e, "Error processing file");

      return {
        statusCode: 500,
        body: "Error creating PDF",
      };
    }
  }
);

```

### The PDF Service

```typescript
import chromium from 'chrome-aws-lambda';

const generate = async (body: string): Promise<any> => {
	const browser = await chromium.puppeteer.launch({
		args: chromium.args,
		defaultViewport: chromium.defaultViewport,
		executablePath: await chromium.executablePath,
		headless: chromium.headless,
		ignoreHTTPSErrors: true
	});

	const page = await browser.newPage();
	await page.setContent(body);

  return page.pdf({
    preferCSSPageSize: true
  });
}

export { generate };
```

### Terraform

The `chrome-aws-lambda` package was deployed to AWS as a layer for our lambda to reference. 

```terraform
resource "aws_lambda_layer_version" "lambda_layer" {
  filename   = "chrome_aws_lambda-10.zip"
  layer_name = "chrome-aws-lambda-${var.environment}"

  compatible_runtimes = ["nodejs12.x"]
}
```

Connect this up to the lambda 

```terraform
resource "aws_lambda_function" "lambda" {
  filename      = var.lambda_filename
  function_name = "${var.service_name}-${var.environment}"
  role          = aws_iam_role.lambda.arn
  handler       = "index.handler"
  runtime       = "nodejs12.x"
  memory_size   = 1024
  publish       = true
  timeout       = 800
  layers        = [aws_lambda_layer_version.lambda_layer.arn]
  environment {
    variables = {
      SENTRY_DSN      = var.sentry_dsn
      ENVIRONMENT     = var.environment
      PACKAGE_VERSION = var.lambda_version
      SERVICE_NAME    = var.service_name
      BUCKET_NAME     = var.bucket_name
    }
  }
}
```

The PDF can be returned directly  via the lambda however there is a 6mb lambda payload limit to consider so the lambda takes the PDF file and saves it to S3 which is configured to be removed after 1 day. A signed download URL is returned with a 5 minute expiry. 

This can then sit behind API Gateway and serve up beautiful PDFs like this:

![PDF](/assets/img/2021/07/05/pdf.png)