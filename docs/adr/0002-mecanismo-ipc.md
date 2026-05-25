# ADR 0002 — Mecanismo de IPC entre Servicios de FTGO

**Estado:** Accepted  
**Fecha:** 2026-05-25  
**Decisores:** Equipo de Arquitectura FTGO  
**Trazabilidad:** PRD NFR-01 · PRD NFR-03 · PRD NFR-04 · FSD UC-01 UC-04 UC-05 · Brief §A.4 R-02 R-04 R-07 · Richardson Cap 3–4

---

## Contexto

Con la decisión ADR 0001 de adoptar microservicios con DDD Bounded Contexts, FTGO
necesita definir cómo se comunican sus servicios entre sí. El flujo transaccional
principal involucra al menos 5 servicios en cadena: el Consumidor crea un pedido
(UC-01) → Order Service coordina con Billing Service (UC-04) → Kitchen Service recibe
el ticket (UC-02) → Delivery Service asigna el courier (UC-03) → el Consumidor hace
tracking en tiempo real (UC-05). Esta cadena puede ejecutarse de forma síncrona
(cada servicio espera la respuesta del anterior) o asíncrona (cada servicio publica
eventos y reacciona a eventos de otros).

La decisión no puede postergarse: el diseño de los contratos de API, las estructuras
de datos en tránsito y la estrategia de consistencia dependen directamente del
mecanismo de IPC. NFR-01 (latencia ≤ 200 ms p95 para acciones del consumidor) y
NFR-03 (tolerancia a fallos de Stripe: cola de retry sin bloquear Order Taking) son
incompatibles entre sí bajo un modelo 100% síncrono: si Billing Service llama a Stripe
de forma síncrona en el mismo hilo que la confirmación del pedido, un timeout de
Stripe de 5–10 s viola NFR-01 directamente.

Las restricciones R-02 (latencia), R-04 (tolerancia a fallos externos), R-07
(consistencia eventual aceptada) y R-05 (Strangler Fig) condicionan el espacio
de soluciones. R-04 es especialmente restrictivo: descarta cualquier opción que
haga depender la respuesta al consumidor de la disponibilidad de Stripe.

---

## Restricciones Aplicables

| ID | Restricción | Fuente | Impacto en esta decisión |
|----|-------------|--------|--------------------------|
| R-02 | Latencia < 200 ms p95 para acciones del consumidor | Brief §A.4 | Las operaciones de lectura y la respuesta inicial al consumidor deben ser síncronas y rápidas |
| R-04 | Sistema toma pedidos aunque Stripe esté caído | Brief §A.4 | El IPC entre Order Service y Billing Service no puede ser síncrono bloqueante |
| R-05 | Strangler Fig 18–24 meses | Brief §A.4 | El mecanismo IPC debe funcionar entre servicios nuevos y el monolito legacy |
| R-07 | Consistencia eventual aceptada entre servicios | Brief §A.4 | Habilita IPC asíncrono con eventos; fuerte consistencia solo dentro del aggregate |
| R-01 | Tráfico pico 5× | Brief §A.4 | El mecanismo IPC no debe ser cuello de botella en horarios pico |

---

## Opciones Consideradas

### Opción 1: REST/JSON Síncrono como IPC Predominante

**Descripción:** Todos los servicios de FTGO se comunican mediante llamadas HTTP/REST
síncronas: Order Service llama a Billing Service vía `POST /charges`, Billing Service
llama a Stripe síncronamente y retorna la respuesta al Order Service, que luego llama
a Kitchen Service vía `POST /tickets`, y así sucesivamente. El consumidor recibe la
respuesta del `POST /orders` solo cuando toda la cadena Order → Billing → Kitchen se
ha completado. Se usan timeouts y circuit breakers (Resilience4j) para limitar el
impacto de servicios lentos.

