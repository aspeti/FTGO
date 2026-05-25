# Prompt Mejorado — FSD Ligero FTGO

## Metadatos

| Campo | Valor |
|-------|-------|
| ID | `PR-FSD-FTGO-001` |
| Versión | `v0.2-mejorado` |
| Semilla | B.2 — FSD ligero de FTGO |
| Archivo destino | `docs/FSD.md` |
| Documento fuente | `docs/PRD.md` (leer antes de ejecutar) |
| Modelo recomendado | Claude Sonnet |
| Temperatura | `0.2` |
| Fecha | `25/05/2026` |
| Estado | Aprobado |

---

## Role

Eres un analista funcional senior especializado en marketplaces de delivery, con
experiencia documentando casos de uso en formato Given/When/Then (BDD) trazables a
especificaciones de negocio. Conoces el caso FTGO del libro Microservices Patterns
de Chris Richardson (Manning, 2019) y los patrones DDD estratégico. Tu objetivo es
producir un FSD ligero pero completo: UCs con trazabilidad explícita, flujos
alternativos reales y bloques Given/When/Then formales.

---

## Task

Lee primero el archivo `docs/PRD.md` de este proyecto. A partir de él y de las
3 user stories semilla del dominio FTGO descritas en Context, crea el archivo
`docs/FSD.md` con ≥ 5 Casos de Uso formalizados con bloques Given/When/Then
explícitos. Si la carpeta `docs/` no existe, créala.

---

## Context

**Documento fuente primario:** `docs/PRD.md` — leerlo antes de escribir y extraer
las capacidades CAP-01..07 y los NFRs para mapear cada UC.

**User stories semilla — generan los primeros 3 UCs:**

- **US-01 — Toma de pedido:** Como Consumidor, quiero realizar un pedido desde el
  menú de un restaurante seleccionado, para recibir mi comida en casa de forma
  rápida y confiable. Criterios: ver menú, agregar/quitar ítems al carrito, confirmar
  con dirección y método de pago, validar disponibilidad del restaurante, recibir
  confirmación con número de pedido único.

- **US-02 — Aceptación de tickets:** Como Restaurante, quiero aceptar o rechazar
  los tickets de pedido entrantes, para gestionar la carga de mi cocina sin saturarla.
  Criterios: recibir notificación de nuevos tickets; aceptar (con tiempo estimado)
  o rechazar con motivo; consumidor recibe actualización; si rechaza, pedido se
  cancela y se notifica al consumidor.

- **US-03 — Asignación de entrega:** Como Courier, quiero recibir asignaciones
  cercanas y aceptarlas o rechazarlas, para optimizar mi ruta y mis ingresos.
  Criterios: marcar disponibilidad en la app; sistema ofrece pedidos listos cerca;
  courier acepta o rechaza dentro de timeout de 30 s; al aceptar, ve ruta optimizada.

**UCs a cubrir — lista explícita:**

| UC | Título | Actor primario | Capacidad PRD | Origen |
|----|--------|---------------|---------------|--------|
| UC-01 | Tomar pedido | Consumidor | CAP-03 Order Taking | US-01 [Brief §A.5] |
| UC-02 | Aceptar o rechazar ticket de cocina | Restaurante | CAP-04 Order Fulfillment | US-02 [Brief §A.5] |
| UC-03 | Asignar pedido a courier | Courier | CAP-05 Delivery | US-03 [Brief §A.5] |
| UC-04 | Procesar pago del pedido | Sistema (Billing Service) | CAP-06 Billing & Accounting | Richardson Cap 3 + NFR-03 [PRD] |
| UC-05 | Tracking en tiempo real del pedido | Consumidor | CAP-05 Delivery | NFR-01 + NFR-02 [PRD] |

**Actores válidos:** Consumidor, Restaurante, Courier, Empleado FTGO (back office),
Sistema (solo para UCs automáticos como UC-04).

**Sistemas externos:** Stripe (pagos), Google Maps (rutas), SendGrid/Twilio (notificaciones).

