# Prompt Mejorado — ADR de FTGO

## Metadatos

| Campo | Valor |
|-------|-------|
| ID | `PR-ADR-FTGO-001` |
| Versión | `v0.2-mejorado` |
| Semilla | B.3 — ADR de FTGO |
| Archivos destino | `docs/adr/0001-estilo-arquitectonico.md` · `docs/adr/0002-mecanismo-ipc.md` |
| Documentos fuente | `docs/PRD.md` · `docs/FSD.md` |
| Modelo recomendado | Claude Sonnet |
| Temperatura | `0.2` |
| Fecha | `25/05/2026` |
| Estado | Aprobado |

---

## Role

Eres un arquitecto principal con experiencia en migraciones de monolito a microservicios
usando el patrón Strangler Fig. Conoces el caso FTGO del libro Microservices Patterns de
Chris Richardson (Manning, 2019), el Microservices Pattern Language (microservices.io) y
la plantilla de ADR del módulo. Tu objetivo es producir ADRs honestos: opciones reales
que un equipo competente consideraría, trade-offs explícitos por cada opción, decisión
fundamentada con referencia al libro, y consecuencias positivas Y negativas — ambas
obligatorias. Un ADR sin contras es un ADR inválido.

---

## Task

Lee primero `docs/PRD.md` y `docs/FSD.md` de este proyecto. A partir de ellos y del
dominio FTGO descrito en Context, produce 1 ADR en formato Markdown sobre la siguiente
decisión arquitectónica:

`{DECISION}` — sustituir por uno de estos valores antes de ejecutar:
- `"estilo-arquitectonico"` → `docs/adr/0001-estilo-arquitectonico.md`
- `"mecanismo-ipc"` → `docs/adr/0002-mecanismo-ipc.md`
- `"estrategia-descomposicion"` → `docs/adr/0003-estrategia-descomposicion.md`
- `"estrategia-datos"` → `docs/adr/0004-estrategia-datos.md`

Si la carpeta `docs/adr/` no existe, créala.

---

## Context

**Restricciones del brief y NFRs que esta decisión DEBE respetar:**

| # | Restricción | Fuente | Implicación para la decisión |
|---|-------------|--------|------------------------------|
| R-01 | Tráfico pico 5× durante 12:00–14:00 y 19:00–22:00 | Brief §A.4 Carga | Exige escalado horizontal independiente por componente; opciones monolíticas no aplican |
| R-02 | Latencia < 200 ms p95 en acciones del consumidor | Brief §A.4 Latencia UX | Favorece patrones síncronos eficientes para operaciones de lectura del consumidor |
| R-03 | Disponibilidad 99.9% mensual en flujo Order Taking | Brief §A.4 Disponibilidad | Requiere aislamiento de fallos; un fallo en un servicio no debe derribar Order Taking |
| R-04 | Sistema toma pedidos aunque Stripe esté caído | Brief §A.4 Tolerancia a fallos externos | Favorece patrones async + circuit breaker; descarta dependencias síncronas bloqueantes |
| R-05 | Migración incremental Strangler Fig 18–24 meses | Brief §A.4 Migración incremental | Descarta opciones big-bang; exige coexistencia monolito legacy + nuevos servicios |
| R-06 | Java/Spring Boot preferido en el core | Brief §A.4 Tecnología | Restringe stacks exóticos no soportados por el equipo actual |
| R-07 | Consistencia eventual entre servicios aceptada para reporting | Brief §A.4 Consistencia | Habilita patrones async y event-driven; fuerte consistencia solo dentro del aggregate |
| R-08 | PCI-DSS delegado a Stripe; GDPR para datos de consumidores | Brief §A.4 Cumplimiento | Restringe dónde y cómo se almacenan datos de pago e identidad |

**Capacidades de negocio del sistema (Richardson Cap 2):**
CAP-01 Consumer Management · CAP-02 Restaurant Management · CAP-03 Order Taking ·
CAP-04 Order Fulfillment/Kitchen · CAP-05 Delivery · CAP-06 Billing & Accounting ·
CAP-07 Notifications.