**Pros:**
- Modelo mental simple para el equipo Java/Spring Boot: HTTP/REST es el stack
  dominante y no requiere aprender brokers de mensajería → R-06 ✓
- Respuesta determinista al consumidor: el `POST /orders` retorna éxito o fallo con
  información completa de la transacción → UC-01 postcondición clara
- Compatible con Strangler Fig: el monolito puede exponer endpoints REST que los
  nuevos servicios consumen → R-05 ✓

**Contras:**
- Viola R-04 directamente: si Stripe tarda 8 s en responder (dentro de su SLA),
  el consumidor espera ≥ 8 s para confirmar el pedido → NFR-01 latencia ≤ 200 ms ✗
- Acoplamiento temporal total: si Kitchen Service está caído en el momento de crear
  el pedido, la transacción falla aunque el problema sea transitorio → R-03 ✗,
  Richardson Cap 3 identifica esto como el problema central que motiva el patrón Saga

**Impacto en dimensiones de comparación:**

| Dimensión | Resultado | Explicación |
|-----------|-----------|-------------|
| (a) Escalabilidad 5× (R-01) | ⚠️ | REST escala bien en lectura; en pico, las llamadas en cadena multiplican la latencia acumulada |
| (b) Latencia < 200 ms (R-02) | ✗ | La cadena síncrona Order→Billing→Stripe→Kitchen supera 200 ms si cualquier servicio tarda |
| (c) Tolerancia fallos externos (R-04) | ✗ | Stripe caído bloquea la confirmación del pedido; viola NFR-03 directamente |
| (d) Compatible Strangler Fig (R-05) | ✓ | El monolito puede exponer y consumir REST sin cambios de infraestructura |
| (e) Complejidad operativa | Baja | Stack conocido por el equipo; sin broker adicional |

---

### Opción 2: Mensajería Asíncrona con Apache Kafka como IPC Predominante

**Descripción:** Los servicios coordinan mediante eventos publicados en tópicos de
Kafka para flujos de larga duración (saga Order → Billing → Kitchen → Delivery →
Notifications). Las APIs REST/JSON síncronas se mantienen solo para operaciones de
consulta del consumidor (GET) y para el canal de entrada inicial (`POST /orders`).
El patrón Transactional Outbox [Richardson Cap 3] garantiza que ningún evento se
pierda si el servicio falla entre la escritura en BD y la publicación en Kafka. El
Order Service actúa como orquestador de la saga vía comandos/respuestas asíncronos
[Richardson Cap 4, Orchestration Saga].

**Pros:**
- Desacoplamiento temporal: si Kitchen Service está caído, los eventos se acumulan
  en Kafka y se procesan al recuperarse → R-04 ✓, NFR-03 ✓
- Cada consumidor de Kafka escala sus partitions independientemente sin impactar
  los productores → R-01 ✓, NFR-04
- Tolerancia a fallos de Stripe: Billing Service consume el evento `OrderApproved`,
  intenta el cobro, y si falla reencola el mensaje con backoff; Order Taking no
  espera → R-04 ✓, NFR-03
- Compatible con Strangler Fig: el monolito puede publicar y consumir eventos Kafka
  mientras se migran servicios → R-05 ✓

**Contras:**
- Consistencia eventual visible para el consumidor: entre la confirmación del pedido
  (UC-01) y la actualización del estado del ticket en Kitchen (UC-02) hay un lag de
  ~500 ms–2 s dependiendo del lag de Kafka y del consumer group; la UX debe mostrar
  el estado `PENDING_KITCHEN` como transición válida, no como error
- Complejidad operativa real de Kafka: gestión de clúster, retention policies,
  consumer lag monitoring, deserializadores y el patrón Outbox requieren 3–5 semanas
  de setup inicial y expertise específico que el equipo Java/Spring Boot no tiene hoy

**Impacto en dimensiones de comparación:**

