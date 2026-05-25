# ADR 0001 — Estilo Arquitectónico para la Migración de FTGO

**Estado:** Accepted  
**Fecha:** 2026-05-25  
**Decisores:** Equipo de Arquitectura FTGO  
**Trazabilidad:** PRD NFR-04 · PRD NFR-06 · FSD UC-01..UC-05 · Brief §A.4 R-01 R-05 · Richardson Cap 1–2

---

## Contexto

El monolito FTGO exhibe síntomas documentados de *monolith hell*: builds superiores a
30 minutos, escalado conflictivo que obliga a aprovisionar toda la app para absorber el
pico 5× de las horas de almuerzo y cena (R-01), y ausencia de aislamiento de fallos que
hace que un error en el módulo de Billing pueda derribar el flujo de Order Taking
(NFR-02: uptime ≥ 99.9%). El lock-in tecnológico en Java monolítico impide adoptar el
stack óptimo por dominio y ralentiza la entrega de cada capacidad.

Es necesario decidir el estilo arquitectónico ahora porque los 5 UCs del FSD
(UC-01 a UC-05) involucran actores, aggregates y flujos de datos completamente
distintos (Consumidor→Order, Restaurante→Ticket, Courier→Delivery, Sistema→Account)
que bajo el monolito comparten un único punto de despliegue. Sin esta decisión no es
posible definir los límites de los Bounded Contexts, los contratos de IPC ni la
estrategia de datos, que son los ADRs siguientes.

Las restricciones R-01 (escalabilidad 5×), R-03 (disponibilidad 99.9% Order Taking),
R-05 (Strangler Fig 18–24 meses) y R-07 (consistencia eventual aceptada para reporting)
condicionan fuertemente el espacio de soluciones: cualquier opción que requiera
reemplazo big-bang del monolito o que escale solo como unidad completa queda descartada.

---

## Restricciones Aplicables

| ID | Restricción | Fuente | Impacto en esta decisión |
|----|-------------|--------|--------------------------|
| R-01 | Tráfico pico 5× durante 12:00–14:00 y 19:00–22:00 | Brief §A.4 | Exige escalar CAP-03 (Order Taking) de forma independiente a CAP-07 (Notifications) |
| R-03 | Disponibilidad 99.9% mensual en Order Taking | Brief §A.4 | Un fallo en Billing o Kitchen no puede propagar al flujo de creación de pedido |
| R-05 | Strangler Fig 18–24 meses | Brief §A.4 | Descarta big-bang; el estilo elegido debe coexistir con el monolito durante la transición |
| R-06 | Java/Spring Boot preferido en core | Brief §A.4 | Restringe opciones que exigen migración masiva de runtime o lenguaje |
| R-07 | Consistencia eventual aceptada para reporting | Brief §A.4 | Habilita separación de modelos de lectura/escritura por servicio |

---

## Opciones Consideradas

### Opción 1: Microservicios con DDD Bounded Contexts (decomposición por capacidad de negocio)

**Descripción:** Cada capacidad CAP-01..07 se modela como un Bounded Context con su
propio aggregate raíz, base de datos independiente (Database-per-Service) y despliegue
autónomo. Los 5 UCs del FSD determinan los límites: UC-01 y UC-04 pertenecen al Order
Service (aggregate `Order`) y Billing Service (aggregate `Account`); UC-02 al Kitchen
Service (aggregate `Ticket`); UC-03 y UC-05 al Delivery Service (aggregate `Delivery`).
La migración sigue el patrón Strangler Fig: cada capacidad se extrae del monolito de
forma incremental durante 18–24 meses sin interrupción de servicio.

**Pros:**
- Escalado independiente por servicio: CAP-03 Order Taking puede escalar a 5× sin
  tocar CAP-07 Notifications → R-01 ✓, NFR-04
- Aislamiento de fallos: un crash en Kitchen Service no impacta la disponibilidad del
  Order Service → R-03 ✓, NFR-02
- Compatible por diseño con Strangler Fig: cada BC se extrae y despliega
  incrementalmente → R-05 ✓
- Libertad tecnológica por dominio: Delivery Service puede usar Kotlin o Go si el
  equipo lo decide → R-06 ⚠️ (Spring Boot en core, libertad en satélites)
- Alineado con Richardson Cap 2: "decompose by business capability" como estrategia
  canónica para FTGO

**Contras:**
- Consistencia distribuida: las transacciones que cruzan aggregates (UC-01 dispara
  UC-04) requieren Sagas en lugar de transacciones ACID → complejidad real para el
  equipo Java, curva de aprendizaje de 2–4 semanas por servicio
- Overhead operativo desde el día 1: cada servicio necesita su propio pipeline CI/CD,
  health checks, distributed tracing y gestión de secretos → impacto en R-05
  (la migración incremental tiene más pasos que una SOA)

