# support-tickets

Serverless REST API for managing support tickets with asynchronous CSV report generation.

**Stack:** AWS Lambda · API Gateway (HTTP API) · DynamoDB · SQS · S3 · Cognito · Python 3.12 · SAM

---

## Architecture

```
Client → API Gateway (Cognito JWT) → Lambda
                                      ├── tickets handler    → DynamoDB
                                      └── reports handler    → DynamoDB + SQS
                                                                     ↓
                                                          report-worker Lambda
                                                                     ↓
                                                               S3 (CSV) → presigned URL
```

Reports are created asynchronously. A worker Lambda consumes the SQS queue, writes a CSV to S3, and stores a presigned download URL back in DynamoDB. Failed jobs are moved to a DLQ after 3 retries, and a CloudWatch alarm fires when the DLQ is non-empty.

---

## API

| Method   | Path                    | Description                              |
|----------|-------------------------|------------------------------------------|
| `POST`   | `/tickets`              | Create a ticket                          |
| `GET`    | `/tickets`              | List tickets (`status`, `limit` filters) |
| `GET`    | `/tickets/{ticketId}`   | Get a ticket                             |
| `PATCH`  | `/tickets/{ticketId}`   | Update a ticket                          |
| `DELETE` | `/tickets/{ticketId}`   | Delete a ticket                          |
| `POST`   | `/reports`              | Request a CSV report (async)             |
| `GET`    | `/reports/{reportId}`   | Poll report status / get download URL    |

All endpoints require a Cognito JWT (`Authorization: Bearer <token>`). See [`docs/openapi.yaml`](docs/openapi.yaml) for the full schema.

---

## Local Development

No AWS account needed — uses [moto](https://github.com/getmoto/moto) to mock all AWS services.

```bash
# install dependencies
python -m venv .venv && source .venv/bin/activate
pip install -r requirements-dev.txt

# run local server (http://localhost:8080)
python local_server.py

# simulate a different user
curl -H "X-User-Id: alice" http://localhost:8080/tickets
```

The report worker runs automatically in the background. Stop with `Ctrl-C`.

---

## Tests

```bash
pytest
```

---

## Deploy

Requires the AWS SAM CLI and an AWS profile named `support-tickets-deploy`.

```bash
sam build
sam deploy        # uses samconfig.toml defaults (region: us-east-1)
```

Stack outputs include `ApiUrl`, `UserPoolId`, and `UserPoolClientId`.

To get a Cognito token after deploying:

```bash
./scripts/get_token.sh
```

---

## Docs

| File | Description |
|------|-------------|
| [`docs/openapi.yaml`](docs/openapi.yaml) | OpenAPI 3.0 spec |
| [`docs/DYNAMODB.md`](docs/DYNAMODB.md) | DynamoDB data model |
| [`docs/RELIABILITY.md`](docs/RELIABILITY.md) | Reliability & error-handling notes |
| [`docs/postman_collection.json`](docs/postman_collection.json) | Postman collection |