| Dimensión | Resultado | Explicación |
|-----------|-----------|-------------|
| (a) Escalabilidad 5× (R-01) | ✓ | Partitions Kafka escalan por servicio independientemente |
| (b) Latencia < 200 ms (R-02) | ✓ | REST síncrono para acciones del consumidor; Kafka solo para saga interna |
| (c) Tolerancia fallos externos (R-04) | ✓ | Stripe caído → Billing reintenta desde Kafka sin bloquear Order Taking |
| (d) Compatible Strangler Fig (R-05) | ✓ | Monolito publica/consume eventos Kafka durante coexistencia |
| (e) Complejidad operativa | Alta | Gestión de clúster Kafka + patrón Outbox + distributed tracing |

---

### Opción 3: IPC Híbrido — REST Síncrono para Lectura + gRPC para Escritura Crítica

**Descripción:** Se diferencia el tipo de IPC según la naturaleza de la operación:
(a) las operaciones de lectura del consumidor (búsqueda de restaurantes, consulta de
estado del pedido UC-05) usan REST/JSON sobre HTTP/1.1 hacia el API Gateway;
(b) las operaciones de escritura crítica entre servicios internos (Order Service →
Billing Service) usan gRPC sobre HTTP/2 con contratos Protobuf para minimizar latencia
y aprovechar streaming bidireccional para el tracking en tiempo real (UC-05);
(c) las notificaciones (CAP-07) se siguen enviando por un broker ligero (RabbitMQ o
Redis Pub/Sub) para no bloquear el flujo transaccional.

**Pros:**
- gRPC reduce la latencia interna entre Order Service y Billing Service a ~5–20 ms
  (vs ~30–80 ms de REST/JSON) gracias a Protobuf y HTTP/2 → R-02 ✓ parcial
- Streaming bidireccional gRPC es natural para el tracking en tiempo real de UC-05
  sin necesidad de polling → NFR-01 ✓ para CAP-05
- Menor overhead operativo que Kafka: no se necesita un clúster de broker pesado para
  la saga principal → R-06 ⚠️

**Contras:**
- gRPC no resuelve R-04: la llamada gRPC de Order Service a Billing Service sigue
  siendo síncrona y bloqueante; si Stripe tarda, el consumidor espera → R-04 ✗,
  NFR-03 ✗ — el problema de tolerancia a fallos externos persiste igual que en Opción 1
- Dos protocolos (REST + gRPC) + un broker (RabbitMQ) aumentan la heterogeneidad
  del stack: el equipo Java/Spring Boot debe dominar tres tecnologías de IPC en lugar
  de una → R-06 ✗, complejidad operativa Alta
- gRPC tiene soporte limitado en browsers; el API Gateway necesita transcoding
  gRPC-Web, añadiendo una capa adicional de complejidad para las apps móviles/web

**Impacto en dimensiones de comparación:**

| Dimensión | Resultado | Explicación |
|-----------|-----------|-------------|
| (a) Escalabilidad 5× (R-01) | ⚠️ | gRPC escala bien, pero RabbitMQ puede ser cuello de botella en picos 5× sin particionado |
| (b) Latencia < 200 ms (R-02) | ✓ | gRPC + Protobuf reduce latencia interna; REST para consumidor |
| (c) Tolerancia fallos externos (R-04) | ✗ | La llamada gRPC a Billing sigue siendo síncrona bloqueante ante timeout de Stripe |
| (d) Compatible Strangler Fig (R-05) | ⚠️ | Monolito debe exponer endpoints gRPC para coexistir; retrofit no trivial |
| (e) Complejidad operativa | Alta | Tres tecnologías IPC simultáneas; equipo sin experiencia en gRPC |

---

## Decisión

**Se adopta la Opción 2: Mensajería Asíncrona con Apache Kafka como IPC Predominante.**

