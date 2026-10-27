# AWS PDF Compression Pipeline

An event-driven serverless pipeline that automatically compresses PDF
files uploaded to Amazon S3. An S3 upload event triggers an AWS Lambda
function, which downloads the original PDF, compresses it with PyMuPDF
(`fitz`), and uploads the compressed file to a destination S3 bucket.
Amazon CloudWatch provides logs and operational metrics.

> **Project status:** This README describes the architecture shown in
> the project diagram. Update the resource names, deployment commands,
> and configuration values to match your actual AWS implementation
> before publishing.

## Architecture

![AWS PDF Compression Pipeline Architecture](architecture.png)

**Processing flow**

1.  A user uploads a PDF to the **source S3 bucket**.
2.  An S3 `ObjectCreated` event invokes the **PDF compressor Lambda
    function**.
3.  Lambda downloads the source PDF to temporary storage (`/tmp`).
4.  PyMuPDF processes the document and saves a compressed PDF.
5.  Lambda uploads the result to the **destination S3 bucket**.
6.  Amazon CloudWatch collects execution logs, errors, duration, and
    other metrics.
7.  Users can download the compressed PDF from the destination bucket or
    access it through an optional application or CloudFront
    distribution.

## Features

-   Automatic processing when a PDF is uploaded to S3.
-   Serverless processing with AWS Lambda.
-   PDF optimization using PyMuPDF (`fitz`).
-   Separate source and destination buckets to preserve the original
    file.
-   Execution logging and troubleshooting through CloudWatch.
-   Optional delivery through CloudFront or a web application.
-   Pay-per-use architecture; costs depend on requests, storage, data
    transfer, and execution.

## AWS services and libraries

  -----------------------------------------------------------------------
  Component                           Purpose
  ----------------------------------- -----------------------------------
  Amazon S3 --- source bucket         Stores original PDF files and emits
                                      upload events.

  AWS Lambda                          Runs the compression code in
                                      response to S3 events.

  Lambda Layer (optional)             Packages PyMuPDF and its compatible
                                      dependencies if they are not
                                      included in the deployment package.

  Amazon S3 --- destination bucket    Stores compressed PDFs.

  Amazon CloudWatch                   Stores Lambda logs and exposes
                                      metrics such as invocations,
                                      errors, and duration.

  Amazon CloudFront (optional)        Distributes files to users through
                                      a CDN.

  PyMuPDF (`fitz`)                    Reads and rewrites PDFs with
                                      compression/optimization options.
  -----------------------------------------------------------------------

## Repository layout

A possible repository layout is:

``` text
aws-pdf-compression-pipeline/
├── lambda_function.py
├── requirements.txt
├── architecture.png
└── README.md
```

Your actual file names may differ. Keep `architecture.png` in the
repository root if you want the diagram above to render on GitHub.

## How to configure it

### 1. Create the S3 buckets

Create a source bucket for uploaded PDFs and a destination bucket for
compressed files. Choose unique bucket names and a suitable AWS Region.

Recommended safeguards:

-   Enable **Block Public Access** on both buckets.
-   Enable encryption at rest.
-   Consider versioning and lifecycle rules if you need recovery or
    storage-cost controls.
-   Do not use the same upload prefix for both source and output files
    unless your event configuration prevents recursive invocations.

### 2. Configure the Lambda function

Create a Lambda function using a supported Python runtime that is
compatible with the PyMuPDF package you deploy.

Configure environment variables such as:

``` text
DESTINATION_BUCKET=your-compressed-pdf-bucket
OUTPUT_PREFIX=compressed/
```

If your code uses different variable names, update these examples to
match it.

Set the Lambda timeout and memory based on the size and complexity of
the PDFs you expect to process. Lambda's writable `/tmp` storage is
temporary and has a configurable size limit, so check that your input
and output PDFs fit within the available storage.

### 3. Package PyMuPDF

Install or package a PyMuPDF version compatible with the Lambda Python
runtime and execution environment. You can bundle dependencies in the
deployment package or use a Lambda Layer.

Example `requirements.txt`:

``` text
PyMuPDF
```

Build and test the deployment artifact for the same operating system,
architecture, and Python runtime used by Lambda. Native dependencies may
not work if built for an incompatible local environment.

