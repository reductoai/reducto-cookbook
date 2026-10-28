# r-1 mini on Amazon SageMaker

This folder shows how to deploy the r-1 mini model package from AWS Marketplace and call it.

- [`r-1-mini-sagemaker.ipynb`](r-1-mini-sagemaker.ipynb) does every step below with `boto3`: create the model, deploy a real-time endpoint, send both request bodies, read the blocks, handle a page that r-1 mini cannot process, run a batch transform job, and delete the resources.
- [`../README.md`](../README.md) describes the request and response contract.

## Prerequisites

- An accepted private offer for r-1 mini from Reducto, and the model package ARN. See [Access](../README.md#access).
- An IAM execution role for SageMaker. The trust policy must allow `sagemaker.amazonaws.com`. For batch transform, the role must also read the input prefix and write the output prefix in your S3 bucket.
- Service quota in your region for one `ml.g6e.2xlarge` for endpoint usage, and one for transform job usage.
- Python 3.9 or later with `boto3`, and AWS credentials that can call SageMaker and S3.

The commands below use these placeholders. Replace each one with your own value.

| Placeholder | Value |
|---|---|
| `<region>` | The region of your subscription, for example `us-east-2` |
| `<model-package-arn>` | The model package ARN from **Manage subscriptions > Configure** |
| `<execution-role-arn>` | The ARN of your SageMaker execution role |
| `<model-name>`, `<endpoint-name>`, `<transform-job-name>` | Names that you choose |
| `<bucket>` | An S3 bucket in the same region |

## 1. Subscribe

1. Send Reducto the 12-digit ID of the AWS account that will run r-1 mini. Reducto sends you a private offer link for that account.
2. Sign in to that AWS account, open the link, and accept the offer.
3. Open **AWS Marketplace > Manage subscriptions**, find r-1 mini, and select **Configure**.
4. Choose version 3 or later, and a region that offers `ml.g6e.2xlarge`. Copy the model package ARN.

## 2. Create the model

A model from an AWS Marketplace model package must run with network isolation.

```sh
aws sagemaker create-model --region <region> \
  --model-name <model-name> \
  --execution-role-arn <execution-role-arn> \
  --primary-container ModelPackageName=<model-package-arn> \
  --enable-network-isolation
```

## 3. Deploy a real-time endpoint

```sh
aws sagemaker create-endpoint-config --region <region> \
  --endpoint-config-name <endpoint-name> \
  --production-variants VariantName=primary,ModelName=<model-name>,InstanceType=ml.g6e.2xlarge,InitialInstanceCount=1,ContainerStartupHealthCheckTimeoutInSeconds=3600

aws sagemaker create-endpoint --region <region> \
  --endpoint-name <endpoint-name> \
  --endpoint-config-name <endpoint-name>

aws sagemaker wait endpoint-in-service --region <region> --endpoint-name <endpoint-name>
```

The endpoint can take 10 to 20 minutes to start. `ContainerStartupHealthCheckTimeoutInSeconds=3600` gives the model enough time to load.

## 4. Invoke the endpoint

Send a sample page from the root of this repository:

```sh
aws sagemaker-runtime invoke-endpoint --region <region> \
  --endpoint-name <endpoint-name> \
  --content-type application/json \
  --body fileb://r-1-mini/samples/invoice.request.json \
  invoice.out.json

cat invoice.out.json
```

The answer has the same shape as [`../samples/invoice.response.json`](../samples/invoice.response.json): a `blocks` list and `usage`.

To send your own page, put a base64 PNG or JPEG in the page-parse body:

```python
import base64
import json

import boto3

runtime = boto3.client("sagemaker-runtime", region_name="<region>")
with open("page.png", "rb") as page:
    body = json.dumps({"image_b64": base64.b64encode(page.read()).decode()})

response = runtime.invoke_endpoint(
    EndpointName="<endpoint-name>", ContentType="application/json", Body=body
)
for block in json.loads(response["Body"].read())["blocks"]:
    print(block["type"], block["bbox"], block["content"][:80])
```

When r-1 mini cannot process a page, `invoke_endpoint` raises a `ModelError` with `OriginalStatusCode` 422 and the body `{"detail": "input could not be processed"}`. Do not retry that page. The notebook shows how to handle it.

## 5. Run a batch transform job

1. Put one JSON request body in each S3 object, for example `s3://<bucket>/r-1-mini/input/invoice.json`. Both request bodies work.
2. Start the job:

   ```sh
   aws sagemaker create-transform-job --region <region> \
     --transform-job-name <transform-job-name> \
     --model-name <model-name> \
     --batch-strategy SingleRecord \
     --max-payload-in-mb 6 \
     --max-concurrent-transforms 1 \
     --transform-input '{"DataSource": {"S3DataSource": {"S3DataType": "S3Prefix", "S3Uri": "s3://<bucket>/r-1-mini/input/"}}, "ContentType": "application/json", "SplitType": "None"}' \
     --transform-output '{"S3OutputPath": "s3://<bucket>/r-1-mini/output/", "Accept": "application/json", "AssembleWith": "None"}' \
     --transform-resources InstanceType=ml.g6e.2xlarge,InstanceCount=1

   aws sagemaker wait transform-job-completed-or-stopped --region <region> --transform-job-name <transform-job-name>
   ```

3. Read the results. Each input object gets one `<input>.out` object in the output prefix, with the same JSON as the real-time response. Check each `.out` file for `detail` before you read `blocks`. A page that r-1 mini cannot process gives `{"detail": "input could not be processed"}`, and the job continues.

Any other error on a record, for example an image that is not a readable PNG or JPEG, fails the whole job. Check each image before you upload it. If the job fails, run `aws sagemaker describe-transform-job` and read `FailureReason`.

A transform job starts its own instance and stops it when the job ends. It does not need a real-time endpoint, so you can delete the endpoint first.

## 6. Delete the resources

AWS bills the endpoint instance while the endpoint exists. Delete the endpoint when you are done:

```sh
aws sagemaker delete-endpoint --region <region> --endpoint-name <endpoint-name>
aws sagemaker delete-endpoint-config --region <region> --endpoint-config-name <endpoint-name>
aws sagemaker delete-model --region <region> --model-name <model-name>
```

Delete the batch input and output objects in S3 if you do not need them.

To unsubscribe, delete all endpoints and models from the model package first. Then open **AWS Marketplace > Manage subscriptions** and cancel the r-1 mini subscription.
