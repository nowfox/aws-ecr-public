# Public AWS Elastic Container Registry
本文是对AWS China区域的修改版，Global区域的参见[monken/aws-ecr-public](https://github.com/monken/aws-ecr-public)
Host any Elastic Container Registry (ECR) publicly on a custom domain using this serverless proxy.

Give it a spin:

```bash
# pull a container from a registry named nginx with no authentication
docker pull 3bq24dacgg.execute-api.cn-northwest-1.amazonaws.com.cn/nginx:alpine
```

## Solution Overview

ECR doesn't support [public registries](https://aws.amazon.com/ecr/faqs/). Instead, the docker client needs to authenticate with ECR using AWS IAM credentials which requires the AWS CLI or an SDK that can generate those credentials.

If you would like to make your registries publicly available then this solution can help. It deploys an API Gateway and a Lambda function that act as a proxy for AWS ECR. [Custom authentication](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-use-lambda-authorizer.html) can easily be added in the API Gateway. Roll your own JWT-based authentication or whatever you desire. Additionally, you can configure the [API Gateway to be private](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-private-apis.html) and thus limit access to docker clients within your VPC.

![diagram](docs/aws-ecr-public.svg)

## Deploy

[![launch](docs/launch-stack.svg)](https://cn-northwest-1.console.amazonaws.cn/cloudformation/home?#/stacks/create/review?filter=active&templateURL=https%3a%2f%2fnowfox.s3.cn-northwest-1.amazonaws.com.cn%2faws-ecr-public%2fv1.1.1%2fECR-Public-cn.json&stackName=ECR-public)

[Download Template](https://nowfox.s3.cn-northwest-1.amazonaws.com.cn/aws-ecr-public/v1.1.1/ECR-Public-cn.json)


### Template Parameters

| Parameter | Required | Description |
| -- | -- | -- |
| DomainName | No | If provided an ACM Certificate and API Domain Name will be created
| ValidationDomain | No | Overwrite default Validation Domain for ACM Certificate
| ValidationMethod | Yes, *Default: EMAIL* | Allow you to use DNS instead of EMAIL for Certificate validation

## FAQ

### How can I host this proxy on a custom domain?

Simply provide the `DomainName` parameter when you create the stack. This will create an ACM certificate and API Domain Name resource. The Regional Domain Name and Hosted Zone ID can be found in the outputs tab of the stack. You will need those to create the DNS record in Route 53 (or similar DNS service).

For Route 53, open your hosted zone, create a **New Record Set**, enter the domain name, set **Alias** to **Yes** and paste the `RegionalDomainName` in the **Alias Target** field.

### How can I restrict access to certain registries?

By default all registries in the account and region will be made publicly available. To limit the number of publicly available repositores, attach a custom policy to the Lambda execution role (look for `${AWS::StackName}-LambdaRole-*`). The following policy will restrict public access to the `myapp` repository (make sure you replace the variables with your region and account id).

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "ecr:GetDownloadUrlForLayer",
                "ecr:BatchGetImage"
            ],
            "NotResource": [
                "arn:aws-cn:ecr:${AWS::Region}:${AWS::AccountId}:repository/myapp"
            ],
            "Effect": "Deny"
        }
    ]
}
```

## Develop

```
npm install --global cfn-include
make build
make test  # create/update CloudFormation stack
make clean # delete CloudFormation stack
```

## In the works

* Cross-account and cross-region access to registries
* Tag-based permissions
* Implement additional endpoints for listing images and tags