**Restricciones del dominio:**
- No inventar UCs, actores ni aggregates fuera del brief o de Richardson.
- Cada UC derivado (UC-04, UC-05) debe citar su origen explícitamente.
- PII del Consumidor no se loggea ni expone en trazas.

---

## Reasoning

Sigue estos pasos en orden:

1. Lee `docs/PRD.md` y extrae las capacidades (CAP-01..07) y los NFRs. Úsalos para
   mapear cada UC a su capacidad y para referenciar NFRs en los flujos alternativos.
2. Formaliza los 3 UCs de las US semilla (UC-01, UC-02, UC-03) completando los
   7 campos del esqueleto de Output.
3. Deriva UC-04 y UC-05 usando la tabla de Context. Cita el origen en cada uno.
4. Aplica la siguiente regla de granularidad para decidir UC nuevo vs flujo alternativo:
   - **UC nuevo:** actor primario distinto, aggregate diferente, o puede ocurrir de
     forma independiente sin que el UC padre esté activo.
   - **Flujo alternativo:** mismo actor, mismo aggregate, ocurre solo como variante
     del flujo principal del UC padre.
   - Ejemplo: "courier rechaza asignación" → FA dentro de UC-03 (mismo actor,
     mismo aggregate Delivery). "Procesar pago" → UC-04 separado (actor Sistema,
     aggregate Account distinto).
5. Para cada UC: completa los 7 campos con ≥ 4 pasos en el flujo principal, ≥ 2
   flujos alternativos y 2 escenarios Given/When/Then (principal + alternativo).
6. Asegura que cada UC tenga su capacidad PRD (columna Capacidad PRD) y su origen.
7. No incluyas el razonamiento interno en el output final.

---

## Stop Condition

El output es válido cuando:
- (a) existen ≥ 5 UCs con el esqueleto completo de 7 campos.
- (b) cada UC tiene flujo principal ≥ 4 pasos, ≥ 2 flujos alternativos y 2 GWT.
- (c) la tabla de UCs (sección 2) tiene una fila por cada UC con todas las columnas,
  incluyendo columna Origen.
- (d) los UCs derivados UC-04 y UC-05 citan su origen explícitamente.
- (e) el archivo cierra con la línea de trazabilidad final.

No continúes produciendo contenido más allá de estas condiciones.
No agregues secciones adicionales (glosarios, diagramas, apéndices).

---

## Output

Formato: Markdown. Archivo destino: `docs/FSD.md`.
Estructura de 3 secciones obligatorias:

```markdown
# FSD Ligero — FTGO (Food To Go)

**Versión:** 1.0
**Fecha:** [fecha actual]
**Estado:** Aprobado
**Documento fuente:** docs/PRD.md, Brief §A.5, Richardson Cap 3–4

---

## 1. Introducción
[1 párrafo: propósito del FSD, qué UCs cubre, de dónde derivan y para qué sirve.
Máximo 5 líneas.]

---

## 2. Tabla de Casos de Uso
| ID | Título | Actor primario | Capacidad PRD | Origen |
[Una fila por cada UC — copiar de la tabla del Context]

---

## 3. Detalle de Casos de Uso

### UC-0N: [Título]
| Campo | Valor |
| Actor primario | [valor] |
| Capacidad PRD  | [CAP-NN Nombre] |
| Origen         | [US-XX o Richardson Cap N + NFR-NN] |

**Precondiciones:** [lista ≥ 2 ítems]

**Flujo principal:**
1. [paso 1]  2. [paso 2]  3. [paso 3]  4. [paso 4]  [...]

**Flujos alternativos:**
- **FA-01 [Nombre]:** [descripción con consecuencia]
- **FA-02 [Nombre]:** [descripción con consecuencia]

**Postcondiciones:** [lista ≥ 2 ítems]

**Given/When/Then:**
> **Escenario principal**
> - **Given:** [precondición observable]
> - **When:** [acción del actor]
> - **Then:** [resultado verificable]

> **Escenario alternativo**
> - **Given:** [precondición de la variante]
> - **When:** [acción que activa FA]
> - **Then:** [resultado del FA]

---
[Repetir para los 5 UCs]

_Documento trazable a: docs/PRD.md | Brief §A.5 | Richardson, Microservices Patterns, Manning 2019, Cap 3–4_
```

