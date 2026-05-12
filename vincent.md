# Palmery Payment and Infrastructure - Individual (Vincent)

This document captures the architecture deliverable for the Palmery payment domain and the current deployment infrastructure.

## 1. System Context Diagram

```mermaid
flowchart LR
    admin[Admin]
    worker[Buruh]
    foreman[Mandor]
    driver[Supir]
    gateway[External Payment Gateway Sandbox]

    subgraph palmery[ Palmery Platform ]
        fe[Palmery Frontend]
        payment[Palmery Payment Service]
        auth[Palmery Auth Service]
        manage[Palmery Manage Service]
    end

    admin --> fe
    worker --> fe
    foreman --> fe
    driver --> fe

    fe --> auth
    fe --> manage
    fe --> payment

    payment --> gateway
    manage --> payment
    auth --> payment
    payment --> auth
```

### Context Notes

- All four user roles access Palmery through the main frontend.
- `palmery-payment` is responsible for wallet balance, payroll records, notifications, wage configuration, and top-up processing.
- `palmery-payment` integrates with a sandbox payment gateway for admin top-up flows.
- `palmery-auth` and `palmery-manage` exchange data indirectly with payment through API calls and domain events.

## 2. Container Diagram

```mermaid
flowchart LR
    user[Browser Users]
    gateway[Payment Gateway Sandbox]

    subgraph public[Public Edge]
        nginx[nginx Reverse Proxy]
    end

    subgraph app[Application Containers]
        fe[Palmery FE\nNext.js]
        authf[Auth Frontend\nNext.js]
        authb[Auth Backend\nSpring Boot]
        manageb[Manage Backend\nSpring Boot]
        paymentb[Payment Backend\nSpring Boot]
        assets[Assets FS\ndufs]
    end

    subgraph data[Data and Messaging]
        postgres[(PostgreSQL)]
        rabbit[(RabbitMQ)]
    end

    user --> nginx
    nginx --> fe
    nginx --> authf
    nginx --> authb
    nginx --> manageb
    nginx --> paymentb
    nginx --> assets

    fe --> authb
    fe --> manageb
    fe --> paymentb

    paymentb --> postgres
    paymentb --> rabbit
    paymentb --> gateway

    authb --> postgres
    manageb --> postgres
```

### Container Notes

- `nginx` is the only public ingress and routes by domain to the correct container.
- `palmery-payment` persists wallet, payroll, notification, user-directory, and top-up data in PostgreSQL.
- RabbitMQ is used for payment-side asynchronous integration, especially payroll and notification processing after domain events.
- The current infra still uses one shared PostgreSQL instance with separate logical databases.

## 3. Component Diagram for `palmery-payment`

```mermaid
flowchart LR
    client[Frontend or Internal Client]
    broker[(RabbitMQ)]
    db[(PostgreSQL)]
    gateway[Payment Gateway Sandbox]

    subgraph controllers[REST Controllers]
        walletc[WalletController]
        payrollc[PayrollController]
        paymentc[PaymentGatewayController]
        notifc[NotificationController]
        wagec[WageConfigController]
        debugc[DebugController]
    end

    subgraph services[Application Services]
        walletd[WalletDashboardService]
        wallets[WalletService]
        payrolls[PayrollManagementService]
        topups[TopUpService]
        notifs[NotificationService]
        users[UserDirectoryService]
        wages[WageConfigService]
        consumer[PaymentEventConsumer]
        publisher[DomainEventPublisher]
        integration[IntegrationDebugService]
    end

    subgraph repos[Repositories]
        walletr[WalletRepository]
        payrollr[PayrollRepository]
        topupr[PaymentTopUpRepository]
        notifr[NotificationRepository]
        userr[AppUserRepository]
        wager[WageConfigRepository]
        checkr[IntegrationCheckRepository]
    end

    client --> walletc
    client --> payrollc
    client --> paymentc
    client --> notifc
    client --> wagec
    client --> debugc

    walletc --> walletd
    payrollc --> payrolls
    paymentc --> topups
    notifc --> notifs
    wagec --> wages
    debugc --> integration
    debugc --> publisher

    walletd --> walletr
    walletd --> payrollr

    topups --> topupr
    topups --> wallets
    topups --> gateway

    payrolls --> payrollr
    payrolls --> wallets
    payrolls --> wages
    payrolls --> publisher

    notifs --> notifr
    notifs --> users

    users --> userr
    wages --> wager
    wallets --> walletr
    integration --> checkr

    consumer --> broker
    consumer --> users
    consumer --> wallets
    consumer --> notifs
    consumer --> payrolls

    publisher --> broker

    walletr --> db
    payrollr --> db
    topupr --> db
    notifr --> db
    userr --> db
    wager --> db
    checkr --> db
```

