# Codebase Function Reference

> **Project:** Support Tickets — a serverless AWS app (Lambda + DynamoDB + SQS + S3)
> **Source root:** `src/`

---

## 📁 `src/common/` — Shared utilities

### [`clients.py`](file:///home/kaustubh/Downloads/support-tickets/src/common/clients.py)

| Function | Input | Output | What it does |
|---|---|---|---|
| `table()` | _(none)_ | DynamoDB Table object | Returns a cached boto3 DynamoDB Table client using the table name from config. Created once per Lambda container. |
| `sqs()` | _(none)_ | SQS client object | Returns a cached boto3 SQS client. |
| `s3()` | _(none)_ | S3 client object | Returns a cached boto3 S3 client, configured with SigV4 so presigned URLs work correctly in every AWS region. |
| `clear_caches()` | _(none)_ | `None` | Wipes all three client caches — used in tests so each test starts fresh. |

---

### [`config.py`](file:///home/kaustubh/Downloads/support-tickets/src/common/config.py)

All functions here read a value from an **environment variable** (set by `template.yaml`) and return it. No inputs; they just expose config.

| Function | Env var read | Returns | What it does |
|---|---|---|---|
| `table_name()` | `TABLE_NAME` | `str` | Name of the DynamoDB table |
| `queue_url()` | `REPORT_QUEUE_URL` | `str` | URL of the SQS report queue |
| `bucket_name()` | `REPORT_BUCKET` | `str` | Name of the S3 bucket for CSV files |
| `presign_ttl_seconds()` | `PRESIGN_TTL_SECONDS` | `int` | How long a presigned download URL is valid (default: 300 s) |
| `max_receive_count()` | `MAX_RECEIVE_COUNT` | `int` | Max times SQS delivers a failing job before it goes to the DLQ (default: 3) |

---

### [`errors.py`](file:///home/kaustubh/Downloads/support-tickets/src/common/errors.py)

These are **exception classes**, not functions. Each one is a domain error with a built-in HTTP status code.

| Class | HTTP Status | Error code string | When raised |
|---|---|---|---|
| `AppError` | 500 | `INTERNAL_ERROR` | Base class — generic server error |
| `ValidationError` | 400 | `VALIDATION_ERROR` | Bad input from the client |
| `NotFoundError` | 404 | `NOT_FOUND` | Ticket or report doesn't exist |
| `UnauthorizedError` | 401 | `UNAUTHORIZED` | Missing or invalid JWT identity |
| `ServiceUnavailableError` | 503 | `SERVICE_UNAVAILABLE` | Downstream failure (e.g. SQS unreachable) |

All accept `(message, details=None)` when constructed.

---

### [`http.py`](file:///home/kaustubh/Downloads/support-tickets/src/common/http.py)

| Function | Input | Output | What it does |
|---|---|---|---|
| `response(status, body, request_id)` | HTTP status int, optional body dict, optional request ID string | API Gateway response dict | Builds the standard `{statusCode, headers, body}` dict that API Gateway expects. Serialises the body to JSON. |
| `error_response(err, request_id)` | An `AppError` exception, request ID string | API Gateway response dict | Converts any `AppError` into the standard error JSON format (`{error: {code, message, requestId}}`). |
| `request_id(event)` | API Gateway event dict | `str` | Pulls the unique request ID out of the event's `requestContext`. Returns `"unknown"` if missing. |
| `user_id(event)` | API Gateway event dict | `str` | Extracts the user's `sub` (UUID) from the JWT claims validated by API Gateway. Raises `UnauthorizedError` if the claim is missing. |
| `json_body(event)` | API Gateway event dict | `dict` | Decodes and parses the request body JSON. Raises `ValidationError` if the body is missing, not valid JSON, or not a JSON object. Handles base64-encoded bodies automatically. |
| `dispatch(event, routes)` | API Gateway event dict, dict mapping route strings to handler functions | API Gateway response dict | Finds the right handler for the incoming `routeKey` (e.g. `"POST /tickets"`), calls it, and catches any `AppError` or unexpected exception, returning a clean error response instead of crashing. |

---