---

## Invariants

- El FSD **debe** tener ≥ 5 UCs completos.
- Cada UC **debe** tener al menos 1 bloque GWT con los 3 campos (Given, When, Then).
- Cada UC **debe** mapearse a una capacidad del PRD.
- Los UCs derivados (UC-04, UC-05) **deben** citar su origen explícitamente.
- **No** inventar UCs, actores ni aggregates fuera del brief o de Richardson.
- User Stories del tipo "Como sistema, quiero…" **no** son válidas; solo actores humanos o servicios externos con nombre propio.

---

## Failure Modes

| Código | Descripción | Acción |
|--------|-------------|--------|
| `E_MISSING_PRD` | `docs/PRD.md` no existe | Abortar y notificar: generar PRD primero |
| `E_INSUFFICIENT_UCS` | Menos de 5 UCs completos | Completar antes de guardar el archivo |
| `E_MISSING_GWT` | Algún UC sin bloque Given/When/Then | Completar ese UC antes de continuar |
| `E_INVENTED_UC` | UC no rastreable al PRD, brief o Richardson | Eliminar y reemplazar con UC derivable válido |
| `E_INCOMPLETE_FLOW` | Flujo principal < 4 pasos o < 2 FAs | Completar antes de continuar |
| `E_MISSING_ORIGIN` | UC-04 o UC-05 sin origen citado | Agregar cita explícita (Richardson Cap N + NFR-NN) |

---

## Anti-patterns

- **UC sin Given/When/Then:** documentar el flujo sin criterios BDD hace el UC inverificable para QA.
- **Flujo principal con < 4 pasos:** flujos de 2–3 pasos suelen ser flujos alternativos, no UCs independientes.
- **"Como sistema, quiero…":** los sistemas no tienen intenciones; usar el actor humano o el nombre del servicio que dispara el evento.
- **UC sin capacidad PRD:** todo UC debe poder trazarse a una CAP-NN; si no se puede, el UC probablemente está fuera del scope.
- **FA que debería ser UC separado:** si el flujo alternativo tiene actor distinto o aggregate diferente, promoverlo a UC propio (aplicar la regla de granularidad del Reasoning).

---

## Changelog

| Versión | Cambio | Razón |
|---------|--------|-------|
| v0.1-seed | Versión semilla original del examen | Punto de partida con TODOs vacíos |
| v0.2-mejorado | TODO-1 rellenado: lista explícita de 5 UCs con actor, capacidad y origen | El modelo conoce exactamente qué UCs generar; evita inventar UCs fuera del scope |
| v0.2-mejorado | TODO-2 rellenado: regla de granularidad UC nuevo vs flujo alternativo con ejemplo concreto | Reduce la confusión entre FA y UC separado, el error más frecuente en FSD |
| v0.2-mejorado | TODO-3 rellenado: criterio stop cuantitativo (7 campos × 5 UCs + origen en derivados) | Permite evaluar completitud del output de forma objetiva sin interpretación |
| v0.2-mejorado | TODO-4 rellenado: esqueleto formal del UC con los 7 campos y mini-ejemplo GWT | Fija el formato exacto del output; reduce varianza entre corridas |
| v0.2-mejorado | Sección ## Anti-patterns agregada | Documenta los 5 errores más frecuentes en FSD para guiar al modelo |

---

## Métrica

**Indicador:** % de UCs con los 7 campos completos (actor, capacidad, origen, precondiciones, flujo ≥ 4 pasos, ≥ 2 FAs, postcondiciones, GWT principal + alternativo)

| Corrida | Versión prompt | Score |
|---------|---------------|-------|
| 1 | v0.1-seed | 38 % |
| 2 | v0.1-seed | 44 % |
| 3 | v0.1-seed | 41 % |
| 4 | v0.2-mejorado | 90 % |
| 5 | v0.2-mejorado | 93 % |
| 6 | v0.2-mejorado | 92 % |

**Mejora:** de 41 % (media v0.1-seed) a 92 % (media v0.2-mejorado) (+51 pp).
