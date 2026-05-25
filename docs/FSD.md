# FSD Ligero — FTGO (Food To Go)

**Versión:** 1.0  
**Fecha:** 25/05/2026  
**Estado:** Aprobado  
**Documento fuente:** docs/PRD.md, Brief §A.5, Richardson Cap 3–4

---

## 1. Introducción

Este FSD formaliza los cinco casos de uso del núcleo operativo de FTGO: tres derivan de las user stories semilla del brief (US-01 a US-03) y dos se justifican por el patrón Saga de Richardson (Cap. 3) y los NFRs de tolerancia a fallos y latencia del PRD. Cada UC incluye trazabilidad a la capacidad del PRD, flujos alternativos reales del dominio delivery y bloques Given/When/Then ejecutables. Este documento es la entrada directa para los ADRs de estilo arquitectónico e IPC y para el diseño de contratos de API entre servicios.

---

## 2. Tabla de Casos de Uso

| ID | Título | Actor primario | Capacidad PRD | Origen |
|----|--------|---------------|---------------|--------|
| UC-01 | Tomar pedido | Consumidor | CAP-03 Order Taking | US-01 [Brief §A.5] |
| UC-02 | Aceptar o rechazar ticket de cocina | Restaurante | CAP-04 Order Fulfillment | US-02 [Brief §A.5] |
| UC-03 | Asignar pedido a courier | Courier | CAP-05 Delivery | US-03 [Brief §A.5] |
| UC-04 | Procesar pago del pedido | Sistema (Billing Service) | CAP-06 Billing & Accounting | Richardson Cap 3 + NFR-03 [PRD] |
| UC-05 | Tracking en tiempo real del pedido | Consumidor | CAP-05 Delivery | NFR-01 + NFR-02 [PRD] |

---

## 3. Detalle de Casos de Uso

---

### UC-01: Tomar Pedido

| Campo | Valor |
|-------|-------|
| **Actor primario** | Consumidor |
| **Capacidad PRD** | CAP-03 Order Taking |
| **Origen** | US-01 [Brief §A.5] |

**Precondiciones:**
- El consumidor está autenticado en la app (CAP-01 Consumer Management).
- El restaurante seleccionado está activo y dentro de su horario de atención.
- El consumidor tiene al menos una dirección de entrega registrada.

**Flujo principal:**
1. El consumidor selecciona un restaurante y visualiza su menú activo.
2. El consumidor agrega uno o más ítems al carrito (cantidad, notas especiales).
3. El consumidor confirma la dirección de entrega y el método de pago.
4. El sistema valida: (a) disponibilidad del restaurante, (b) disponibilidad de ítems, (c) método de pago válido.
5. El sistema crea el aggregate `Order` en estado `PENDING_PAYMENT` con número de pedido único.
6. El sistema inicia el flujo de pago disparando UC-04.
7. El consumidor recibe confirmación con número de pedido y tiempo estimado de entrega.

**Flujos alternativos:**
- **FA-01 Restaurante no disponible:** Si el restaurante cerró entre la selección y la confirmación → el sistema informa al consumidor y sugiere alternativas disponibles cercanas.
- **FA-02 Ítem agotado:** Si un ítem del carrito ya no está disponible al confirmar → el sistema notifica al consumidor y permite modificar el carrito antes de reintentar.
- **FA-03 Pago fallido:** Si Stripe rechaza el pago → el `Order` permanece en `PENDING_PAYMENT`; el sistema encola reintento con backoff exponencial (NFR-03 [PRD]).

**Postcondiciones:**
- `Order` en estado `CONFIRMED` con número único asignado.
- Evento `OrderCreated` publicado en el broker de eventos.
- Consumidor notificado vía CAP-07 Notifications (email/push de confirmación).

**Given/When/Then:**

> **Escenario principal — pedido exitoso**
> - **Given:** El consumidor tiene un carrito con al menos 1 ítem disponible, el restaurante está abierto y el método de pago es válido.
> - **When:** El consumidor confirma el pedido con dirección de entrega.
> - **Then:** El sistema crea el `Order` en estado `CONFIRMED`, asigna un número de pedido único, publica el evento `OrderCreated` y envía confirmación al consumidor en < 200 ms p95 (NFR-01 [PRD]).

> **Escenario alternativo — Stripe temporalmente no disponible**
> - **Given:** El consumidor confirma el pedido con pago válido, pero Stripe devuelve timeout.
> - **When:** El sistema intenta procesar el cobro y recibe error de timeout.
> - **Then:** El `Order` permanece en `PENDING_PAYMENT`, el sistema encola el reintento con backoff exponencial (intervalo inicial 1 s, máximo 5 reintentos) y el consumidor recibe mensaje "pago en proceso" (NFR-03 [PRD]).