### [`logging_utils.py`](file:///home/kaustubh/Downloads/support-tickets/src/common/logging_utils.py)

| Function / Class | Input | Output | What it does |
|---|---|---|---|
| `JsonFormatter.format(record)` | A `logging.LogRecord` | JSON string | Converts a log record into a single-line JSON object with `timestamp`, `level`, `message`, `logger`, and any extra fields. Needed so CloudWatch Logs Insights can query structured fields. |
| `get_logger(name)` | Optional logger name (default `"app"`) | `logging.Logger` | Returns a logger configured with `JsonFormatter`. Safe to call many times — it only attaches the handler once. |

---

## 📁 `src/handlers/` — Lambda entry points (HTTP + SQS)

### [`tickets.py`](file:///home/kaustubh/Downloads/support-tickets/src/handlers/tickets.py)

These are thin HTTP adapters — each one parses the API Gateway event and delegates to `ticket_service`.

| Function | Input | Output | What it does |
|---|---|---|---|
| `create(event, rid)` | API Gateway event, request ID | HTTP 201 response | Reads the authenticated user + JSON body, calls service to create a ticket, returns the new ticket. |
| `get(event, rid)` | API Gateway event, request ID | HTTP 200 response | Reads user + `ticketId` from path, fetches that ticket, returns it. |
| `list_(event, rid)` | API Gateway event, request ID | HTTP 200 response | Reads user + optional `status`, `limit`, `nextToken` query params, returns a paginated list of tickets. |
| `patch(event, rid)` | API Gateway event, request ID | HTTP 200 response | Reads user + `ticketId` + JSON body with fields to change, updates and returns the ticket. |
| `delete(event, rid)` | API Gateway event, request ID | HTTP 204 response | Reads user + `ticketId`, deletes the ticket, returns empty body. |
| `lambda_handler(event, context)` | Lambda event + context | API Gateway response | The real Lambda entry point. Dispatches to the right function above based on route. |

---

### [`reports.py`](file:///home/kaustubh/Downloads/support-tickets/src/handlers/reports.py)

