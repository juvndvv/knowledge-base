---
title: "Comunicación entre bounded contexts: context mapping e integración"
topic: ddd
subtopic: diseño-estratégico
tags: [ddd, bounded-context, context-map, context-mapping, upstream, downstream, partnership, shared-kernel, customer-supplier, conformist, anticorruption-layer, open-host-service, published-language, separate-ways, big-ball-of-mud, integracion, rest, rpc, mensajeria, evento-de-integracion, evento-de-dominio, outbox, consistencia-eventual]
aliases: [integración entre contextos, bounded context integration, context mapping patterns, ACL, OHS, capa anticorrupción, servicio de host abierto, lenguaje publicado, integration events, domain events, transactional outbox, eventual consistency, sync vs async]
sources:
  - Evans, "Domain-Driven Design Reference" (2015) — https://domainlanguage.com/wp-content/uploads/2016/05/DDD_Reference_2015-03.pdf
  - Fowler, bliki "BoundedContext" — https://martinfowler.com/bliki/BoundedContext.html
  - DevIQ, "Context Mapping" — https://deviq.com/domain-driven-design/context-mapping
  - Microsoft, "Communication in a microservice architecture" — https://learn.microsoft.com/en-us/dotnet/architecture/microservices/architect-microservice-container-applications/communication-in-microservice-architecture
  - Microsoft, "Domain events: design and implementation" — https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/domain-events-design-implementation
  - Jovanović, "Domain Events vs Integration Events" — https://milanjovanovic.tech/blog/domain-events-vs-integration-events
  - Richardson, "Pattern: Transactional outbox" — https://microservices.io/patterns/data/transactional-outbox
  - Vernon, "Implementing Domain-Driven Design", cap. 13 (índice) — https://www.oreilly.com/library/view/implementing-domain-driven-design/9780133039900/ch13lev1sec4.html
created: 2026-10-29
---

# Comunicación entre Bounded Contexts en DDD

## Resumen

La comunicación entre bounded contexts (BC) tiene dos dimensiones distintas que conviene no mezclar:

1. **Relación (context mapping):** cómo se relacionan los modelos y los equipos. Evans define estos patrones en la *DDD Reference*: Partnership, Shared Kernel, Customer/Supplier, Conformist, Anticorruption Layer, Open-host Service, Published Language, Separate Ways y Big Ball of Mud.
2. **Mecanismo técnico:** cómo viajan los datos. Puede ser síncrono (REST/RPC) o asíncrono (mensajería con eventos de integración). Vernon dedica el capítulo 13 de *Implementing DDD* a "Integration Using RESTful Resources" e "Integration Using Messaging".

Un mismo patrón de relación (por ejemplo, Open-host Service + Published Language) se puede implementar con REST o con eventos.

## Por qué hace falta: el mismo término, modelos distintos