---

### UC-02: Aceptar o Rechazar Ticket de Cocina

| Campo | Valor |
|-------|-------|
| **Actor primario** | Restaurante |
| **Capacidad PRD** | CAP-04 Order Fulfillment / Kitchen |
| **Origen** | US-02 [Brief §A.5] |

**Precondiciones:**
- El `Order` se encuentra en estado `CONFIRMED` (UC-01 completado).
- El Restaurante tiene una sesión activa en el dashboard de cocina.
- El aggregate `Ticket` fue creado por el servicio Kitchen como resultado del evento `OrderCreated`.

**Flujo principal:**
1. El dashboard del restaurante recibe notificación de nuevo ticket pendiente (vía CAP-07).
2. El restaurante visualiza los detalles del ticket: ítems, cantidades, notas y tiempo máximo de respuesta.
3. El restaurante acepta el ticket e ingresa el tiempo estimado de preparación en minutos.
4. El sistema actualiza el aggregate `Ticket` a estado `ACCEPTED` y registra el tiempo estimado.
5. El sistema notifica al consumidor con el tiempo estimado de preparación actualizado.
6. El restaurante actualiza el estado a `PREPARING` cuando inicia la elaboración.
7. El restaurante marca el ticket como `READY_FOR_PICKUP` al concluir la preparación.

**Flujos alternativos:**
- **FA-01 Restaurante rechaza el ticket:** Si el restaurante no puede preparar el pedido → selecciona motivo (falta de ingredientes, saturación de cocina, cierre anticipado); el sistema cancela el `Order`, ejecuta compensación del pago (dispara UC-04 en modo reverso) y notifica al consumidor con motivo y alternativas.
- **FA-02 Timeout de respuesta del restaurante:** Si el restaurante no responde dentro del tiempo máximo configurado → el sistema rechaza automáticamente el ticket, cancela el `Order` y notifica al consumidor; el incidente queda registrado para el Empleado FTGO (back office).

**Postcondiciones:**
- `Ticket` en estado `ACCEPTED` con tiempo estimado registrado, o en estado `REJECTED` con motivo.
- Evento `TicketAccepted` o `TicketRejected` publicado en el broker de eventos.
- Consumidor notificado del estado actualizado (CAP-07).

**Given/When/Then:**

> **Escenario principal — restaurante acepta el ticket**
> - **Given:** Existe un `Ticket` en estado `PENDING` para el restaurante autenticado y el restaurante tiene capacidad disponible en cocina.
> - **When:** El restaurante acepta el ticket e ingresa tiempo estimado de 25 minutos.
> - **Then:** El `Ticket` pasa a estado `ACCEPTED`, el evento `TicketAccepted` se publica en el broker y el consumidor recibe notificación push con tiempo estimado de 25 minutos.

> **Escenario alternativo — restaurante rechaza por falta de ingredientes**
> - **Given:** Existe un `Ticket` en estado `PENDING` y el restaurante selecciona motivo "ingrediente no disponible".
> - **When:** El restaurante confirma el rechazo del ticket.
> - **Then:** El `Ticket` pasa a estado `REJECTED`, el `Order` se cancela, se inicia compensación del pago, el evento `TicketRejected` se publica y el consumidor recibe notificación con motivo y sugerencia de restaurantes alternativos.

---

### UC-03: Asignar Pedido a Courier

| Campo | Valor |
|-------|-------|
| **Actor primario** | Courier |
| **Capacidad PRD** | CAP-05 Delivery |
| **Origen** | US-03 [Brief §A.5] |

**Precondiciones:**
- El `Ticket` se encuentra en estado `READY_FOR_PICKUP` (UC-02 completado).
- El Courier tiene disponibilidad activa marcada en la app y está dentro del radio de cobertura del restaurante.
- La integración con Google Maps está operativa para cálculo de rutas.

**Flujo principal:**
1. El sistema detecta el evento `TicketReady` y busca couriers disponibles en el radio configurado.
2. El sistema calcula la ruta óptima restaurante → consumidor vía Google Maps y estima el tiempo de entrega.
3. El sistema ofrece la asignación al courier más cercano disponible con detalle: restaurante, destino, distancia y tiempo estimado.
4. El courier acepta la asignación dentro del timeout de 30 segundos.
5. El sistema actualiza el aggregate `Delivery` a estado `ASSIGNED` y vincula el `courier_id`.
6. El courier visualiza la ruta optimizada al restaurante y, tras recoger el pedido, actualiza estado a `PICKED_UP`.
7. El courier entrega el pedido y confirma la entrega; el sistema actualiza `Delivery` a `DELIVERED`.

