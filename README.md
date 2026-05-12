# Palmery Tutorial 9

## Scope

- `palmery-payment`
- `palmery-infra`

## Deliverables

- C4 documentation: [vincent.md](vincent.md)
- Architecture summary and risks for the same scope

## Summary

The current solution centers on `palmery-payment` as the wallet, payroll, notification, and top-up backend, while `palmery-infra` deploys the wider Palmery platform on one Docker Compose stack behind `nginx`. The design already uses synchronous HTTP for user-facing flows and RabbitMQ for asynchronous domain events consumed by the payment service.

The main architectural risks in this scope are shared infrastructure bottlenecks, a single public ingress point, and dependence on one shared PostgreSQL instance. The detailed document includes C4 system context, container, component, and deployment views, plus focused risk notes and improvement recommendations for this payment-and-infra slice.
