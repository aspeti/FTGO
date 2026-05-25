# PRD Ligero — FTGO (Food To Go)

**Versión:** 1.0  
**Fecha:** 25/05/2026  
**Estado:** Aprobado  
**Trazabilidad:** Brief §A.1–A.5, Richardson Cap 1–2

---

## 1. Contexto y Objetivos

FTGO (Food To Go) es una plataforma de delivery de comida que conecta consumidores, restaurantes y couriers a través de aplicaciones móviles y web. Actualmente opera sobre un monolito Java (WAR) que exhibe síntomas del *monolith hell*: builds lentos (> 30 min), escalado conflictivo (toda la app o nada), ausencia de aislamiento de fallos (un bug en Billing puede derribar Order Taking) y lock-in tecnológico que impide adoptar el stack óptimo por dominio.

La dirección decidió migrar a una arquitectura de microservicios aplicando la estrategia **Strangler Fig** durante un horizonte de **18 a 24 meses**: nuevas capacidades se construyen como servicios independientes mientras el monolito se reduce gradualmente, sin reemplazos big-bang ni paradas de servicio. El objetivo es lograr escalado horizontal independiente por capacidad, aislamiento de fallos, despliegues continuos por equipo y libertad tecnológica por dominio, manteniendo la continuidad operativa durante toda la transición.

---

## 2. Stakeholders

| Rol | Descripción | Necesidad principal |
|-----|-------------|---------------------|
| **Consumidor** | Usuario final (móvil/web) que realiza pedidos de comida | UX rápida, tracking en tiempo real, transparencia del estado del pedido |
| **Restaurante** | Negocio registrado que prepara y despacha comida | Gestión eficiente de tickets de cocina, control de carga, dashboard de pedidos activos |
| **Courier** | Repartidor independiente que recoge y entrega pedidos | Asignaciones de proximidad, rutas optimizadas, pago confiable y puntual |
| **Empleado FTGO (back office)** | Personal de soporte, finanzas y operaciones | Visibilidad end-to-end, reportes operativos, herramientas de resolución de incidentes |
| **Equipo de arquitectura** | Responsable del rediseño y la migración | Calidad arquitectónica, trazabilidad de decisiones, mantenibilidad a largo plazo |
| **Sistemas externos** | Stripe (pagos), Google Maps (rutas), SendGrid/Twilio (notificaciones) | Integración confiable mediante contratos de API versionados y manejo de fallos con retry |

---

## 3. Capacidades de Negocio

### CAP-01: Consumer Management

Gestiona el ciclo de vida completo del consumidor: registro, autenticación, administración de direcciones de entrega y preferencias. Desde el punto de vista arquitectónico, es el Bounded Context que emite el aggregate raíz `Consumer` y es consultado por Order Taking para validar que el consumidor existe y tiene crédito disponible antes de confirmar un pedido. No necesariamente es un único microservicio; puede subdividirse en autenticación (delegable a un IdP) y perfil/preferencias.

### CAP-02: Restaurant Management

Administra el catálogo de restaurantes registrados: información del negocio, menús, horarios de operación y disponibilidad en tiempo real. Es el Bounded Context que publica el aggregate `Restaurant` y el valor de objeto `MenuItem`. Order Taking lo consulta para validar que los ítems solicitados pertenecen al restaurante activo y tienen precio vigente. La gestión del menú es de escritura baja y lectura alta, lo que favorece una estrategia de réplica de datos vía eventos hacia un read model desnormalizado.

### CAP-03: Order Taking

Núcleo del negocio. Recibe la solicitud del consumidor, valida ítems y restaurante, calcula el total y coordina la confirmación del pedido mediante el aggregate `Order`. Orquesta la saga de creación de pedido, que involucra validar consumidor (CAP-01), obtener precio del menú (CAP-02), reservar crédito (CAP-06) y crear el ticket de cocina (CAP-04). Es el dominio core con mayor criticidad de disponibilidad (99.9%) y el que exige consistencia fuerte dentro de su propio aggregate.

### CAP-04: Order Fulfillment / Kitchen

Traduce los pedidos aprobados en tickets de cocina y gestiona su ciclo de preparación: `ACCEPTED → PREPARING → READY_FOR_PICKUP`. El aggregate `Ticket` vive en este Bounded Context y reacciona a eventos del Order Service. El restaurante interactúa con esta capacidad para actualizar el estado de preparación; cuando el ticket está listo, dispara la asignación del courier en CAP-05. Es un dominio de soporte con bajo acoplamiento hacia el consumidor.

### CAP-05: Delivery

Gestiona la asignación de couriers disponibles, el cálculo y actualización de rutas en tiempo real mediante integración con Google Maps, y el tracking del pedido desde el restaurante hasta la puerta del consumidor. Expone el estado de entrega al consumidor y a back office con tolerancia a degradación (disponibilidad 99.5%). La lógica de asignación (matching) y el tracking son subdominios diferenciados que pueden escalar independientemente según el pico geográfico.

### CAP-06: Billing & Accounting

Responsable de los cobros al consumidor mediante Stripe, el cálculo y pago de comisiones a restaurantes y la liquidación de payouts a couriers. La integración con Stripe delega el cumplimiento PCI-DSS. Este dominio implementa patrones de consistencia eventual con reconciliación periódica y mantiene una cola de retry para procesar cobros cuando Stripe no responde. Constituye un dominio de soporte cuya falla no debe impedir tomar pedidos (degradación controlada con cola).