### 4. Grant least-privilege IAM permissions

The Lambda execution role typically needs:

-   `s3:GetObject` for the source bucket and allowed input prefix.
-   `s3:PutObject` for the destination bucket and allowed output prefix.
-   CloudWatch Logs permissions, commonly provided by the
    `AWSLambdaBasicExecutionRole` managed policy.

The S3 bucket's resource policy and Lambda permissions must also allow
the configured event invocation. Avoid broad permissions such as `s3:*`
on all resources.

### 5. Configure the S3 trigger

Add an S3 event notification for object creation to invoke the Lambda
function. If possible, filter events to the intended input prefix and
`.pdf` suffix.

**Important:** Keep generated files in a separate destination bucket or
a separate prefix that does not match the trigger filter. Otherwise, the
output upload can trigger the function again.

### 6. Test the pipeline

1.  Upload a sample PDF to the source bucket.
2.  Open the Lambda function's **Monitor** tab and inspect its
    CloudWatch logs.
3.  Confirm the function completes without errors.
4.  Check the destination bucket for the compressed PDF.
5.  Download and open the output to verify that it is valid and that its
    pages and content are preserved.
6.  Compare input and output file sizes to measure the result.

Compression results vary by document. Some PDFs may shrink
substantially; others may show little improvement or even grow slightly.
Always validate the output rather than assuming every file will become
smaller.

## Operational considerations

-   **Temporary files:** Use unique temporary file names and clean them
    up after processing.
-   **Duplicate events:** S3 notifications are delivered at least once,
    so design the function to handle duplicate invocations safely.
-   **Large PDFs:** Check Lambda memory, timeout, `/tmp` capacity, and
    service quotas. Use a different processing architecture if documents
    exceed Lambda's practical limits.
-   **Errors:** Log the source key and a useful error message without
    logging sensitive document contents.
-   **Retries and failed jobs:** Consider an SQS queue, dead-letter
    queue, or failure destination for workloads that need controlled
    retries and recovery.
-   **Security:** Keep buckets private, use least-privilege IAM
    policies, and use presigned URLs or authenticated access when users
    need downloads.
-   **Monitoring:** Create CloudWatch alarms for errors, throttles, or
    unusual duration where appropriate.
-   **Costs:** Review current AWS pricing for Lambda, S3 storage and
    requests, CloudWatch, and optional CloudFront/data transfer in your
    selected Region.

## Troubleshooting

  -----------------------------------------------------------------------
  Problem                             What to check
  ----------------------------------- -----------------------------------
  Lambda does not run after upload    S3 event notification, Lambda
                                      invocation permission, event
                                      filters, and Region configuration.

  `ModuleNotFoundError: fitz`         PyMuPDF package or Layer contents,
                                      Python version, architecture, and
                                      deployment artifact.

  `AccessDenied`                      Lambda execution-role permissions
                                      and S3 bucket policies.

  Output is missing                   CloudWatch logs, destination bucket
                                      name, output key, and `PutObject`
                                      permission.

  Function times out or runs out of   PDF size, Lambda timeout, memory
  memory                              allocation, and `/tmp` storage.

  Function runs repeatedly            Ensure output objects do not match
                                      the source trigger's prefix/suffix
                                      filters.

  Output is not smaller               Compression depends on the PDF's
                                      existing encoding and content;
                                      validate quality and size for each
                                      document.
  -----------------------------------------------------------------------

## Future improvements

-   Add an SQS queue to buffer uploads and control processing
    concurrency.
-   Add a dead-letter queue and retry monitoring.
-   Record job status and file-size savings in DynamoDB.
-   Add a small authenticated web interface for upload and download.
-   Generate presigned S3 download URLs.
-   Add automated tests and a CI/CD deployment workflow.
-   Publish operational metrics such as processing time and compression
    ratio.

## Learning outcomes

This project demonstrates practical experience with event-driven AWS
architecture, serverless compute, S3 event notifications, IAM
permissions, Python dependency packaging, PDF processing, CloudWatch
monitoring, and cloud troubleshooting.

## License

Choose and add a license before making this repository public. If you do
not intend to grant reuse rights, state that clearly instead of assuming
a license.