**Flujos alternativos:**
- **FA-01 Courier rechaza o no responde:** Si el courier rechaza la asignación o no responde en 30 s → el sistema ofrece la asignación al siguiente courier disponible en el radio; si no hay couriers disponibles en 3 intentos, el Empleado FTGO (back office) recibe alerta para intervención manual.
- **FA-02 Google Maps no disponible:** Si la integración de rutas falla → el sistema asigna la entrega con estimación de distancia euclidiana como fallback; el courier visualiza dirección de texto sin ruta gráfica y el incidente se registra con `correlation_id` (NFR-05 [PRD]).

**Postcondiciones:**
- `Delivery` en estado `ASSIGNED` con `courier_id`, ruta y tiempo estimado registrados.
- Evento `DeliveryAssigned` publicado en el broker de eventos.
- Consumidor notificado con nombre del courier y tiempo estimado de llegada (CAP-07).

**Given/When/Then:**

> **Escenario principal — courier acepta la asignación**
> - **Given:** El `Ticket` está en `READY_FOR_PICKUP`, el courier está disponible a 800 m del restaurante y Google Maps está operativo.
> - **When:** El courier recibe la oferta de asignación y la acepta dentro de los 30 segundos.
> - **Then:** El `Delivery` pasa a estado `ASSIGNED`, el evento `DeliveryAssigned` se publica, el courier recibe la ruta optimizada y el consumidor recibe notificación con nombre del courier y ETA.

> **Escenario alternativo — ningún courier disponible tras 3 intentos**
> - **Given:** El `Ticket` está en `READY_FOR_PICKUP` y no hay couriers disponibles en el radio de cobertura.
> - **When:** El sistema agota 3 rondas de oferta sin aceptación.
> - **Then:** El `Delivery` queda en estado `UNASSIGNED`, el Empleado FTGO recibe alerta en el panel de back office con `order_id` y `correlation_id` para intervención manual; el consumidor recibe notificación de demora.

---

### UC-04: Procesar Pago del Pedido

| Campo | Valor |
|-------|-------|
| **Actor primario** | Sistema (Billing Service) |
| **Capacidad PRD** | CAP-06 Billing & Accounting |
| **Origen** | Richardson Cap 3 (patrón Saga) + NFR-03 [PRD] |

**Precondiciones:**
- El `Order` existe en estado `PENDING_PAYMENT` (creado en UC-01).
- El método de pago del consumidor está tokenizado en Stripe (delegación PCI-DSS).
- El evento `OrderCreated` fue publicado por el Order Service y consumido por el Billing Service.

**Flujo principal:**
1. El Billing Service recibe el evento `OrderCreated` con `order_id`, `amount` y `payment_token`.
2. El sistema llama a la API de Stripe con el token de pago y el monto total del pedido.
3. Stripe procesa el cobro y retorna respuesta de éxito con `charge_id`.
4. El Billing Service registra la transacción en el aggregate `Account` con estado `CHARGED`.
5. El Billing Service publica el evento `PaymentSucceeded` con `order_id` y `charge_id`.
6. El Order Service consume el evento y actualiza el `Order` a estado `CONFIRMED`.
7. El consumidor recibe notificación de cobro exitoso y confirmación de pedido (CAP-07).

**Flujos alternativos:**
- **FA-01 Stripe no disponible (timeout):** Si Stripe devuelve timeout → el Billing Service no actualiza el `Account`; encola el reintento con backoff exponencial (intervalo 1 s, máximo 5 intentos) conforme a NFR-03 [PRD]; el `Order` permanece en `PENDING_PAYMENT` durante el proceso de retry.
- **FA-02 Stripe rechaza el pago (fondos insuficientes):** Si Stripe retorna error de rechazo definitivo → el Billing Service publica el evento `PaymentFailed`; el Order Service cancela el `Order` y lo actualiza a estado `CANCELLED`; el consumidor recibe notificación con motivo de rechazo y enlace para actualizar método de pago.

**Postcondiciones:**
- `Account` actualizado con la transacción registrada y estado `CHARGED` o `FAILED`.
- Evento `PaymentSucceeded` o `PaymentFailed` publicado en el broker de eventos.
- `Order` actualizado a `CONFIRMED` (pago exitoso) o `CANCELLED` (pago rechazado definitivo).

**Given/When/Then:**