Fowler: "total unification of the domain model for a large system will not be feasible or cost-effective". Conceptos como *Customer* o *Product* aparecen en varios contextos, cada uno los modela distinto, y hace falta un mapeo entre ellos. Fowler recomienda dibujar estas relaciones en un **context map**. ([Fowler, BoundedContext](https://martinfowler.com/bliki/BoundedContext.html))

Ejemplo: en *Ventas*, `Cliente` tiene límite de crédito y pedidos. En *Envíos*, `Destinatario` solo tiene dirección y teléfono. Son la misma persona, pero distintos modelos.

## Upstream / Downstream

- **Upstream (U):** el proveedor cuyas decisiones afectan a otros (publisher, supplier, API host).
- **Downstream (D):** el consumidor que depende de él (subscriber, customer, consumidor de la API).

Si el publisher cambia el formato del mensaje, impacta a todos los suscriptores. Lo contrario no ocurre. ([DevIQ](https://deviq.com/domain-driven-design/context-mapping))

## Patrones de context mapping

| Patrón | Tipo | Idea clave (Evans, *DDD Reference*) |
|---|---|---|
| **Partnership** | Colaborativo | Si el fallo de uno implica el fallo de ambos, "forge a partnership between the teams". |
| **Shared Kernel** | Colaborativo | Subconjunto del modelo compartido explícitamente. "Keep this kernel small." |
| **Customer/Supplier** | U → D | Relación cliente/proveedor explícita. Se negocian y presupuestan las necesidades del downstream. |
| **Conformist** | U → D | El upstream no tiene motivación para ayudar. El downstream adopta su modelo tal cual para ahorrarse la traducción. |
| **Anticorruption Layer (ACL)** | U → D | El downstream crea una capa aislante que expone la funcionalidad del upstream "in terms of your own domain model". |
| **Open-host Service (OHS)** | U → muchos D | Protocolo abierto como conjunto de servicios para que todos se integren igual. |
| **Published Language (PL)** | U ↔ D | Lenguaje compartido bien documentado como medio de intercambio (JSON Schema, XML, Protobuf, Avro según DevIQ). Suele ir con OHS. |
| **Separate Ways** | Sin relación | Sin relación significativa. Se declara que el BC no se conecta con nadie. |
| **Big Ball of Mud** | Caos | Se pone una frontera alrededor del desorden y no se modela sofisticadamente dentro. |

Fuente: [DDD Reference (Evans, 2015)](https://domainlanguage.com/wp-content/uploads/2016/05/DDD_Reference_2015-03.pdf), [DevIQ](https://deviq.com/domain-driven-design/context-mapping).

### Context map de ejemplo (e-commerce)

```mermaid
flowchart LR
    subgraph Core
        Ventas[Ventas<br/>Core]
        Catalogo[Catálogo]
    end
    Pagos[Pagos<br/>pasarela externa]
    Envios[Envíos]
    Facturacion[Facturación]
    Legacy[ERP legacy<br/>Big Ball of Mud]
    RRHH[RRHH]

    Catalogo -- "U: OHS + PL<br/>D: Ventas" --> Ventas
    Ventas <-- "Partnership" --> Envios
    Pagos -- "U<br/>D: Conformist" --> Ventas
    Ventas -- "U: Customer/Supplier<br/>D: Facturación" --> Facturacion
    Legacy -- "U<br/>D: ACL" --> Facturacion
    Ventas -. "Shared Kernel<br/>Money, Moneda" .- Facturacion
    RRHH ~~~ Ventas
```

*RRHH sigue Separate Ways: no tiene ninguna conexión.*

### Ejemplo: Anticorruption Layer

Facturación consume clientes de un ERP legacy cuyo modelo no quiere importar.

```mermaid
flowchart LR
    subgraph Facturación
        Dom[Dominio<br/>Cliente, Factura]
        ACL[ACL<br/>Facade + Adapter + Translator]
    end
    ERP[(ERP legacy<br/>KUNNR, NAME1, STCD1...)]
    Dom -->|"ClienteRepository.porId(id)"| ACL
    ACL -->|"llamada SOAP/REST"| ERP
```

```ts
// Puerto definido en el dominio de Facturación
interface ClienteFacturable { id: ClienteId; razonSocial: string; nif: Nif; }
interface ClientesFacturables { porId(id: ClienteId): Promise<ClienteFacturable>; }

// ACL: traduce el modelo del ERP al lenguaje de Facturación
class ClientesFacturablesDesdeErp implements ClientesFacturables {
  constructor(private erp: ErpClient) {}
  async porId(id: ClienteId) {
    const raw = await this.erp.getCustomer(id.value); // { KUNNR, NAME1, STCD1 }
    return { id, razonSocial: raw.NAME1.trim(), nif: Nif.parse(raw.STCD1) };
  }
}
```

El dominio de Facturación nunca ve `KUNNR` ni `NAME1`. Si cambia el ERP, solo cambia la ACL.

## Mecanismos de integración

### Síncrono: REST / RPC

HTTP es síncrono: "The client sends a request and waits for a response". Hay un solo receptor por petición. ([Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/architect-microservice-container-applications/communication-in-microservice-architecture))

```mermaid
sequenceDiagram
    participant V as Ventas
    participant C as Catálogo (OHS)
    V->>C: GET /productos/42
    C-->>V: 200 {sku, nombre, precio} (Published Language)
    Note over V: Si Catálogo cae, Ventas no puede completar la operación
```

Microsoft desaconseja encadenar llamadas síncronas entre servicios: "If possible, never depend on synchronous communication (request/response) between multiple microservices, not even for queries". Llama a la cadena de peticiones un anti-patrón porque la arquitectura "won't be resilient when some microservices fail". La alternativa es replicar solo los atributos necesarios mediante eventos de integración y consistencia eventual.

### Asíncrono: mensajería con eventos de integración

Con AMQP y similares, el emisor "usually doesn't wait for a response". La comunicación con varios receptores (pub/sub) "must be asynchronous". ([Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/architect-microservice-container-applications/communication-in-microservice-architecture))

**Evento de dominio vs. evento de integración** ([Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/domain-events-design-implementation), [Milan Jovanović](https://milanjovanovic.tech/blog/domain-events-vs-integration-events)):

| | Evento de dominio | Evento de integración |
|---|---|---|
| Alcance | Dentro del BC | Cruza BCs |
| Transporte | En memoria (p. ej. MediatR) | Broker (RabbitMQ, Azure Service Bus, SQS...) |
| Momento | Antes o durante el commit | Solo tras persistir con éxito |
| Sincronía | Síncrono o asíncrono | "should always be asynchronous" |
| Contrato | Interno, puede usar tipos del dominio | Público: mínimo, tipos primitivos, versionado |

Un handler de un evento de dominio puede generar el evento de integración. Para no perderlo si el proceso cae entre el commit y la publicación, se usa el **Transactional Outbox**: el mensaje se guarda en una tabla outbox dentro de la misma transacción, y un relay lo publica después en orden. ([microservices.io](https://microservices.io/patterns/data/transactional-outbox))

```mermaid
sequenceDiagram
    participant API as Ventas: Application Service
    participant Agg as Pedido (Aggregate)
    participant DB as BD Ventas (+ outbox)
    participant Relay as Outbox Relay
    participant Bus as Broker
    participant Env as Envíos
    participant Fac as Facturación

    API->>Agg: confirmar()
    Agg-->>API: PedidoConfirmado (evento de dominio)
    API->>DB: TX: guarda Pedido + PedidoConfirmadoIntegrationEvent en outbox
    Relay->>DB: lee outbox pendiente
    Relay->>Bus: publica ventas.pedido-confirmado.v1
    Bus-->>Env: entrega
    Bus-->>Fac: entrega
    Note over Env,Fac: Consumidores idempotentes + reintentos.<br/>Consistencia eventual.
```

Ejemplo de evento de integración como Published Language:

```json
{
  "type": "ventas.pedido-confirmado",
  "version": 1,
  "eventId": "3f1c...",
  "occurredAt": "2026-10-29T18:30:00Z",
  "data": { "pedidoId": "P-1001", "clienteId": "C-77", "total": { "amount": "59.90", "currency": "EUR" } }
}
```

### Comparativa

| Criterio | Síncrono (REST/RPC) | Asíncrono (eventos) |
|---|---|---|
| Acoplamiento temporal | Alto: ambos deben estar disponibles | Bajo: el broker desacopla |
| Consistencia | Inmediata en la respuesta | Eventual |
| Receptores | Uno | Cero a muchos (pub/sub) |
| Complejidad | Baja al inicio | Outbox, idempotencia, reintentos, compensaciones |
| Encaja con | Consultas puntuales, OHS con API pública | Propagar cambios, réplica de datos, procesos largos |

Microsoft advierte que la consistencia eventual "requires more complex code in order to detect possible inconsistencies... and the need to implement compensatory actions". ([Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/domain-events-design-implementation))

## Reglas prácticas

- El context map describe la **realidad política**, no la ideal. Evans: "Map the existing terrain. Take up transformations later."
- Ante un upstream cuyo modelo no te conviene, la ACL es la defensa. Conformist solo si el modelo upstream es aceptable.
- Shared Kernel es el acoplamiento más íntimo. Mantenlo pequeño.
- Los eventos de integración son contratos públicos: versiónalos y no expongas el modelo de dominio interno.
- Evita cadenas síncronas entre BCs. Replica los datos necesarios mediante eventos.

## Relacionado

- [DDD táctico: building blocks](building-blocks-tacticos.md) (eventos de dominio, agregados)