### CAP-07: Notifications

Envía comunicaciones transaccionales al consumidor y a restaurantes/couriers a través de email (SendGrid), SMS y push (Twilio): confirmación de pedido, alertas de estado, recibos y notificaciones de asignación de courier. Es un dominio genérico de alta cohesión y bajo acoplamiento; consume eventos de dominio publicados por CAP-03, CAP-04 y CAP-05 sin dependencia inversa. Puede degradarse sin afectar el flujo transaccional principal.

---

## 4. Requisitos No Funcionales

### NFR-01: Latencia de Experiencia de Usuario

- **Métrica:** Tiempo de respuesta ≤ 200 ms p95 para todas las acciones del consumidor (búsqueda, creación de pedido, consulta de estado).
- **Origen:** Brief §A.4 Latencia UX
- **Justificación:** En marketplaces de delivery, cada 100 ms de latencia adicional reduce la tasa de conversión; superar 200 ms p95 en flujos de checkout genera abandono medible.

### NFR-02: Disponibilidad de Order Taking

- **Métrica:** Uptime ≥ 99.9% mensual (≤ 43.8 min de downtime/mes) para las capacidades CAP-03 y CAP-06.
- **Origen:** Brief §A.4 Disponibilidad
- **Justificación:** La indisponibilidad de Order Taking implica pérdida directa de ingresos y deterioro de la confianza del consumidor; es la capacidad de mayor criticidad de negocio.

### NFR-03: Tolerancia a Fallos de Sistemas Externos

- **Métrica:** El sistema debe procesar ≥ 100% de los pedidos intentados durante interrupciones de Stripe de hasta 10 minutos, mediante cola de retry con backoff exponencial (intervalo inicial 1 s, máximo 5 reintentos).
- **Origen:** Brief §A.4 Tolerancia a fallos externos
- **Justificación:** Stripe tiene SLA del 99.99% pero experimenta micro-interrupciones; rechazar pedidos por fallos de pago externo es inaceptable para la propuesta de valor.

### NFR-04: Escalabilidad Horizontal Bajo Carga Pico

- **Métrica:** Cada servicio desplegado debe escalar a 5× su tráfico base en ≤ 3 minutos (horizontal pod autoscaler, p. ej. Kubernetes HPA) durante ventanas de pico 12:00–14:00 y 19:00–22:00 hora local.
- **Origen:** Brief §A.4 Carga / Escalabilidad horizontal
- **Justificación:** El tráfico de FTGO presenta picos 5× durante las horas de almuerzo y cena; un escalado monolítico implica sobreaprovisionar toda la app, encareciendo el costo operativo.

### NFR-05: Trazabilidad End-to-End

- **Métrica:** 100% de las acciones del consumidor deben generar spans de distributed tracing con `correlation_id` propagado en todos los servicios involucrados; retención mínima de trazas: 30 días.
- **Origen:** Brief §A.4 Trazabilidad
- **Justificación:** En una arquitectura distribuida, la ausencia de trazabilidad convierte el diagnóstico de incidentes en una operación de horas; un SLA de resolución de incidentes de soporte requiere trazas consultables.

### NFR-06: Migración Incremental sin Interrupción de Servicio

- **Métrica:** Durante los 18–24 meses de migración, el tiempo de inactividad planificado por evento de migración (Strangler Fig cut-over) debe ser ≤ 5 minutos, medido como `downtime_per_migration_event`.
- **Origen:** Brief §A.4 Migración incremental
- **Justificación:** Una migración big-bang con ventana de mantenimiento extendida es inviable para un marketplace operativo 24/7; cada cut-over debe ser invisible para el consumidor final.

---

## 5. Alcance

### Dentro del alcance

- Las 7 capacidades de negocio (CAP-01 a CAP-07) definidas en el Brief §A.2, con su descripción de responsabilidad, aggregate raíz y relaciones entre Bounded Contexts.
- Definición de los 6 stakeholders y sus necesidades principales como guía de priorización.
- Los 6 NFRs base con métricas numéricas, origen trazable y justificación de negocio.
- La estrategia de migración Strangler Fig como restricción arquitectónica no negociable durante el horizonte 2025–2026.
- Integraciones externas declaradas: Stripe (pagos y PCI-DSS), Google Maps (rutas y tracking), SendGrid/Twilio (notificaciones).
- El flujo transaccional principal: consumidor crea pedido → validación → confirmación → ticket de cocina → asignación courier → entrega → notificación → cobro.

### Fuera del alcance

- **Código fuente y diseño técnico detallado:** este PRD no incluye código Java, esquemas de base de datos, contratos de API ni diagramas de secuencia; esos artefactos pertenecen al FSD y al nivel de diseño técnico.
- **Reemplazo big-bang del monolito:** el PRD no aprueba ni planifica la eliminación total del monolito en una sola operación; la estrategia Strangler Fig excluye explícitamente este enfoque.
- **Módulo de gestión de personal interno de restaurantes:** la administración de empleados, turnos y roles dentro del restaurante está fuera del dominio de FTGO v1.
- **Funcionalidades de fidelización y gamificación:** puntos de recompensa, niveles de membresía y programas de lealtad no forman parte del scope de la migración inicial.
- **Soporte multi-país y multi-moneda:** la versión inicial del rediseño asume una sola región/moneda; la internacionalización completa es un roadmap posterior.

---

_Documento trazable a: Brief §A.1–A.5 | Richardson, Microservices Patterns, Manning 2019, Cap 1–2_
