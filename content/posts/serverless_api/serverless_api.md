---
title: "Using Terraform for Amazon API Gateway Deployment"
date: "2024-03-11"
tags: ["terraform", "lambda", "aws", "api", "dynamodb"]
#categories: [""]
series: ["Terraform"]
ShowToc: true
TocOpen: false
cover:
    image: "/posts/serverless_api/cover.png"
    hiddenInSingle: true
summary: "This article offers a comprehensive tutorial on creating and deploying an Amazon API Gateway resource using Terraform, and demonstrates how to invoke a Lambda function through an HTTPS endpoint."
---

This guide provides a detailed walkthrough on how to create and deploy an Amazon API Gateway resource using Terraform. It also illustrates how to call a Lambda function via an HTTPS endpoint, which will interact directly with DynamoDB.

One of the key advantages of using Terraform is the ability to launch the entire project with a single click. The concept of infrastructure as code is vital for projects as it enables tracking of configuration modifications and promotes effective collaboration within large teams.

For Mac users, you can install [Homebrew](https://docs.brew.sh/Installation) and set up AWS CLI, Terraform, and Bruno.

```bash
brew install awscli
brew install terraform
brew install bruno # An Opensource API client, we'll use for testing
aws configure # Follow the steps to provide your AWS account credentials
```

**(Do not run this in production, use a testing account instead)**
```bash
git clone https://github.com/tbalza/lambda-dynamodb-api.git
cd lambda-dynamodb-api
```
```bash
terraform init
```
This will initialize terraform, and download all the necessary libraries and modules, so that our HCL code in main.tf can interact with AWS using the credentials we set up earlier with the CLI. All the needed files will be stored in the .terraform folder for later use.
![init](/posts/serverless_api/init.png)


```bash
terraform plan -out tfplan
```
This command lays out all the changes that will take place, evaluating the current state of resources and what is indicated as an end result in our code.
![plan](/posts/serverless_api/plan.png)

```bash
terraform apply -auto-approve tfplan
```
After the apply is run, it will go ahead and create and configure our infrastructure as described in our code.
![apply](/posts/serverless_api/apply.png)


after you're done you can clean up by

```bash
terraform destroy
```
Ideally you want to `terraform plan -destroy -out tfplan` and then `terraform apply tfplan` as this allows you to double check all changes before commiting them to aws.

## Break down of all the sections

Using the commands provided above, you can swiftly set up and test the infrastructure. Additionally, you can easily clean it up to avoid any unexpected charges in the future. In the upcoming sections, we'll provide a brief explanation of the function of each block.

### Setup IAM Permissions

#### Create custom IAM policy
```hcl

module "iam_policy" {
  source  = "terraform-aws-modules/iam/aws//modules/iam-policy"
  version = "~> 5.37.1"

  name        = "lambda-apigateway-role-policy"
  description = "Custom policy with permission to DynamoDB and CloudWatch Logs"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid = "Stmt1428341300017"
        Action = [
          "dynamodb:DeleteItem",
          "dynamodb:GetItem",
          "dynamodb:PutItem",
          "dynamodb:Query",
          "dynamodb:Scan",
          "dynamodb:UpdateItem"
        ]
        Effect   = "Allow"
        Resource = "*"
      },
      {
        Sid      = ""
        Resource = "*"
        Action = [
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents"
        ]
        Effect = "Allow"
      }
    ]
  })
}
```
This is a custom stand alone policy that allows the principal who assumes it to edit DynamoDB and CW Logs.

#### Create role for lambda function, attach custom IAM policy and trusted entity

```hcl
module "iam_assumable_role_lambda" {
  source  = "terraform-aws-modules/iam/aws//modules/iam-assumable-role"
  version = "~> 5.37.1"

  create_role = true
  role_name   = "lambda-apigateway-role"

  create_custom_role_trust_policy = true
  custom_role_trust_policy        = data.aws_iam_policy_document.custom_trust_policy.json

  custom_role_policy_arns = [module.iam_policy.arn] # Get the ARN from the iam_policy module
}

# Create trusted entity
data "aws_iam_policy_document" "custom_trust_policy" {
  statement {
    effect  = "Allow"
    actions = ["sts:AssumeRole"]

    principals {
      type        = "Service"
      identifiers = ["lambda.amazonaws.com"]
    }
  }
}
```

This creates a role, and attaches the previous policy to that role. Additionally it give the role the trusted entity required for the lambda role to assume it.

#### Create trigger equivalent to explicitly grant permissions to the API Gateway to invoke your Lambda function
```hcl
resource "aws_lambda_permission" "apigw" {
  statement_id  = "AllowExecutionFromAPIGateway"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.example.arn
  principal     = "apigateway.amazonaws.com"

  source_arn = "${aws_api_gateway_deployment.dev.execution_arn}/*"
}
```

When creating resources via ClickOps, the trigger is automatically assigned to the lambda function when it is associated with the API, however via CLI we need to explicitly set this in order to have the correct permissions.

![trigger](/posts/serverless_api/trigger.png)
### Create Lambda Function

#### lambda_function.py

```python
from __future__ import print_function

import boto3
import json

print('Loading function')


def lambda_handler(event, context):
    '''Provide an event that contains the following keys:

      - operation: one of the operations in the operations dict below
      - tableName: required for operations that interact with DynamoDB
      - payload: a parameter to pass to the operation being performed
    '''
    #print("Received event: " + json.dumps(event, indent=2))

    operation = event['operation']

    if 'tableName' in event:
        dynamo = boto3.resource('dynamodb').Table(event['tableName'])

    operations = {
        'create': lambda x: dynamo.put_item(**x),
        'read': lambda x: dynamo.get_item(**x),
        'update': lambda x: dynamo.update_item(**x),
        'delete': lambda x: dynamo.delete_item(**x),
        'list': lambda x: dynamo.scan(**x),
        'echo': lambda x: x,
        'ping': lambda x: 'pong'
    }

    if operation in operations:
        return operations[operation](event.get('payload'))
    else:
        raise ValueError('Unrecognized operation "{}"'.format(operation))
```

This lambda function is written in Python, and works with any table name we specify via the POST method. Actions such as, create, read, update, delete, list, echo and ping, are supported.

#### Zip the lambda_function.py to enable uploading to AWS

```hcl
data "archive_file" "lambda_zip" {
  type        = "zip"
  source_file = "${path.module}/lambda_function.py"
  output_path = "${path.module}/lambda_function.zip"
}
```
We use built in TF tools to zip the Python script, which is needed in order to upload it to AWS via this method.

#### Create lambda function from lambda_function.py
```hcl
resource "aws_lambda_function" "example" {
  function_name = "LambdaFunctionsOverHttps"
  handler       = "lambda_function.lambda_handler"
  runtime       = "python3.12"

  filename = data.archive_file.lambda_zip.output_path

  # Associate function to previously created role
  role = module.iam_assumable_role_lambda.iam_role_arn # Get the ARN from the iam_assumable_role_lambda module

}
```
Finally, we upload the lambda function and attach the role we created earlier.

### DynamoDB
#### Create a simple dynamodb table
```hcl
module "dynamodb_table" {
  source  = "terraform-aws-modules/dynamodb-table/aws"
  version = "~> 4.0.1"

  name     = "lambda-apigateway"
  hash_key = "id" # primary key

  attributes = [
    {
      name = "id"
      type = "S"
    }
  ]
}
```

This creates a table with an "id" primary column that will expect strings.

### API
#### Create API
```hcl
resource "aws_api_gateway_rest_api" "DynamoDBOperations" {
  name           = "DynamoDBOperations"
  description    = "API for DynamoDB Operations"
  api_key_source = "HEADER"
  endpoint_configuration {
    types = ["REGIONAL"]
  }
}
```
This creates de API by itself, and defines the end point configuration type.

#### Create resource
```hcl
resource "aws_api_gateway_resource" "DynamoDBManager" {
  rest_api_id = aws_api_gateway_rest_api.DynamoDBOperations.id
  parent_id   = aws_api_gateway_rest_api.DynamoDBOperations.root_resource_id
  path_part   = "dynamodbmanager"
}
```
These are the various parts of your API that clients can access. They are represented as a hierarchical structure similar to a file path. For example, in the API endpoint https://api.example.com/users/profile, 'users' and 'profile' are resources. Resources can have child resources, creating a tree-like structure.

#### Create POST method
```hcl
resource "aws_api_gateway_method" "post" {
  rest_api_id      = aws_api_gateway_rest_api.DynamoDBOperations.id
  resource_id      = aws_api_gateway_resource.DynamoDBManager.id
  http_method      = "POST"
  authorization    = "NONE"
  api_key_required = false
}
```
These are the HTTP methods (also known as verbs) that clients can use to interact with the resources. Common methods include GET, POST, PUT, DELETE, and PATCH. Each method represents a different type of operation that can be performed on a resource. For example, a GET method on a 'users' resource might retrieve a list of users, while a POST method on the same resource might create a new user.

#### Link API to Lambda function
```hcl
resource "aws_api_gateway_integration" "lambda" {
  rest_api_id = aws_api_gateway_rest_api.DynamoDBOperations.id
  resource_id = aws_api_gateway_resource.DynamoDBManager.id
  http_method = aws_api_gateway_method.post.http_method

  integration_http_method = "POST"
  type                    = "AWS"
  uri                     = aws_lambda_function.example.invoke_arn
  passthrough_behavior    = "WHEN_NO_MATCH"
}
```
`uri = aws_lambda_function.example.invoke_arn`: This line sets the URI of the integrated backend. In this case, it's a Lambda function, and the ARN (Amazon Resource Name) used to invoke the function is retrieved from another resource named `example` of type `aws_lambda_function`.


#### Create response code
```hcl
resource "aws_api_gateway_method_response" "response" {
  rest_api_id = aws_api_gateway_rest_api.DynamoDBOperations.id
  resource_id = aws_api_gateway_resource.DynamoDBManager.id
  http_method = aws_api_gateway_method.post.http_method
  status_code = "200"
}
```
`http_method = aws_api_gateway_method.post.http_method`: This line sets the HTTP method for the method response. The method is retrieved from another resource of type `aws_api_gateway_method` with the name `post`.

status_code = "200": This line sets the HTTP status code for the method response. In this case, it's "200", which typically represents a successful HTTP request.

#### Integrate response code
```hcl
resource "aws_api_gateway_integration_response" "lambda" {
  depends_on  = [aws_api_gateway_integration.lambda, aws_api_gateway_method_response.response]
  rest_api_id = aws_api_gateway_rest_api.DynamoDBOperations.id
  resource_id = aws_api_gateway_resource.DynamoDBManager.id
  http_method = aws_api_gateway_method.post.http_method
  status_code = aws_api_gateway_method_response.response.status_code
}
```
`depends_on = [aws_api_gateway_integration.lambda, aws_api_gateway_method_response.response]`: This line specifies that the creation of this resource depends on the successful creation of other resources in order the execute correctly.

This creates an integration response that depends on the successful creation of the API Gateway integration and method response.

Both blocks are part of configuring an API Gateway to handle responses from a backend service, in this case, a Lambda function.

#### Deploy in "Dev"
```hcl
resource "aws_api_gateway_deployment" "dev" {
  depends_on  = [aws_api_gateway_integration.lambda]
  rest_api_id = aws_api_gateway_rest_api.DynamoDBOperations.id
  stage_name  = "dev"
}
```
This deploys our API to the "dev" stage, generating our invoke URL that we will use to interact with the API

![api](/posts/serverless_api/api.png)

### Output API Invoke URL
```hcl
output "api_invoke_url" {
  description = "api_invoke_url"
  value       = "${aws_api_gateway_deployment.dev.invoke_url}/${aws_api_gateway_resource.DynamoDBManager.path_part}"
}
```
This line from `outputs.tf` prints out our Invoke URL generated earlier, and displays it after `terraform apply`.

![output](/posts/serverless_api/output.png)

### Test with API Client

Open Bruno. Create Collection > New Request > Past the invoke URL generated inside URL: POST

![api-client](/posts/serverless_api/api-client.png)

Then in the BODY section, select Raw > JSON, past the code below, and press cmd+enter to execute
```json
{
    "operation": "create",
    "tableName": "lambda-apigateway",
    "payload": {
        "Item": {
            "id": "1234ABCD",
            "number": 88
        }
    }
}
```

### Verify Results in DynamoDB

Go to DynamoDB > Tables > lambda-apigateway > Explore Table Items
![dynamodb](/posts/serverless_api/dynamodb.png)

Here you'll see that our POST payload, sent to the API Invoke URL, interacts with Lambda, which in turn modifies our DynamoDB table.


### Clean up
![destroy](/posts/serverless_api/destroy.png)

Run this command to delete all our previously created resources and configurations.

### Take Aways

We've created an API that is publicly accessible via our Invoke URL, which allows us to interact with the DynamoDB table directly via a Lambda function.

In future articles we'll delve deeper into GitOps, and CI/CD pipelines building on this knowledge, and implement authentication, DNS and certificate configurations.