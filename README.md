# RabbitMQ (rabbitmq)

RabbitMQ is a widely deployed open source message broker. It supports multiple messaging protocols and can be deployed in distributed and federated configurations to meet high-scale, high-availability requirements.

**APIs.json:** [https://raw.githubusercontent.com/api-evangelist/rabbitmq/refs/heads/main/apis.yml](https://raw.githubusercontent.com/api-evangelist/rabbitmq/refs/heads/main/apis.yml)

## Scope

- **Type:** Index

## Tags

- AMQP
- Distributed Systems
- Event Streaming
- Message Broker
- Messaging
- Queue

## Timestamps

- **Created:** 2024-01-01
- **Modified:** 2026-05-19

## APIs

### RabbitMQ Management HTTP API

HTTP-based API for management and monitoring of RabbitMQ nodes and clusters. Allows you to manage exchanges, queues, bindings, users, virtual hosts, permissions, and more.

- **Human URL:** [https://www.rabbitmq.com/management.html](https://www.rabbitmq.com/management.html)
- **Base URL:** `http://localhost:15672/api`

#### Tags

- HTTP
- Management
- Monitoring
- REST API

#### Properties

- [Documentation](https://rawcdn.githack.com/rabbitmq/rabbitmq-server/v3.12.0/deps/rabbitmq_management/priv/www/api/index.html)
- [OpenAPI](openapi/rabbitmq-management.yml) — [OpenAPI Specification](https://spec.openapis.org/oas/latest.html)
- [Postman Collection](collections/rabbitmq-management.postman_collection.json) — [Postman Collection 2.1](https://schema.getpostman.com/json/collection/v2.1.0/collection.json)
- [Open Collection](collections/rabbitmq-management.opencollection.json) — [Open Collection 1.0](https://schema.opencollection.com/opencollection/v1.0.0.json)
- [Authentication](https://www.rabbitmq.com/management.html#permissions)

### RabbitMQ AMQP Messaging API

AMQP 0-9-1 messaging protocol for producing and consuming messages via exchanges, queues, and bindings with support for multiple exchange types, message acknowledgment, and consumer groups.

- **Human URL:** [https://www.rabbitmq.com/tutorials/amqp-concepts.html](https://www.rabbitmq.com/tutorials/amqp-concepts.html)

#### Tags

- AMQP
- Messaging
- Pub-Sub
- Queues

#### Properties

- [Documentation](https://www.rabbitmq.com/tutorials/amqp-concepts.html)
- [AsyncAPI](asyncapi/rabbitmq-messaging.yml) — [AsyncAPI Specification](https://www.asyncapi.com/docs/reference/specification/latest)
- [JSON Schema](json-schema/rabbitmq-message.json) — [JSON Schema](https://json-schema.org/specification)
- [Postman Collection](collections/rabbitmq-management.postman_collection.json) — [Postman Collection 2.1](https://schema.getpostman.com/json/collection/v2.1.0/collection.json)
- [Open Collection](collections/rabbitmq-management.opencollection.json) — [Open Collection 1.0](https://schema.opencollection.com/opencollection/v1.0.0.json)

## Common Properties

- [LinkedIn](https://www.linkedin.com/company/rabbitmq)
- [GitHub Organization](https://github.com/rabbitmq)
- [Community](https://www.rabbitmq.com/community.html)
- [Support](https://www.rabbitmq.com/contact.html)
- [Blog](https://www.rabbitmq.com/blog/)
- [Website](https://www.rabbitmq.com/)

## Maintainers

**FN:** Kin Lane
**Email:** kin@apievangelist.com