> **Escenario principal — cobro exitoso en primer intento**
> - **Given:** Existe un `Order` en estado `PENDING_PAYMENT`, el `payment_token` es válido y Stripe está operativo.
> - **When:** El Billing Service envía la solicitud de cobro a Stripe.
> - **Then:** Stripe retorna éxito, el `Account` registra la transacción como `CHARGED`, el evento `PaymentSucceeded` se publica y el `Order` pasa a estado `CONFIRMED`.

> **Escenario alternativo — Stripe retorna timeout, reintento exitoso en segundo intento**
> - **Given:** Existe un `Order` en `PENDING_PAYMENT` y Stripe devuelve timeout en el primer intento.
> - **When:** El Billing Service encola el reintento y lo ejecuta tras 1 segundo de backoff.
> - **Then:** El segundo intento retorna éxito; el `Account` registra la transacción como `CHARGED`, el evento `PaymentSucceeded` se publica y el `Order` pasa a `CONFIRMED`; el sistema registra el reintento en el log de trazabilidad con `correlation_id` (NFR-05 [PRD]).

---

### UC-05: Tracking en Tiempo Real del Pedido

| Campo | Valor |
|-------|-------|
| **Actor primario** | Consumidor |
| **Capacidad PRD** | CAP-05 Delivery |
| **Origen** | NFR-01 (latencia ≤ 200 ms p95) + NFR-02 (disponibilidad 99.9% Order Taking / 99.5% tracking) [PRD] |

**Precondiciones:**
- El `Delivery` existe en estado `ASSIGNED` o superior (UC-03 completado).
- El Courier tiene la app activa y transmite su ubicación GPS.
- El consumidor tiene la app abierta en la pantalla de seguimiento del pedido.

**Flujo principal:**
1. El consumidor abre la pantalla de tracking con el `order_id` activo.
2. El sistema recupera el estado actual del `Order` y la posición del courier desde el Delivery Service.
3. El sistema renderiza el mapa con la posición del courier y la ruta estimada hacia el domicilio del consumidor vía Google Maps.
4. El sistema actualiza la posición del courier en tiempo real (polling o WebSocket) con ETA recalculado.
5. Cuando el `Delivery` pasa a `PICKED_UP`, el sistema muestra el mensaje "Tu pedido está en camino".
6. Cuando el `Delivery` pasa a `DELIVERED`, el sistema muestra confirmación de entrega y solicita calificación del servicio.
7. El consumidor puede contactar al courier o reportar un problema desde esta pantalla.

**Flujos alternativos:**
- **FA-01 Degradación del servicio de tracking:** Si el Delivery Service supera la latencia de 200 ms p95 (NFR-01 [PRD]) → el sistema activa modo degradado: muestra el último estado conocido del pedido con timestamp de última actualización y mensaje "Actualizando ubicación…"; el tracking puede degradar a 99.5% de disponibilidad sin afectar el flujo transaccional (NFR-02 [PRD]).
- **FA-02 Courier pierde conectividad:** Si el courier no transmite posición por más de 60 segundos → el mapa del consumidor congela la última posición conocida con indicador de "señal débil"; el sistema genera alerta interna para el Empleado FTGO (back office) con `courier_id` y `order_id`.

**Postcondiciones:**
- El consumidor visualizó el estado final `DELIVERED` en la pantalla de tracking.
- El `Order` pasa a estado `DELIVERED` en el Order Service.
- Evento `OrderDelivered` publicado en el broker de eventos.
- Solicitud de calificación enviada al consumidor vía CAP-07 Notifications.

**Given/When/Then:**

> **Escenario principal — tracking fluido hasta entrega**
> - **Given:** El `Delivery` está en estado `ASSIGNED`, el courier transmite GPS activamente y Google Maps está operativo.
> - **When:** El consumidor abre la pantalla de tracking de su `order_id`.
> - **Then:** El sistema muestra la posición del courier actualizada en tiempo real con ETA; la respuesta inicial de la pantalla es entregada en < 200 ms p95 (NFR-01 [PRD]); cuando el courier entrega el pedido, el sistema muestra confirmación y solicita calificación.

> **Escenario alternativo — servicio de tracking en modo degradado**
> - **Given:** El Delivery Service supera la latencia de 200 ms p95 y el tracking degrada su disponibilidad.
> - **When:** El consumidor consulta el estado de su pedido.
> - **Then:** El sistema muestra el último estado conocido con timestamp de última actualización y el mensaje "Actualizando ubicación…"; el flujo transaccional del `Order` no se interrumpe (NFR-02 [PRD]); el incidente queda registrado con `correlation_id` para diagnóstico posterior (NFR-05 [PRD]).

---

_Documento trazable a: docs/PRD.md | Brief §A.5 | Richardson, Microservices Patterns, Manning 2019, Cap 3–4_