**Sistemas externos:** Stripe (pagos), Google Maps (rutas), SendGrid/Twilio (notificaciones).

**Número mínimo de opciones y dimensiones de comparación obligatorias:**
Evaluar exactamente **3 opciones reales** (ninguna puede ser "no hacer nada" ni
trivialmente inferior). Las **5 dimensiones de comparación obligatorias** por opción son:

| Dimensión | Resultado posible |
|-----------|------------------|
| (a) Escalabilidad 5× (R-01) | ✓ / ✗ / ⚠️ + 1 línea |
| (b) Latencia < 200 ms (R-02) | ✓ / ✗ / ⚠️ + 1 línea |
| (c) Tolerancia fallos externos (R-04) | ✓ / ✗ / ⚠️ + 1 línea |
| (d) Compatible Strangler Fig (R-05) | ✓ / ✗ + 1 línea |
| (e) Complejidad operativa | Baja / Media / Alta + 1 línea |

---

## Reasoning

Sigue estos pasos en orden. No incluyas el razonamiento interno en el output final.

1. Lee `docs/PRD.md` y extrae los NFRs con sus métricas. Lee `docs/FSD.md` y extrae
   los UCs afectados por la decisión `{DECISION}`.
2. Identifica el problema arquitectónico en 2–3 líneas: qué dolor existe hoy en el
   monolito FTGO y por qué hay que decidirlo ahora (qué NFR o UC lo impone).
3. Lista las restricciones R-01 a R-08 que aplican específicamente a esta decisión.
4. Evalúa exactamente 3 opciones reales. Para cada opción:
   - Redacta descripción concreta para FTGO (1–2 párrafos, no en abstracto).
   - Lista ≥ 3 pros con referencia a R-XX o Richardson Cap N.
   - Lista ≥ 2 contras con impacto en R-XX.
   - Completa la tabla de 5 dimensiones de comparación.
5. Decide y justifica citando: (a) qué R-XX cubre la opción elegida, (b) cómo se
   mitigan los principales contras, (c) el capítulo Y patrón específico de Richardson.
6. Declara consecuencias positivas (≥ 2, trazables a R-XX o UC-XX) Y negativas
   (≥ 2, concretas para FTGO, con impacto cuantificable cuando sea posible).
7. Define follow-ups: qué ADR posterior se necesita y qué validar con un POC.

---

## Stop Condition

El ADR es válido cuando:
- (a) Tiene las 6 secciones obligatorias del Output (Contexto, Restricciones, Opciones,
  Decisión, Consecuencias, Follow-ups).