**Impacto en dimensiones de comparación:**

| Dimensión | Resultado | Explicación |
|-----------|-----------|-------------|
| (a) Escalabilidad 5× (R-01) | ✓ | Cada servicio escala sus réplicas independientemente (Kubernetes HPA) |
| (b) Latencia < 200 ms (R-02) | ✓ | REST síncrono para acciones del consumidor; async para saga interna |
| (c) Tolerancia fallos externos (R-04) | ✓ | Billing Service aislado; su fallo no bloquea Order Taking |
| (d) Compatible Strangler Fig (R-05) | ✓ | Extracción incremental de BCs; monolito sigue operando |
| (e) Complejidad operativa | Alta | Múltiples pipelines, Sagas, distributed tracing, DB-per-service |

---

### Opción 2: Monolito Modular como destino final (no solo etapa intermedia)

**Descripción:** El monolito actual se refactoriza internamente en módulos con
interfaces bien definidas (CAP-01..07 como paquetes Java con boundaries explícitas),
sin separar el despliegue. Cada módulo tiene su propio esquema de base de datos dentro
de una instancia PostgreSQL compartida. El sistema se despliega como una única unidad
WAR/JAR pero con fronteras de código que imitan los Bounded Contexts. Esta opción es
especialmente viable si el equipo considera que el overhead operativo de microservicios
supera el beneficio para el tamaño actual de FTGO.

**Pros:**
- Transacciones ACID entre módulos: UC-01 (Order) + UC-04 (Billing) en una sola
  transacción local sin Saga → menor complejidad de consistencia, R-07 ✓
- Sin overhead de red entre módulos: latencia intra-proceso sub-milisegundo → R-02 ✓
- Compatible con Strangler Fig como etapa intermedia: el monolito modular puede
  usarse para preparar los Bounded Contexts antes de extraerlos → R-05 ✓ parcial

**Contras:**
- Escalado no independiente: un pico en CAP-05 Delivery (matching de couriers) obliga
  a escalar toda la aplicación, incluyendo CAP-07 Notifications → R-01 ✗, NFR-04 ✗
- Un fallo en un módulo mal aislado puede seguir propagándose al mismo proceso JVM:
  un `OutOfMemoryError` en el módulo de reporting derriba Order Taking → R-03 ✗
- Deuda técnica diferida: las boundaries de módulo se erosionan con el tiempo si no
  hay enforcement tooling (ArchUnit, jQAssistant); en 18 meses se regresa al punto
  de partida

**Impacto en dimensiones de comparación:**

| Dimensión | Resultado | Explicación |
|-----------|-----------|-------------|
| (a) Escalabilidad 5× (R-01) | ✗ | Toda la app escala como unidad; CAP-03 no puede escalarse sin CAP-07 |
| (b) Latencia < 200 ms (R-02) | ✓ | Llamadas intra-proceso; sin overhead de red entre módulos |
| (c) Tolerancia fallos externos (R-04) | ⚠️ | Aislamiento parcial: JVM compartida permite propagación de errores críticos |
| (d) Compatible Strangler Fig (R-05) | ⚠️ | Útil como etapa intermedia, pero no como destino final si el objetivo es escalado independiente |
| (e) Complejidad operativa | Baja | Un solo artefacto de despliegue; pipeline CI/CD existente reutilizable |

---

### Opción 3: Arquitectura Orientada a Servicios (SOA) con servicios de grano grueso

**Descripción:** Se definen 3 servicios de grano grueso agrupando capacidades
relacionadas: (A) `OrderPlatformService` cubre CAP-01, CAP-02, CAP-03, CAP-04;
(B) `LogisticsService` cubre CAP-05 Delivery; (C) `FinanceService` cubre CAP-06 y
CAP-07. Cada servicio tiene su propia base de datos y se comunica por contratos
SOAP/REST centralizados a través de un ESB (Enterprise Service Bus) o API Manager.
Es el patrón SOA clásico que muchas organizaciones Java adoptaron antes de microservicios.

**Pros:**
- Menos servicios que gestionar que la Opción 1: 3 servicios vs 7+ → menor overhead
  operativo inicial → R-06 ✓ (equipo Java tiene experiencia con ESB)
- Permite cierto aislamiento de fallos: `FinanceService` caído no bloquea
  `OrderPlatformService` → R-03 ⚠️
- Compatible con Strangler Fig a nivel macro: el monolito se divide en 3 unidades
  grandes en lugar de N pequeñas → R-05 ✓ parcial

**Contras:**
- `OrderPlatformService` agrupa CAP-01..04 y sigue siendo un monolito interno: en
  horas pico, escalar el servicio implica escalar también la gestión de consumers
  y restaurantes que tienen tráfico bajo → R-01 ✗ parcial
