---
layout: post
title: "Structured Logging in Python with Only the Standard Library"
date: 2026-10-15 10:20:00 +0530
description: "Emit JSON logs with Python's standard logging module: a custom formatter, request IDs via contextvars, handler filters, and a pytest check."
---

Plain-text logs are fine until you need to answer a question like "show me every error for request `a1b2` in the last hour." At that point you want logs as data: one JSON object per line, with consistent field names you can filter on.

This post builds that with Python's standard `logging` module. No third-party packages are needed.

![Structured logging pipeline](/assets/python-structured-logging-pipeline.png)

## 1. A JSON formatter

```python
import json
import logging
from datetime import datetime, timezone

# Attributes every LogRecord has; anything else came from `extra=`.
_RESERVED = set(vars(logging.makeLogRecord({}))) | {"message", "asctime"}

class JsonFormatter(logging.Formatter):
    def format(self, record: logging.LogRecord) -> str:
        payload = {
            "ts": datetime.fromtimestamp(record.created, tz=timezone.utc).isoformat(),
            "level": record.levelname,
            "logger": record.name,
            "msg": record.getMessage(),
        }
        # Copy custom fields passed via extra={...}
        for key, value in record.__dict__.items():
            if key not in _RESERVED and not key.startswith("_"):
                payload[key] = value
        if record.exc_info:
            payload["exc"] = self.formatException(record.exc_info)
        return json.dumps(payload, default=str)
```

`default=str` keeps a stray `datetime` or `UUID` from crashing your logging call.

## 2. Request context without passing it everywhere

`contextvars` works with threads and with `asyncio`, so a filter can attach the current request ID to every record:

```python
import contextvars
import uuid

request_id_var = contextvars.ContextVar("request_id", default="-")

class RequestIdFilter(logging.Filter):
    def filter(self, record: logging.LogRecord) -> bool:
        record.request_id = request_id_var.get()
        return True
```

Set it once at the edge of your app, for example in middleware:

```python
def handle_request(path: str) -> None:
    token = request_id_var.set(uuid.uuid4().hex[:8])
    try:
        log.info("request started", extra={"path": path})
        ...
    finally:
        request_id_var.reset(token)
```

## 3. Wiring it up

```python
import sys

def configure_logging(level: str = "INFO") -> None:
    handler = logging.StreamHandler(sys.stdout)
    handler.setFormatter(JsonFormatter())
    handler.addFilter(RequestIdFilter())
    root = logging.getLogger()
    root.handlers[:] = [handler]
    root.setLevel(level)

configure_logging()
log = logging.getLogger("orders")

handle_request("/orders/42")
try:
    1 / 0
except ZeroDivisionError:
    log.exception("failed to compute total", extra={"order_id": 42})
```

Output (one line per event, wrapped here for readability):

```json
{"ts": "2026-10-18T06:12:01.123456+00:00", "level": "INFO", "logger": "orders",
 "msg": "request started", "path": "/orders/42", "request_id": "3f9c1a7e"}
```

Put the filter on the handler, not on a logger. Filters on a logger only run for records created by that logger. A handler filter also catches records that propagate up from child loggers.

## 4. Rules that keep logs useful

- **Constant message, variable fields.** Write `log.info("payment captured", extra={"amount_cents": 1299})`, not an f-string. Then you can group on `msg`.
- **Pick field names once.** `user_id` everywhere, never `uid` in one module and `userId` in another.
- **Don't collide with reserved names.** `extra={"name": ...}` or `extra={"message": ...}` raises a `KeyError`, because those attributes already exist on the record.
- **Never log secrets.** Tokens, passwords and full card numbers stay out. Log an ID that lets you look them up instead.
- **Log to stdout in containers.** Let the platform's collector ship the lines. Your app shouldn't manage log files.

## 5. Testing it

```python
def test_json_log_has_request_id(capsys):
    configure_logging()
    request_id_var.set("test123")
    logging.getLogger("t").info("hello", extra={"k": 1})
    line = json.loads(capsys.readouterr().out.strip())
    assert line["request_id"] == "test123"
    assert line["k"] == 1
```

That's about 50 lines for logs you can actually query. If you outgrow it later, libraries such as `structlog` build on the same ideas.

---

## About ByteBridge Talent LLC

ByteBridge Talent LLC is a technology talent and IT staffing company based in Grand Island, Nebraska, founded in February 2025. We provide IT staff augmentation, cloud and DevOps engineering, and software development talent, working with clients on contract, contract-to-hire and direct-hire terms. Our anchor direct client is Medica Health Plans: as a direct vendor, we've placed more than 60 technology professionals there over roughly two years across software, cloud and data roles. Learn more at [bytebridgetalent.com](https://bytebridgetalent.com).