### Component Notes

- `PaymentGatewayController` and `TopUpService` implement the admin top-up flow and webhook confirmation.
- `PayrollManagementService` creates payroll drafts, approves or rejects payroll, and emits `PayrollDiproses` domain events.
- `PaymentEventConsumer` listens to RabbitMQ and turns upstream operational events into wallet initialization, payroll creation, and user notifications.
- `NotificationService` and `UserDirectoryService` keep user-targeted notifications inside the payment domain instead of reaching back into auth synchronously for each request.

## 4. Deployment Diagram

```mermaid
flowchart TB
    internet[Internet Users]
    gateway[External Payment Gateway]

    subgraph vm[Single Staging VM]
        subgraph publicnet[Docker Network: palmery-public]
            nginx[nginx container\nPort 80]
        end

        subgraph internalnet[Docker Network: palmery-internal]
            fe[Palmery FE]
            authf[Auth Frontend]
            authb[Auth Backend]
            manageb[Manage Backend]
            paymentb[Payment Backend]
            rabbit[RabbitMQ]
            postgres[PostgreSQL]
            assets[dufs Assets Server]
        end

        volpg[(postgres-data volume)]
        volassets[(assets-data volume)]
    end

    internet --> nginx
    nginx --> fe
    nginx --> authf
    nginx --> authb
    nginx --> manageb
    nginx --> paymentb
    nginx --> assets

    paymentb --> rabbit
    paymentb --> postgres
    authb --> postgres
    manageb --> postgres
    assets --> volassets
    postgres --> volpg

    paymentb --> gateway
```

### Deployment Notes

- The current deployment is a single-VM compose stack, so reliability depends heavily on that host.
- `palmery-public` exposes only `nginx`, all application containers and data services remain on `palmery-internal`.
- PostgreSQL and asset storage use Docker volumes for persistence.
- RabbitMQ is an internal-only integration backbone for the payment workflow.

## 5. Risk Summary

| Risk | Impact | Mitigation |
|---|---|---|
| Shared PostgreSQL instance for multiple services | Resource contention or noisy-neighbor failures across domains | Move toward stronger database isolation and backup strategy per service |
| Single staging VM | One host outage removes the full platform | Replicate environment or separate critical services |
| `nginx` as single ingress point | Misconfiguration or outage blocks all public access | Harden config, add health checks, consider redundant edge deployment |
| Payment webhook handled inside one service instance | Missed confirmations can leave top-ups pending | Add retry-safe webhook handling and monitoring |
| Event consumer coupled to one queue | Queue backlog delays payroll and notifications | Monitor queue depth and scale consumer instances |

## 6. Architecture Justification

The current payment-and-infra architecture is already aligned with Palmery’s most important flows, wallet funding, payroll generation, and internal notifications. The strongest design choice in this scope is the combination of synchronous API endpoints for interactive user actions and asynchronous RabbitMQ consumers for operational events. That split keeps top-up and wallet screens responsive while allowing payroll generation and notification fan-out to run independently of the upstream transaction that produced the event.

The main weakness is infrastructure centralization. `palmery-infra` currently deploys the entire platform on one compose stack with one shared PostgreSQL host and one public reverse proxy. This is acceptable only for staging and milestone delivery, but it becomes the limiting factor for fault isolation and scale. For the next evolution, we will do deployment isolation, observability, and database separation around the existing payment service and whole palmery service.
