# Image Compression Workflow with Terraform

## Project Description

This project focuses on automating the compression of `.jpg` images using AWS native services and Infrastructure as Code (IaC) with Terraform.

### AWS Services Used

- Amazon S3
- AWS Lambda
- IAM Roles and Policies
- Amazon SNS
- Amazon CloudWatch
- Docker (for dependency packaging)

---

## Key Features

- **Source S3 Bucket**: Entry point where users upload `.jpg` images.
- **Lambda Function**: Triggered by S3 events to compress images.
- **Pillow Lambda Layer**: Provides Python image-processing dependencies.
- **Destination S3 Bucket**: Stores compressed images.
- **Failed S3 Bucket**: Stores images that fail compression.
- **SNS Notifications**: Sends email notifications on success or failure.

---

## High-Level Workflow

![alt text](./Images/image-compression-images/image%20compression%20and%20notifier%20workflow@4x.png)

1. User uploads a `.jpg` file to the source bucket.
2. S3 triggers the Lambda function.
3. Lambda compresses the image using Pillow.
4. Compressed image is stored in the destination bucket.
5. Failures are routed to a failed bucket.
6. SNS sends notifications to the admin.
7. Logs are written to CloudWatch.

---

## Repository Setup

I used Git for version control and configured a `.gitignore` file to prevent sensitive files and large build artifacts from being pushed to GitHub.

## ![alt text](./images/image-compression-images/image1.png)

## Provider Configuration

The AWS provider was configured with the region set to `eu-west-1`.  
This region represents an AWS geographical area containing multiple availability zones.

## ![alt text](./images/image-compression-images/image2.png)

---

## S3 Buckets Provisioning

Three buckets were provisioned:

- **Source Bucket** – for image uploads
- **Destination Bucket** – for compressed images
- **Failed Bucket** – for images that fail compression

## ![alt text](./images/image-compression-images/image3.png)

All buckets use `force_destroy = true` to allow Terraform to delete them even when non-empty.

---

## IAM Role and Policies

An IAM role was created for the Lambda function. Roles allow AWS services to assume temporary permissions.
![alt text](./images/image-compression-images/image4.png)

### Policies Attached

- `s3:GetObject` – Read images from source bucket
- `s3:PutObject` – Write images to destination and failed buckets
- `sns:Publish` – Send email notifications
- CloudWatch Logs permissions – Enable debugging and monitoring
  ![alt text](./images/image-compression-images/image5.png)

---

## Lambda Packaging with Terraform

Terraform’s `archive_file` data source is used to zip the Lambda code at runtime.  
Unnecessary files such as `*.jpg` test files are excluded to reduce package size.
![alt text](./images/image-compression-images/image6.png)

---

## Lambda Layer (Pillow)

Pillow dependencies must be compiled in a Linux environment to be compatible with AWS Lambda.
![alt text](./images/image-compression-images/image9.png)

### Build Process

- A Dockerfile installs Pillow in a Lambda-compatible directory structure.
- The directory is zipped into `pillow-layer.zip`.
- Terraform deploys this as a Lambda layer.

![alt text](./images/image-compression-images/image10.png)

### Docker Commands Used

```bash
docker build -t pillow-builder .
docker create --name pillow-temp pillow-builder
docker cp pillow-temp:/tmp/pillow-layer.zip .
docker rm pillow-temp
```

---

## Lambda Function Configuration

- Function name: `image-compressor`
- Runtime: Python 3.10
- Handler: `index.handler`
- Execution role: IAM role created earlier
- Environment variables:

  - `DEST_BUCKET`
  - `FAILED_BUCKET`
  - `SNS_TOPIC_ARN`
    ![alt text](./images/image-compression-images/image11.png)

---

## Lambda Permissions

An `aws_lambda_permission` resource allows S3 to invoke the Lambda function via event notifications.
![alt text](./images/image-compression-images/image12.png)

---

## SNS Configuration

- SNS topic created for notifications
- Admin email subscribed via Terraform
- Email value passed using `terraform.tfvars`
  ![alt text](./images/image-compression-images/image13.png)

---

## Lambda Code

```python
import boto3
from PIL import Image
import io
import os

s3 = boto3.client("s3")
sns = boto3.client("sns")

def compress_image(image_bytes):
    image = Image.open(io.BytesIO(image_bytes))
    buffer = io.BytesIO()
    image.save(buffer, format="JPEG", quality=20)
    return buffer.getvalue()

def publish_notification(status, key, error_message=None):
    dest_bucket = os.environ.get("DEST_BUCKET")
    sns_topic_arn = os.environ.get("SNS_TOPIC_ARN")

    if status == "success":
        message = f"Image compression successful!\nFile: {key}\nDestination: {dest_bucket}"
    else:
        message = f"Image compression failed!\nFile: {key}\nError: {error_message}"

    sns.publish(
        TopicArn=sns_topic_arn,
        Subject=f"Image Compression {status.upper()}",
        Message=message
    )

def handler(event, context):
    dest_bucket = os.environ.get("DEST_BUCKET")
    failed_bucket = os.environ.get("FAILED_BUCKET")

    for record in event["Records"]:
        src_bucket = record["s3"]["bucket"]["name"]
        key = record["s3"]["object"]["key"]

        try:
            obj = s3.get_object(Bucket=src_bucket, Key=key)
            compressed = compress_image(obj["Body"].read())

            s3.put_object(
                Bucket=dest_bucket,
                Key=key,
                Body=compressed,
                ContentType="image/jpeg"
            )

            publish_notification("success", key)

        except Exception as e:
            publish_notification("failed", key, str(e))
            obj = s3.get_object(Bucket=src_bucket, Key=key)
            s3.put_object(
                Bucket=failed_bucket,
                Key=key,
                Body=obj["Body"].read(),
                ContentType="image/jpeg"
            )
```

---

## Demo Results

- Terraform successfully provisions all resources
  ![alt text](./images/image-compression-images/image14.png)
  ![alt text](./images/image-compression-images/image15.png)
- Image upload triggers compression
  ![alt text](./images/image-compression-images/image16.png)
- Average processing time: ~5 seconds
  ![alt text](./images/image-compression-images/image17.png)
- Email notifications received
  ![alt text](image18
  .png)
- Logs visible in CloudWatch
  ![alt text](./images/image-compression-images/image19.png)

---

## Key Takeaways

This project strengthened my understanding of:

- Terraform and Infrastructure as Code
- AWS Lambda internals
- IAM security design
- Docker for dependency packaging
- Event-driven architectures
- Debugging with CloudWatch
