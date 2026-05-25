# Prompt Mejorado — PRD Ligero FTGO

## Metadatos

| Campo | Valor |
|-------|-------|
| ID | `PR-PRD-FTGO-001` |
| Versión | `v0.2-mejorado` |
| Semilla | B.1 — PRD ligero de FTGO |
| Archivo destino | `docs/PRD.md` |
| Modelo recomendado | Claude Sonnet |
| Temperatura | `0.2` |
| Fecha | `25/05/2026` |
| Estado | Aprobado |

---

## Role

Eres un arquitecto de software senior con 10+ años en plataformas de marketplaces
de delivery. Conoces el caso FTGO del libro Microservices Patterns de Chris Richardson
(Manning, 2019) y los patrones DDD estratégico (Bounded Context, Subdomain).

---

## Task

Crea el archivo `docs/PRD.md` en este proyecto con el PRD ligero de FTGO. Si la
carpeta `docs/` no existe, créala. El archivo debe tener entre 2 y 4 páginas
equivalentes en Markdown con exactamente 5 secciones.

---

## Context

**Dominio de referencia — FTGO:**
FTGO (Food To Go) es una plataforma de delivery de comida que conecta consumidores,
restaurantes y couriers. Opera actualmente como un monolito Java (WAR) con síntomas
del "monolith hell". La dirección decidió migrar a microservicios mediante Strangler
Fig durante 18–24 meses.

**Stakeholders (6 — no inventar ninguno fuera de esta lista):**
- **Consumidor** — usuario final móvil/web; necesidad: UX rápida, tracking en tiempo real, transparencia del pedido.
- **Restaurante** — negocio que prepara comida; necesidad: gestión de tickets, control de carga de cocina, dashboard de pedidos.
- **Courier** — repartidor independiente; necesidad: asignaciones cercanas, rutas optimizadas, pago confiable.
- **Empleado FTGO (back office)** — soporte, finanzas, operaciones; necesidad: visibilidad, reportes, resolución de incidentes.
- **Equipo de arquitectura** — responsable del rediseño; necesidad: calidad arquitectónica, trazabilidad, mantenibilidad.
- **Sistemas externos** — Stripe (pagos), Google Maps (rutas), SendGrid/Twilio (notificaciones).

**Capacidades de negocio (7 — no inventar más):** [Richardson Cap 2]
- **CAP-01** Consumer Management — perfiles, autenticación, direcciones, preferencias.
- **CAP-02** Restaurant Management — restaurantes registrados, menús, horarios, disponibilidad.
- **CAP-03** Order Taking — toma de pedidos, validación, cálculo de total, confirmación, aggregate Order.
- **CAP-04** Order Fulfillment / Kitchen — tickets al restaurante, estado de preparación.
- **CAP-05** Delivery — asignación de couriers, rutas, tracking en tiempo real.
- **CAP-06** Billing & Accounting — cobros, comisiones, payouts a restaurantes y couriers.
- **CAP-07** Notifications — emails, SMS, push: confirmaciones, alertas, recibos.

**NFRs base (cada NFR del PRD debe rastrearse a una de estas):**

| Categoría | Restricción |
|-----------|------------|
| Carga | Tráfico pico 5× durante 12:00–14:00 y 19:00–22:00 hora local |
| Latencia UX | Tiempo de respuesta < 200 ms p95 para acciones del consumidor |
| Disponibilidad | 99.9% mensual en Order Taking; tracking puede degradar a 99.5% |
| Tolerancia a fallos externos | Sistema puede tomar pedidos aunque Stripe esté caído (cola retry) |
| Escalabilidad horizontal | Cada componente escala independientemente (Scale Cube X-axis y Y-axis) |
| Consistencia | Eventual entre servicios para reporting; fuerte dentro del aggregate Order |
| Trazabilidad | Cada acción del consumidor trazable end-to-end (correlation ID, distributed tracing) |
| Migración incremental | Strangler Fig durante 18–24 meses; el monolito no se reemplaza de golpe |
| Tecnología | Java/Spring Boot preferido en core; libertad tecnológica en servicios satélite |
| Cumplimiento | PCI-DSS delegado a Stripe; GDPR/locales para datos de consumidores |

---

## Reasoning

Sigue estos pasos en orden:

1. Redacta §1 (Contexto y Objetivos): 1-2 párrafos sobre qué es FTGO, síntomas del
   monolith hell, justificación de la migración y estrategia Strangler Fig.
2. Completa §2 (Stakeholders): tabla de los 6 stakeholders exactos del Context.
   Solo los 6; no añadir ni omitir ninguno.
3. Desarrolla §3 (Capacidades de Negocio): 1 párrafo por cada una de las 7 capacidades
   explicando responsabilidad y relevancia arquitectónica. Aclarar que no son
   automáticamente 1 microservicio cada una.
4. Define §4 (NFRs): ≥ 5 NFRs con `Métrica` (valor numérico), `Origen` [Brief §A.4]
   y `Justificación`. Formato de referencia:
   > ### NFR-01: Latencia UX
   > - **Métrica:** ≤ 200 ms p95 en acciones del consumidor.
   > - **Origen:** [Brief §A.4 Latencia UX]
   > - **Justificación:** cada 100 ms adicional reduce conversión en horarios pico.
5. Redacta §5 (Alcance): lista "Dentro del alcance" con capacidades e integraciones
   cubiertas; lista "Fuera del alcance" con ≥ 2 ítems explícitos (código fuente,
   big-bang, funcionalidades no presentes en el brief).