- El ESB/API Manager centralizado se convierte en un único punto de fallo y cuello de
  botella de rendimiento: todas las llamadas IPC pasan por él → R-03 ✗, Richardson
  Cap 1 cita el ESB como anti-patrón en sistemas distribuidos modernos
- No hay ganancia de aislamiento de datos: CAP-01 y CAP-03 comparten la misma BD
  en `OrderPlatformService`, perpetuando el acoplamiento a nivel de esquema

**Impacto en dimensiones de comparación:**

| Dimensión | Resultado | Explicación |
|-----------|-----------|-------------|
| (a) Escalabilidad 5× (R-01) | ⚠️ | Escalado por servicio grueso, no por capacidad; CAP-03 arrastra a CAP-01/02 |
| (b) Latencia < 200 ms (R-02) | ⚠️ | ESB centralizado puede introducir latencia adicional de 20–80 ms por hop |
| (c) Tolerancia fallos externos (R-04) | ⚠️ | Depende de la resiliencia del ESB; si el bus falla, toda la comunicación se interrumpe |
| (d) Compatible Strangler Fig (R-05) | ✓ | Migración en 3 fases grandes es viable dentro de 18–24 meses |
| (e) Complejidad operativa | Media | Menos artefactos que microservicios pero ESB requiere expertise específico |

---

## Decisión

**Se adopta la Opción 1: Microservicios con DDD Bounded Contexts.**

La Opción 1 es la única que satisface simultáneamente R-01 (escalado 5× independiente
por capacidad), R-03 (aislamiento de fallos para 99.9% de Order Taking) y R-07
(consistencia eventual aceptada, que es la base del patrón Database-per-Service).
La Opción 2 viola R-01 al no permitir escalado independiente, y la Opción 3 introduce
un ESB como single point of failure que viola R-03 y que Richardson Cap 1 identifica
explícitamente como anti-patrón en el contexto de microservicios.

Los contras de la Opción 1 se mitigan de dos formas concretas: (a) la complejidad de
Sagas se gestiona mediante el patrón Orchestration Saga del Order Service como
orquestador central, reduciendo la lógica de compensación a un único lugar en lugar
de distribuirla entre servicios; (b) el overhead operativo se introduce gradualmente
siguiendo el plan Strangler Fig: los primeros servicios extraídos (CAP-03 Order Taking
y CAP-06 Billing) establecen los pipelines y patrones que los equipos replican para
las capacidades restantes.

Esta decisión se fundamenta en el patrón *Decompose by Business Capability* descrito
en Richardson Cap 2, que establece que cada capacidad de negocio de FTGO
(Consumer Management, Order Taking, Kitchen, Delivery, Billing, Notifications) debe
mapearse a un servicio con un aggregate raíz propio, base de datos independiente y
ciclo de vida de despliegue autónomo.

---

## Consecuencias

### Positivas
- CAP-03 Order Taking puede escalar de forma autónoma a 5× durante picos sin impactar
  las capacidades de bajo tráfico (CAP-02 Restaurant Management) → R-01, NFR-04
- Un fallo en Kitchen Service (CAP-04) o Delivery Service (CAP-05) no derriba el
  aggregate `Order`: el uptime 99.9% de CAP-03 se puede garantizar con deploys
  independientes → R-03, NFR-02

### Negativas
- La consistencia distribuida introduce un lag visible para el consumidor de ~500 ms–2 s
  entre la confirmación del pedido (UC-01) y la actualización del estado del ticket en
  Kitchen (UC-02): la UX debe diseñarse con estados intermedios explícitos (`CONFIRMED`,
  `PENDING_KITCHEN`) para no confundir al usuario con información desactualizada.
- El número de artefactos de CI/CD, Dockerfiles, schemas de BD y configuraciones de
  health check crece linealmente con el número de servicios: para los 7+ BCs de FTGO,
  el equipo debe invertir 4–6 semanas en establecer plantillas de servicio y pipelines
  antes de extraer el primer Bounded Context productivo.

---

## Follow-ups

- **ADR posterior necesario:** ADR 0002 debe decidir el mecanismo de IPC entre
  servicios (REST síncrono vs mensajería asíncrona Kafka vs híbrido), ya que esta
  decisión define que los servicios existen pero no cómo se comunican.
- **POC recomendado:** Extraer CAP-07 Notifications como primer servicio piloto
  (bajo riesgo, sin aggregate crítico) para validar el pipeline CI/CD, el patrón
  Strangler Fig y las plantillas de despliegue antes de abordar CAP-03 Order Taking.
- **Consideración de migración:** El Strangler Fig para CAP-03 requiere colocar un
  API Gateway/proxy inverso frente al monolito que reencamine gradualmente el tráfico
  de `POST /orders` hacia el nuevo Order Service; este routing incremental debe
  planificarse en un ADR de infraestructura separado.
