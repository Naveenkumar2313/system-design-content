# CS-019 — Research References and Evidence Register

## Supplied documents

1. Codorythm 250 Case Studies catalogue.
2. System Design Knowledge Library.

## Technical references

- HTTP Semantics, RFC 9110: https://www.rfc-editor.org/rfc/rfc9110
- HTTP Webhooks and callbacks are application-defined; define signatures, timestamp, idempotency, retry, and replay explicitly.
- OWASP SSRF Prevention Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html
- OWASP Authorization Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html
- Apache Kafka: https://kafka.apache.org/documentation/
- RabbitMQ: https://www.rabbitmq.com/docs
- Amazon SQS: https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html
- PostgreSQL: https://www.postgresql.org/docs/
- Redis: https://redis.io/docs/
- TLS 1.3, RFC 8446: https://www.rfc-editor.org/rfc/rfc8446

## Claims requiring verification

- Queue ordering, acknowledgment, visibility timeout, retention, and redelivery behavior.
- Kafka partition/replay and consumer offset semantics.
- RabbitMQ routing, quorum queue, and dead-letter exchange behavior.
- SQS delay, duplicate delivery, visibility timeout, and FIFO constraints.
- Database partition, index, transaction, replication, and retention behavior.
- Circuit-breaker library/provider thresholds and half-open probes.
- HMAC algorithm and canonical signing convention agreed by provider/consumer.