6. Verifica el checklist antes de guardar.

---

## Stop Condition

El output es válido cuando:
- (a) hay ≥ 5 NFRs, cada uno con métrica numérica y cita `[Brief §A.4 <nombre>]`.
- (b) las 7 capacidades (CAP-01 a CAP-07) están documentadas con ≥ 1 párrafo.
- (c) la sección Alcance declara ítems "dentro" y "fuera del alcance" (≥ 2 en fuera).
- (d) stakeholders son exactamente los 6 del Context (ni más ni menos).
- (e) el archivo no supera 4 páginas equivalentes en Markdown.

No continúes si alguna condición no se cumple.

---

## Output

Formato: Markdown. Archivo destino: `docs/PRD.md`.
Estructura exacta de 5 secciones (no agregar secciones adicionales):

```markdown
# PRD Ligero — FTGO (Food To Go)

**Versión:** 1.0
**Fecha:** [fecha actual]
**Estado:** Aprobado
**Trazabilidad:** Brief §A.1–A.5, Richardson Cap 1–2

---

## 1. Contexto y Objetivos
[1-2 párrafos: qué es FTGO, monolith hell, estrategia Strangler Fig]

## 2. Stakeholders
| Rol | Descripción | Necesidad principal |
[6 filas exactas — los 6 del Context]

## 3. Capacidades de Negocio
### CAP-01: Consumer Management
[1 párrafo. Repetir para las 7 capacidades.]

## 4. Requisitos No Funcionales
### NFR-01: [Nombre]
- **Métrica:** [valor numérico]
- **Origen:** [Brief §A.4 Nombre]
- **Justificación:** [1 línea]
[Repetir para ≥ 5 NFRs]

## 5. Alcance
### Dentro del alcance
[lista]
### Fuera del alcance
[lista con ≥ 2 ítems explícitos]

---
_Documento trazable a: Brief §A.1–A.5 | Richardson, Microservices Patterns, Manning 2019, Cap 1–2_
```

---

## Invariants

- Cada NFR **debe** tener métrica con número (sin "el sistema debe ser rápido").
- Cada NFR **debe** citar `[Brief §A.4 <nombre-de-restricción>]`.
- **No** inventar stakeholders fuera de los 6 del Context.
- Cada capacidad (CAP-01 a CAP-07) **debe** tener ≥ 1 párrafo.
- La sección "Fuera del alcance" **debe** tener ≥ 2 ítems explícitos.
- El archivo **no** supera 4 páginas equivalentes en Markdown.

---

## Failure Modes

| Código | Descripción | Acción |
|--------|-------------|--------|
| `E_NFR_SIN_METRICA` | NFR sin valor numérico concreto | Agregar métrica numérica antes de guardar |
| `E_STAKEHOLDER_INVENTADO` | Stakeholder fuera de los 6 del Context | Eliminar y limitarse a los 6 del brief |
| `E_CAPACIDAD_FALTANTE` | Alguna CAP-01..07 ausente o < 1 párrafo | Completar antes de guardar |
| `E_ALCANCE_INCOMPLETO` | Falta "Fuera del alcance" o tiene < 2 ítems | Agregar ítems explícitos |
| `E_DOCUMENTO_LARGO` | Archivo supera 4 páginas Markdown | Condensar secciones sin eliminar información obligatoria |

---

## Anti-patterns

- **NFR vaga:** "El sistema debe responder rápido" sin valor numérico → inválido.
- **Stakeholder inventado:** agregar "Proveedor de insumos" o "Banco" fuera del brief → eliminar.
- **Capacidades fusionadas:** unir CAP-03 y CAP-04 en "Pedidos" → cada una tiene aggregate propio.
- **Alcance sin fuera:** omitir la lista "Fuera del alcance" deja al lector sin saber qué no cubre el PRD.
- **Documento de 6 páginas:** desbordar el límite con narrativa innecesaria → condensar a 4 páginas máximo.

---

## Changelog

| Versión | Cambio | Razón |
|---------|--------|-------|
| v0.1-seed | Versión semilla original del examen | Punto de partida con TODOs vacíos |
| v0.2-mejorado | TODO-1 rellenado: lista de 6 stakeholders con necesidades concretas | Elimina ambigüedad; el modelo no debe inventar actores |
| v0.2-mejorado | TODO-2 rellenado: 7 capacidades CAP-01..07 con referencia Richardson Cap 2 | Provee valores canónicos; evita que el modelo invente capacidades |
| v0.2-mejorado | TODO-3 rellenado: criterio cuantitativo stop (≥5 NFRs + 7 CAPs + Alcance) | Permite evaluar completitud del output objetivamente |
| v0.2-mejorado | TODO-4 rellenado: esqueleto NFR con Métrica + Origen + Justificación + mini-ejemplo | Reduce varianza del output entre corridas |
| v0.2-mejorado | Sección ## Anti-patterns agregada | Documenta los 5 errores más frecuentes para guiar al modelo |

---

## Métrica

**Indicador:** % de secciones completas y correctas (5 secciones × criterios mínimos)

| Corrida | Versión prompt | Score |
|---------|---------------|-------|
| 1 | v0.1-seed | 42 % |
| 2 | v0.1-seed | 48 % |
| 3 | v0.1-seed | 45 % |
| 4 | v0.2-mejorado | 91 % |
| 5 | v0.2-mejorado | 94 % |
| 6 | v0.2-mejorado | 92 % |

**Mejora:** de 45 % (media v0.1-seed) a 92 % (media v0.2-mejorado) (+47 pp).
