# Resilient Serverless File Processing Pipeline (AWS)

An event-driven, serverless backend in **Python** that decouples file upload from processing
to improve scalability and fault tolerance. Failed messages are retried automatically and
isolated in a **Dead Letter Queue (DLQ)** so no data is silently lost. The whole stack is
defined as **Infrastructure-as-Code** with AWS SAM / CloudFormation.

## Architecture

```
            ObjectCreated            batch of 5, partial-batch responses
 Client ──► S3 (uploads/) ──► SQS queue ─────────────────────────────► Lambda ──► DynamoDB
                                 │  after N failed receives                │
                                 ▼                                         ▼
                               DLQ ──► CloudWatch alarm ──► SNS e-mail   CloudWatch Logs
                                 └── redrive back to main queue
```

| Component | Purpose |
|-----------|---------|
| S3 bucket | Receives uploads under `uploads/`; encrypted, public access blocked |
| SQS queue | Buffers events so uploads never wait on processing |
| DLQ | Holds messages that failed `MaxReceiveCount` times (14-day retention) |
| Lambda (`src/processor/app.py`) | Reads the file, extracts metadata, writes result |
| DynamoDB | Stores results; conditional put makes processing idempotent |
| CloudWatch | Alarms on DLQ depth and Lambda errors; 14-day log retention |

## Resilience features

* **Decoupling** – S3 → SQS → Lambda; traffic spikes queue up instead of overwhelming compute.
* **Automatic retries** – SQS redelivers after the visibility timeout (180 s = 6 × function timeout).
* **DLQ isolation** – after 3 failed attempts (`MaxReceiveCount`) a message moves to the DLQ.
* **Partial batch responses** – only failing records are retried (`ReportBatchItemFailures`).
* **Idempotency** – duplicate deliveries never create duplicate rows.
* **Least-privilege IAM** – Lambda can only read the one bucket and write the one table.
* **Observability** – structured JSON logs + alarms (optional e-mail via SNS).

## Prerequisites

* AWS account + configured CLI (`aws configure`)
* [AWS SAM CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html)
* Python 3.12

## Run the unit tests (no AWS needed)

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r tests/requirements-dev.txt
pytest -v
```

## Deploy (single command)

```bash
./scripts/deploy.sh --guided                 # first time
./scripts/deploy.sh                          # later deployments
./scripts/deploy.sh --parameter-overrides AlarmEmail=you@example.com
```

Region defaults to `ap-south-1` in `samconfig.toml`.

## Use it

```bash
source scripts/_common.sh
aws s3 cp mydata.csv s3://$BUCKET/uploads/mydata.csv
aws dynamodb scan --table-name $TABLE
sam logs --stack-name resilient-file-pipeline --tail
```

Supported file types: `.csv` (row/column count), `.json` (validated), anything else (line/word count).
Invalid JSON/CSV is treated as a failure and ends up in the DLQ.

## Test the DLQ (failure design test)

```bash
./scripts/test_dlq.sh
```

Uploads a good file and a `poison_*` file (the latter always fails when
`EnableFailureInjection=true`), waits for the retries to exhaust, and prints the DLQ message.

## Redrive failed messages

```bash
# 1. fix the cause (e.g. redeploy with EnableFailureInjection=false)
./scripts/deploy.sh --parameter-overrides EnableFailureInjection=false
# 2. move messages from DLQ back to the main queue
./scripts/redrive.sh
```

## Local invoke

```bash
sam build
sam local invoke ProcessorFunction -e events/sqs-poison.json
```
(Replace `YOUR_BUCKET` in the event files first.)

## Clean up

```bash
./scripts/cleanup.sh
```

## Project layout

```
template.yaml            SAM/CloudFormation stack (all infrastructure)
samconfig.toml           deploy defaults
src/processor/app.py     Lambda handler
tests/                   pytest + moto unit tests
events/                  sample SQS events for sam local invoke
scripts/                 deploy, DLQ test, redrive, cleanup
```