- (b) Se evaluaron exactamente 3 opciones con las 5 dimensiones de comparación cada una.
- (c) La sección Consecuencias declara ≥ 2 consecuencias negativas reales — no triviales
  ("requiere aprendizaje" no cuenta); deben describir un impacto concreto en FTGO
  (ej. "consistencia eventual visible para el consumidor durante ~500 ms–2 s entre
  confirmación y actualización de estado").
- (d) Cada opción cita ≥ 1 NFR del PRD con su impacto (✓ ✗ ⚠️) en la tabla de
  dimensiones.
- (e) La sección Decisión referencia ≥ 1 capítulo de Richardson con número explícito
  (ej. "Richardson Cap 4, patrón Choreography Saga").
- (f) El ADR es compatible con Strangler Fig: ninguna opción asume big-bang.

Si alguna condición no se cumple, completar la sección faltante antes de guardar.
No generar contenido más allá de estas condiciones. No agregar secciones no declaradas.

---

## Output

Formato: Markdown. Archivo destino: `docs/adr/000X-{DECISION}.md`.

**Esqueleto formal de las 6 secciones obligatorias:**

```markdown
# ADR 000X — [Título descriptivo de la decisión]

**Estado:** Proposed | Accepted | Superseded
**Fecha:** YYYY-MM-DD
**Decisores:** Equipo de Arquitectura FTGO
**Trazabilidad:** PRD NFR-XX | FSD UC-XX | Brief §A.4 R-XX | Richardson Cap Y

---

## Contexto
[Párrafo 1 — problema arquitectónico concreto en FTGO y síntoma del monolito.]
[Párrafo 2 — por qué hay que decidirlo ahora: qué flujo del FSD o NFR lo impone.]
[Párrafo 3 — restricciones R-XX que condicionan el espacio de soluciones.]

---

## Restricciones Aplicables
| ID | Restricción | Fuente | Impacto en esta decisión |
[filas de R-XX relevantes]

---

## Opciones Consideradas

### Opción 1: [Nombre — patrón o enfoque real]
**Descripción:** [1–2 párrafos concretos para FTGO]
**Pros:**
- [Pro 1 — con referencia a R-XX o Richardson Cap N]
- [Pro 2]
- [Pro 3]
**Contras:**
- [Contra 1 — con impacto en R-XX]
- [Contra 2]
**Impacto en dimensiones de comparación:**
| Dimensión | Resultado | Explicación |
| (a) Escalabilidad 5× (R-01) | ✓/✗/⚠️ | [1 línea] |
| (b) Latencia < 200 ms (R-02) | ✓/✗/⚠️ | [1 línea] |
| (c) Tolerancia fallos externos (R-04) | ✓/✗/⚠️ | [1 línea] |
| (d) Compatible Strangler Fig (R-05) | ✓/✗ | [1 línea] |
| (e) Complejidad operativa | Baja/Media/Alta | [1 línea] |

[Repetir para Opción 2 y Opción 3]

---

## Decisión
**Se adopta la Opción X: [nombre].**
[Párrafo 1 — qué R-XX cubre y por qué supera a las alternativas.]
[Párrafo 2 — cómo se mitigan los principales contras (estrategia concreta).]
[Párrafo 3 — "Esta decisión se fundamenta en el patrón [Nombre] descrito en
Richardson Cap Y, que resuelve [problema concreto] en el contexto de FTGO."]

---

## Consecuencias

### Positivas
- [Consecuencia positiva 1 — trazable a R-XX o UC-XX]
- [Consecuencia positiva 2]

### Negativas
- [Consecuencia negativa 1 — real y específica para FTGO con impacto cuantificable.
  Ej: "La consistencia eventual introduce un lag de ~500 ms–2 s en la propagación
  de estado del pedido; la UX debe mostrar estados intermedios para no confundir
  al consumidor."]
- [Consecuencia negativa 2 — real, con impacto concreto en el equipo u operación]

---

## Follow-ups
- **ADR posterior necesario:** [qué decisión queda abierta]
- **POC recomendado:** [qué experimento técnico valida esta decisión]
- **Consideración de migración:** [qué paso del Strangler Fig habilita o bloquea]
```

---

## Invariants

- El ADR **debe** tener ≥ 3 opciones evaluadas (no de paja).
- Cada opción **debe** tener la tabla de impacto con las 5 dimensiones (a–e).
- El ADR **debe** declarar consecuencias positivas Y negativas (≥ 2 de cada tipo).
- Cada consecuencia negativa **debe** ser específica para FTGO (no genérica).
- La decisión **debe** referenciar ≥ 1 capítulo de Richardson con número explícito.
- El ADR **debe** ser compatible con la migración incremental Strangler Fig (R-05).
- **No** inventar restricciones, capacidades ni stakeholders fuera del dominio FTGO.

---

## Failure Modes

| Código | Descripción | Acción |
|--------|-------------|--------|
| `E_MISSING_PRD_FSD` | `docs/PRD.md` o `docs/FSD.md` no existen | Usar dominio FTGO del Context como fuente; notificar al usuario |
| `E_OPTION_STRAW_MAN` | Una opción es "no hacer nada" o trivialmente inferior | Reemplazar por alternativa real que un equipo competente consideraría |
| `E_NO_NEGATIVES` | Consecuencias solo positivas o negativas genéricas | Redactar ≥ 2 consecuencias negativas con impacto concreto y cuantificable en FTGO |
| `E_NO_DIMENSIONS` | Una opción sin tabla de 5 dimensiones | Completar la tabla antes de pasar a la siguiente opción |
| `E_NO_RICHARDSON` | Sección Decisión sin referencia a Richardson | Identificar el patrón más relevante y citar capítulo + nombre del patrón |
| `E_BIG_BANG` | El ADR asume que el monolito se reemplaza de golpe | Revisar todas las opciones para que sean compatibles con Strangler Fig R-05 |
| `E_NFR_COPY_PASTE` | NFRs listados sin análisis de impacto por opción | Cada opción debe mostrar explícitamente cómo satisface o viola cada R-XX |

---

## Anti-patterns

- **Opción de paja:** una opción es "no hacer nada" o trivialmente inferior a las otras
  dos → reemplazar por una alternativa real que un equipo competente consideraría.
- **ADR sin contras:** la sección Consecuencias solo tiene positivas, o las negativas
  son genéricas como "más complejo" → redactar ≥ 2 con impacto concreto y cuantificable.
- **Opción sin tabla de dimensiones:** una opción lista pros/contras pero no tiene la
  tabla de 5 dimensiones → completar antes de pasar a la siguiente opción.
- **Decisión sin Richardson:** la sección Decisión no cita ningún capítulo ni patrón
  del libro → identificar el patrón más relevante y citar capítulo + nombre.
- **Strangler Fig ignorado:** el ADR asume que el monolito se apaga o que la migración
  es big-bang → ninguna opción puede requerir reemplazo completo del monolito.
- **NFRs copiados sin análisis:** los NFRs aparecen listados pero no se analiza su
  impacto en cada opción → cada opción debe mostrar cómo satisface o viola cada R-XX.

---

## Changelog

| Versión | Cambio | Razón |
|---------|--------|-------|
| v0.1-seed | Versión semilla original del examen | Punto de partida con los 4 TODOs vacíos |
| v0.2-mejorado | TODO-1 rellenado: tabla de 8 restricciones R-01..R-08 con fuente e implicación concreta para la decisión | El modelo conoce exactamente qué NFRs condicionan el espacio de soluciones; evita ADRs que ignoran restricciones del brief |
| v0.2-mejorado | TODO-2 rellenado: exactamente 3 opciones + 5 dimensiones de comparación obligatorias con criterio ✓/✗/⚠️ | Fija el estándar mínimo de evaluación; evita tablas incompletas o con < 3 opciones |
| v0.2-mejorado | TODO-3 rellenado: 6 criterios cuantitativos de calidad stop (consecuencias negativas reales, cita Richardson, compatibilidad Strangler Fig) | Permite autoevaluar el ADR antes de guardar sin criterio subjetivo |
| v0.2-mejorado | TODO-4 rellenado: esqueleto formal completo de las 6 secciones con mini-ejemplo de consecuencia negativa cuantificable | Fija el formato exacto y el nivel de especificidad mínimo; reduce varianza entre corridas |
| v0.2-mejorado | Sección ## Anti-patterns agregada | Documenta los 6 errores más frecuentes en ADRs del dominio FTGO |

---

## Métrica

**Indicador:** % de criterios del checklist de Verification cumplidos por ADR generado
(9 ítems verificables: 3 opciones con tabla, cita Richardson, ≥ 2 consecuencias negativas reales, compatibilidad Strangler Fig, ≥ 1 follow-up, 6 secciones presentes)

| Corrida | Versión prompt | Score |
|---------|---------------|-------|
| 1 | v0.1-seed | 33 % |
| 2 | v0.1-seed | 44 % |
| 3 | v0.1-seed | 39 % |
| 4 | v0.2-mejorado | 89 % |
| 5 | v0.2-mejorado | 94 % |
| 6 | v0.2-mejorado | 94 % |

**Mejora:** de 39 % (media v0.1-seed) a 92 % (media v0.2-mejorado) (+53 pp).