| Function | Input | Output | What it does |
|---|---|---|---|
| `create(event, rid)` | API Gateway event, request ID | HTTP 202 response | Reads user + optional JSON body (which may contain a `status` filter), kicks off async report generation, returns the new report record immediately. |
| `get(event, rid)` | API Gateway event, request ID | HTTP 200 response | Reads user + `reportId`, fetches the report status (and a download URL if it's `COMPLETED`). |
| `lambda_handler(event, context)` | Lambda event + context | API Gateway response | Lambda entry point — dispatches to `create` or `get`. |

---

### [`report_worker.py`](file:///home/kaustubh/Downloads/support-tickets/src/handlers/report_worker.py)

| Function | Input | Output | What it does |
|---|---|---|---|
| `lambda_handler(event, context)` | SQS trigger event (batch of messages) | `{batchItemFailures: [...]}` | Processes each SQS message (a report job). For each one, calls `report_service.process_job`. If a message fails it's added to `batchItemFailures` so only that message is retried, not the whole batch. |

---

## 📁 `src/services/` — Business logic (no HTTP, no boto3)

### [`ticket_service.py`](file:///home/kaustubh/Downloads/support-tickets/src/services/ticket_service.py)

| Function | Input | Output | What it does |
|---|---|---|---|
| `now_utc()` | _(none)_ | ISO-8601 UTC string | Returns the current time formatted like `2024-01-01T12:00:00.000Z`. Used as the timestamp on all records. |
| `new_ticket_id()` | _(none)_ | `str` | Generates a unique ticket ID: millisecond timestamp (hex) + random bytes. Naturally sorts newest-first. |
| `_text(data, field, max_len, required)` | Input dict, field name, max character length, whether the field is required | `str` or `None` | Validates a text field — checks it's present (if required), is a non-empty string, and is within the length limit. Raises `ValidationError` on failure. |
| `_enum(data, field, allowed, default)` | Input dict, field name, tuple of valid values, optional default | value or `None` | Validates an enum-style field — checks the value is one of the allowed options. Raises `ValidationError` if not. Returns the default if field is absent. |
| `validate_status_filter(status)` | Optional status string | Same optional status string | Checks the status is one of `OPEN`, `IN_PROGRESS`, `CLOSED`. Raises `ValidationError` if not. Returns it unchanged if valid. |
| `create_ticket(user_id, data)` | User ID string, request body dict | New ticket dict | Validates all fields, builds a complete ticket record (with generated ID + timestamps), saves it to DynamoDB, returns the ticket. |
| `get_ticket(user_id, ticket_id)` | User ID string, ticket ID string | Ticket dict | Fetches a single ticket from DynamoDB. Only returns tickets owned by that user. |
| `update_ticket(user_id, ticket_id, data)` | User ID string, ticket ID string, dict of fields to update | Updated ticket dict | Validates only allowed fields are being changed, applies them in DynamoDB, returns the full updated ticket. |
| `delete_ticket(user_id, ticket_id)` | User ID string, ticket ID string | `None` | Deletes a ticket from DynamoDB. Raises `NotFoundError` if it doesn't exist. |
| `list_tickets(user_id, status, limit_raw, token)` | User ID string, optional status filter string, optional limit string, optional pagination token string | `{items: [...], nextToken?: ...}` | Validates and parses query params, fetches one page of tickets from DynamoDB (newest-first), returns items and an optional token for the next page. |

---

### [`report_service.py`](file:///home/kaustubh/Downloads/support-tickets/src/services/report_service.py)

| Function | Input | Output | What it does |
|---|---|---|---|
| `request_report(user_id, data, request_id)` | User ID string, optional body dict (`{status?: ...}`), request ID string | Public report dict (status = `QUEUED`) | Creates a report record in DynamoDB (`QUEUED`), then sends a message to SQS to trigger the worker. If SQS fails, marks the report `FAILED`. Returns the report immediately (async processing happens later). |
| `get_report(user_id, report_id)` | User ID string, report ID string | Report dict (with `downloadUrl` if `COMPLETED`) | Fetches the report from DynamoDB. If it's completed, also generates a presigned S3 URL for the CSV download and includes it in the response. |
| `_csv_cell(value)` | Any value | Safe `str` | Sanitises a single CSV cell — prepends `'` to values starting with `=`, `+`, `-`, `@` to prevent spreadsheet formula injection. |
| `build_csv(rows)` | Iterable of ticket dicts | `(bytes, int)` — the CSV content + row count | Writes all tickets into CSV format (with headers), sanitises each cell, and returns the raw bytes ready to upload to S3. |
| `process_job(job, receive_count)` | Job dict `{reportId, userId, statusFilter}`, SQS receive count int | `str` outcome (`"completed"`, `"duplicate"`, `"dropped"`) | The worker's core logic: fetches tickets, builds the CSV, uploads to S3, and marks the report `COMPLETED`. Skips already-completed reports (idempotent). On final failure, marks it `FAILED`. |

---

## 📁 `src/repositories/` — Database / storage access

### [`keys.py`](file:///home/kaustubh/Downloads/support-tickets/src/repositories/keys.py)

These build the DynamoDB partition/sort key strings. All inputs are IDs, all outputs are formatted key strings.

| Function | Input | Output | Example output |
|---|---|---|---|
| `user_pk(user_id)` | User ID | `str` | `"USER#abc123"` |
| `ticket_sk(ticket_id)` | Ticket ID | `str` | `"TICKET#00abc..."` |
| `report_sk(report_id)` | Report ID | `str` | `"REPORT#uuid..."` |
| `ticket_status_gsi_pk(user_id, status)` | User ID, status string | `str` | `"USER#abc123#STATUS#OPEN"` |
| `report_status_gsi_pk(status)` | Status string | `str` | `"REPORT_STATUS#QUEUED"` |

---

### [`ticket_repository.py`](file:///home/kaustubh/Downloads/support-tickets/src/repositories/ticket_repository.py)

| Function | Input | Output | What it does |
|---|---|---|---|
| `_public(item)` | Raw DynamoDB item dict | Cleaned ticket dict | Strips internal DynamoDB keys, returning only the public-facing fields. |
| `encode_token(last_key)` | DynamoDB `LastEvaluatedKey` dict (or `None`) | Base64 URL-safe string (or `None`) | Converts DynamoDB's pagination cursor into a client-safe token. |
| `decode_token(token, user_id)` | Pagination token string, user ID string | DynamoDB `ExclusiveStartKey` dict | Decodes and validates the pagination token. Raises `ValidationError` if the token is invalid or belongs to a different user. |
| `put(ticket, user_id)` | Ticket dict, user ID string | `None` | Saves a new ticket to DynamoDB. Fails if the item already exists (prevents duplicate writes). |
| `get(user_id, ticket_id)` | User ID string, ticket ID string | Ticket dict | Fetches a ticket by ID. Raises `NotFoundError` if it doesn't exist. |
| `update(user_id, ticket_id, fields, updated_at)` | User ID, ticket ID, dict of changed fields, timestamp string | Updated ticket dict | Updates specified fields in DynamoDB. Also keeps the GSI index in sync when `status` changes. Raises `NotFoundError` if ticket is gone. |
| `delete(user_id, ticket_id)` | User ID string, ticket ID string | `None` | Deletes the ticket. Raises `NotFoundError` if it doesn't exist. |
| `list_page(user_id, status, limit, token)` | User ID, optional status filter, max count int, optional pagination token | `([tickets], next_token)` | Queries DynamoDB for one page of tickets, newest-first. Uses the GSI if filtering by status. Returns items + an opaque token for the next page (or `None` if last page). |
| `iter_all(user_id, status)` | User ID string, optional status filter string | Generator of ticket dicts | Yields every ticket for a user by repeatedly calling `list_page` in batches of 100. Used by the report worker to stream all tickets without loading them all into memory. |

---

### [`report_repository.py`](file:///home/kaustubh/Downloads/support-tickets/src/repositories/report_repository.py)

| Function | Input | Output | What it does |
|---|---|---|---|
| `_public(item)` | Raw DynamoDB item dict | Cleaned report dict | Strips internal DynamoDB fields, returning only the public-facing report fields. |
| `s3_key(user_id, report_id)` | User ID string, report ID string | S3 key string | Builds the S3 object path for a report's CSV: `reports/{userId}/{reportId}.csv`. Always the same for a given report — makes writes idempotent. |
| `create(user_id, report_id, status_filter, now)` | User ID, report ID, optional status filter string, timestamp string | Raw DynamoDB item dict | Creates a new report record with status `QUEUED` in DynamoDB. Returns the full item. |
| `get(user_id, report_id)` | User ID string, report ID string | Raw DynamoDB item dict | Fetches a raw report item. Scoped to the user's partition key (one user cannot see another's reports). Raises `NotFoundError` if not found. |
| `public(item)` | Raw DynamoDB item dict | Cleaned report dict | Exposes the internal `_public` helper. |
| `set_status(user_id, report_id, status, now, extra, not_completed)` | User ID, report ID, new status string, timestamp, optional extra fields dict, safety flag | `bool` — `True` if updated, `False` if already `COMPLETED` | Atomically updates a report's status. **Never overwrites `COMPLETED`** (safe for concurrent workers). Manages the GSI index: adds the report to the index for `QUEUED/PROCESSING/FAILED`, removes it when `COMPLETED`. |
| `list_by_status(status, created_before)` | Status string, ISO timestamp string | List of raw DynamoDB items | Recovery helper — finds all reports in a given status that were created before a certain time. Uses the GSI so no expensive Scan is needed. |
| `publish_job(user_id, report_id, status_filter)` | User ID string, report ID string, optional status filter string | `None` | Sends a message to the SQS report queue with the job details. This triggers the worker Lambda. |
| `put_csv(user_id, report_id, data)` | User ID string, report ID string, CSV bytes | S3 key string | Uploads the finished CSV to S3 with server-side encryption. Always overwrites the same key (idempotent). |
| `presign(user_id, report_id)` | User ID string, report ID string | Presigned URL string | Generates a temporary S3 download URL for the report CSV. The URL is only valid for `presign_ttl_seconds()`. |
