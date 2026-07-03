# Arquitectura del Sistema - QuackPackage (Ciudad Talca)

## Diagrama UML de Componentes (Historial Unificado E1 + E2 + E3)

Este documento describe la arquitectura de componentes del sistema QuackPackage para la sucursal de Talca. Contiene la consolidación de toda la infraestructura heredada y las nuevas funcionalidades implementadas para cumplir con los requerimientos acumulativos de las entregas 1, 2 y 3.

## Vista General del Sistema (Ecosistema Completo)

```mermaid
graph TB
    subgraph "Cliente"
        USER[Usuario / Navegador]
    end

    subgraph "AWS Frontend (E1)"
        CF[CloudFront CDN]
        S3[S3 Bucket<br/>Sitio estático React]
    end

    subgraph "AWS API Layer (E1)"
        APIGW[API Gateway HTTP / REST<br/>api.sbarriag.com]
    end

    subgraph "AWS EC2 - Docker Compose (Talca)"
        MASTER[Master<br/>Express API]
        CONNECTOR[Connector<br/>RabbitMQ Consumer]
        DB[(PostgreSQL 16)]
    end

    subgraph "AWS Servicios Serverless (E3)"
        SFN[Step Functions<br/>State Machine]
        SQS[SQS Queue<br/>quackpackage-subscriptions]
        LAMBDA[Lambda<br/>subscription-executor]
    end

    subgraph "Infraestructura de Cómputo Asíncrono (E2 RNF01/06)"
        JOB_MASTER[Jobs Master API<br/>Bull / Redis Queue]
        WORKER_LAMBDA[Worker Lambda<br/>Cómputo Dijkstra]
    end

    subgraph "Servicios Externos del Curso"
        AUTH0[Auth0 IdP<br/>OIDC / JWT]
        WEBPAY[Transbank WebPay<br/>Ambiente Integración]
        BROKER[RabbitMQ Broker<br/>broker.iic2173.org]
        AUDITOR[Central / Auditor<br/>Cola de Auditoría E1/E2]
        NEWRELIC[New Relic<br/>APM y logs]
    end

    subgraph "Otras Ciudades"
        CITIES[15 ciudades hermanas<br/>via broker central]
    end

    %% Flujos de Frontend e Identidad (E1)
    USER -->|HTTPS| CF
    CF -->|origin| S3
    USER -->|OAuth login| AUTH0
    APIGW -.->|RNF07: Authorizer| AUTH0
    USER -->|HTTPS + JWT| APIGW
    APIGW -->|HTTP Proxy| MASTER
    
    %% Base de Datos y Consumo Mensajería (E1)
    MASTER --> DB
    CONNECTOR --> DB
    CONNECTOR <-->|AMQP over TLS| BROKER
    BROKER <--> CITIES
    
    %% Auditoría (E1 RF03 & E2 Requisitos)
    MASTER -->|Manda mensajes de Auditoría| AUDITOR
    CONNECTOR -->|Manda mensajes de Auditoría| AUDITOR
    
    %% Flujo de Pagos (E2)
    MASTER -->|createTransaction / commitTransaction| WEBPAY
    USER -->|redirect pago| WEBPAY
    
    %% Arquitectura de Workers Asíncronos (E2 RNF01)
    MASTER -->|POST /job Cálculo de Rutas| JOB_MASTER
    JOB_MASTER -->|Invoca Tarea Pesada| WORKER_LAMBDA
    WORKER_LAMBDA -->|Retorna Matriz Ruteo / Dijkstra| JOB_MASTER
    MASTER -->|Poll GET /job/:id| JOB_MASTER

    %% Orquestación de Suscripciones (E3 RF01)
    MASTER -->|StartExecution| SFN
    SFN -->|SendMessage| SQS
    SQS -->|trigger| LAMBDA
    LAMBDA -->|POST /subscriptions/:id/execute| APIGW
    
    %% Observabilidad
    MASTER -->|métricas| NEWRELIC
    CONNECTOR -->|métricas| NEWRELIC

    classDef aws fill:#FF9900,stroke:#232F3E,color:#fff
    classDef external fill:#e0e7ff,stroke:#4338ca
    classDef ec2 fill:#84cc16,stroke:#365314,color:#fff
    classDef db fill:#0891b2,stroke:#164e63,color:#fff
    classDef workers fill:#7c3aed,stroke:#5b21b6,color:#fff
    
    class CF,S3,APIGW,SFN,SQS,LAMBDA,WORKER_LAMBDA aws
    class AUTH0,WEBPAY,BROKER,AUDITOR,NEWRELIC,CITIES external
    class MASTER,CONNECTOR,JOB_MASTER ec2
    class DB db
    class JOB_MASTER,WORKER_LAMBDA workers