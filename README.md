# Palmery - Tutorial 9

### System Context Diagram (C4 Level 1)
![alt text](contextdiagram.png)

### Container Diagram (C4 Level 2)
![alt text](containerdiagram.png)

### Deployment Diagram
![alt text](deploymentdiagram.png)

---

### Risk Storming Summary

| No | Risk | Impact | Mitigation |
|---|------|--------|------------|
| 1 | Synchronous cascading failures between services | System-wide outage | Circuit breakers, async communication |
| 2 | Database contention from shared DB server | Slow queries, timeouts | Database per service, read replicas |
| 3 | Harvest traffic spikes from concurrent worker submissions | Service overwhelmed, timeouts | Message queue buffering, horizontal scaling |
| 4 | Monthly payroll batch processing blocks other operations | System unresponsive | Async event-driven payroll generation |
| 5 | Tight coupling requires coordinated deployments | Slow feature delivery | Event-based communication, independent deployments |
| 6 | No API Gateway for centralized rate limiting or routing | Security gaps, inconsistent auth | Introduce API Gateway |
| 7 | Auth service is a single point of failure for all services | Total platform inaccessibility | JWT local validation, service redundancy |
| 8 | No distributed tracing makes debugging difficult | Slow incident response | Centralized logging, tracing tools |

### Future Context Diagram (After EDA Migration)
![alt text](futurecontextdiagram.png)

### Future Container Diagram (EDA with Message Broker)
![alt text](futurecontainerdiagram.png)

---

## Architecture Modification Justification
### Risk Analysis Notes

The current Palmery architecture relies on synchronous REST communication between three tightly coupled services (palmery-auth, palmery-manage, palmery-payment). This creates cascading failure risks because when one service goes down, dependent services fail as well. Under nationwide scale with thousands of concurrent harvest submissions and monthly payroll processing for all workers, the system would face severe bottlenecks at both the database layer and application layer simultaneously. The lack of an API Gateway also means there is no centralized rate limiting or traffic management.

### Architecture Modification Justification

To address these risks, we propose migrating to an Event-Driven Architecture (EDA) using RabbitMQ or Kafka as a message broker. The key change is replacing synchronous inter-service REST calls with asynchronous domain events. For example, `HarvestApproved` triggers payroll generation and `DeliveryApproved` triggers wallet updates. This provides fault isolation because if Payment Service is temporarily down, events simply queue in the broker and are processed upon recovery rather than causing upstream failures. We also decompose `repo-manage` into separate Plantation, Harvest, and Delivery services with independent databases, enabling each service to scale independently based on its specific load patterns.

The addition of an API Gateway (Spring Cloud Gateway) provides centralized routing, rate limiting, and JWT validation. Combined with the database-per-service pattern, this architecture allows independent team ownership, independent deployment cycles, and horizontal scaling of individual services during peak loads such as harvest season mornings and end-of-month payroll. The event-driven approach also naturally supports future features like real-time dashboards, audit trails, and notification automation without modifying existing services.