La Opción 2 es la única que satisface R-04 (tolerancia a fallos de Stripe) sin
violar R-02 (latencia ≤ 200 ms al consumidor), porque separa el canal de entrada del
consumidor (REST síncrono `POST /orders` → responde en < 200 ms con `PENDING_PAYMENT`)
del flujo interno de la saga (Order→Billing→Kitchen→Delivery vía eventos Kafka). La
Opción 1 viola R-04 estructuralmente y la Opción 3 lo replica al mantener la llamada
a Billing como síncrona.

Los contras de la Opción 2 se mitigan de forma concreta: (a) el lag de consistencia
eventual de 500 ms–2 s se gestiona diseñando los estados intermedios del pedido como
transiciones explícitas en la UX de UC-01 y UC-05 (el consumidor ve `CONFIRMING →
CONFIRMED → KITCHEN_PENDING → PREPARING`), no como estados de error; (b) la
complejidad operativa de Kafka se reduce usando Confluent Cloud o MSK (AWS) como
servicio gestionado en lugar de operar el clúster internamente, eliminando la carga de
gestión de brokers del equipo.

Esta decisión se fundamenta en el patrón *Transactional Outbox* descrito en
Richardson Cap 3 y en el patrón *Orchestration Saga* descrito en Richardson Cap 4,
que establecen juntos la forma canónica de gestionar transacciones distribuidas en
FTGO: el Order Service orquesta la saga publicando comandos en Kafka (`ReserveCredit`,
`CreateTicket`, `AssignDelivery`) y consume respuestas de cada participante, sin que
ningún servicio tenga dependencia síncrona de otro durante el flujo de creación del pedido.

---

## Consecuencias

### Positivas
- Order Taking puede confirmar el pedido del consumidor en < 200 ms (solo persiste el
  `Order` localmente y publica el evento `OrderCreated`) aunque Stripe esté caído;
  el cobro ocurrirá cuando Stripe recupere disponibilidad → R-04 ✓, NFR-03 ✓
- Kitchen Service, Delivery Service y Notifications Service pueden caer y reiniciarse
  sin perder eventos: Kafka retiene los mensajes durante el tiempo de retención
  configurado, habilitando el uptime 99.9% de Order Taking de forma independiente a
  los servicios downstream → R-03 ✓, NFR-02 ✓

### Negativas
- La consistencia eventual entre Order Service y Kitchen Service introduce un lag
  real de ~500 ms–2 s entre la confirmación del pedido y la aparición del ticket en
  el dashboard del restaurante (UC-02): si el consumidor recibe `CONFIRMED` pero el
  restaurante aún no ve el ticket, se producen consultas de soporte que el Empleado
  FTGO debe gestionar con la herramienta de visibility de back office.
- La adopción del patrón Transactional Outbox requiere que cada servicio FTGO
  implemente una tabla `outbox` en su base de datos y un proceso de relay (Debezium o
  polling scheduler): esto añade 1–2 semanas de implementación por servicio nuevo y
  es un punto de fallo adicional si el relay se detiene sin alerta configurada.

---

## Follow-ups

- **ADR posterior necesario:** ADR 0003 debe decidir la estrategia de descomposición
  de datos (Database-per-Service estricto vs esquemas compartidos con RLS): esta
  decisión de IPC asume Database-per-Service, pero el equipo debe definir cómo
  migrar los datos compartidos del monolito actual.
- **POC recomendado:** Implementar el patrón Transactional Outbox + Kafka para el
  flujo UC-01 → UC-04 (Order Service → Billing Service) en un entorno de staging
  con Stripe en modo sandbox, midiendo el lag real de propagación y validando que
  la latencia de respuesta al consumidor se mantiene bajo 200 ms p95 (NFR-01).
- **Consideración de migración:** Durante el Strangler Fig, el monolito legacy puede
  publicar eventos Kafka usando un CDC (Change Data Capture) sobre su base de datos
  existente (Debezium + MySQL binlog), sin modificar el código del monolito, lo que
  permite iniciar la integración con los nuevos servicios antes de completar la
  extracción del Bounded Context.
