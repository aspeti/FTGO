# FTGO Architecture Documentation — Examen Módulo 4

**Maestrante:** Rodrigo Aspeti  
**Branch:** `release/exam-lab`  
**Fecha de entrega:** 2026-05-22  
**Caso de estudio:** FTGO (Food To Go) — Richardson, Microservices Patterns, Manning 2019

---

## Estructura del repositorio

```
├── README.md
├── docs/
│   ├── PRD.md                    ← Artefacto 1: PRD ligero (5 secciones, 6 NFRs, 7 CAPs)
│   ├── FSD.md                    ← Artefacto 2: FSD con 5 UCs Given/When/Then
│   ├── adr/
│   │   ├── 0001-estilo-arquitectonico.md  ← Artefacto 3: ADR estilo (3 opciones, Richardson Cap 2)
│   │   └── 0002-mecanismo-ipc.md          ← Artefacto 4: ADR IPC (3 opciones, Richardson Cap 3–4)
│   └── diagrams/
│       ├── c4_context.mmd                 ← Artefacto 5: C4 Nivel 1 (4 Person, 5 System_Ext)
│       └── c4_container.mmd               ← Artefacto 6: C4 Nivel 2 (7 svc, 6 DB, 1 Kafka)
└── prompts_mejorados/
    ├── prd_mejorado.md           ← Artefacto 7a: Prompt PRD mejorado (v0.2)
    ├── fsd_mejorado.md           ← Artefacto 7b: Prompt FSD mejorado (v0.2)
    ├── c4_mejorado.md            ← Artefacto 7c: Prompt C4 mejorado (v0.2)
    └── adr_mejorado.md           ← Artefacto 7d: Prompt ADR mejorado (v0.2)
```

> Todos los artefactos del examen están completos. Verificar manualmente el branch antes de entregar.

---

## Trazabilidad de artefactos

| Artefacto | Fuente principal | Trazabilidad |
|-----------|-----------------|--------------|
| `docs/PRD.md` | Brief §A.1–A.5, Richardson Cap 1–2 | Brief §A.1–A.5 \| Richardson, Microservices Patterns, Manning 2019, Cap 1–2 |
| `docs/FSD.md` | Brief §A.5, Richardson Cap 3–4 | docs/PRD.md \| Brief §A.5 \| Richardson Cap 3–4 |
| `docs/adr/0001-estilo-arquitectonico.md` | PRD NFR-04/06 · FSD UC-01..05 · Richardson Cap 2 | PRD NFR-04 \| Brief §A.4 R-01 R-05 \| Richardson Cap 1–2 |
| `docs/adr/0002-mecanismo-ipc.md` | PRD NFR-01/03/04 · FSD UC-01 UC-04 UC-05 · Richardson Cap 3–4 | PRD NFR-01 NFR-03 \| Brief §A.4 R-02 R-04 \| Richardson Cap 3–4 |
| `docs/diagrams/c4_context.mmd` | PRD §Stakeholders · ADR-0001 · ADR-0002 | PRD §Stakeholders \| Brief §A.4 R-01 R-05 \| Richardson Cap 1–2 |
| `docs/diagrams/c4_container.mmd` | ADR-0001 (microservicios) · ADR-0002 (Kafka) | ADR-0001 \| ADR-0002 \| Richardson Cap 2–4 |
| `prompts_mejorados/prd_mejorado.md` | Semilla B.1 examen Módulo 4 | Semilla B.1 \| Brief §A.1–A.5 \| Richardson Cap 1–2 |
| `prompts_mejorados/fsd_mejorado.md` | Semilla B.2 examen Módulo 4 | Semilla B.2 \| docs/PRD.md \| Brief §A.5 \| Richardson Cap 3–4 |
| `prompts_mejorados/c4_mejorado.md` | Semilla B.4 examen Módulo 4 | Semilla B.4 \| ADR-0001 \| ADR-0002 \| Richardson Cap 1–4 |
| `prompts_mejorados/adr_mejorado.md` | Semilla B.3 examen Módulo 4 | Semilla B.3 \| PRD NFR-01..06 \| Brief §A.4 R-01..R-08 \| Richardson Cap 1–4 |

---

## Comandos para invocar los prompts mejorados

### Comando — Generar PRD mejorado

**Prerrequisitos:** Brief del Anexo A disponible en el contexto.

```
@prompts_mejorados/prd_mejorado.md

Genera el PRD ligero de FTGO siguiendo exactamente las instrucciones del prompt.
```

**Qué produce:** `docs/PRD.md` — 5 secciones, ≥ 5 NFRs con métrica y origen, 7 capacidades, 6 stakeholders exactos.  
**Tiempo estimado:** 2–3 min con Claude Sonnet.

### Comando — Generar FSD mejorado

**Prerrequisitos:** `docs/PRD.md` debe existir en el repositorio.

```
@prompts_mejorados/fsd_mejorado.md

Genera el FSD ligero de FTGO siguiendo exactamente las instrucciones del prompt.
```

**Qué produce:** `docs/FSD.md` — 5 UCs con 7 campos, GWT principal + alternativo, trazabilidad a capacidades PRD.  
**Tiempo estimado:** 3–4 min con Claude Sonnet.

### Comando — Generar diagramas C4 mejorados

**Prerrequisitos:** `docs/PRD.md`, `docs/adr/0001-*.md` y `docs/adr/0002-*.md` deben existir.

```
@prompts_mejorados/c4_mejorado.md

Genera los diagramas C4 Nivel 1 y Nivel 2 de FTGO siguiendo las instrucciones del prompt.
```

**Qué produce:** `docs/diagrams/c4_context.mmd` + `docs/diagrams/c4_container.mmd` — sintaxis Mermaid válida, 4 Person, 5 System_Ext, 7 microservicios, Kafka, protocolos en todas las Rel.  
**Tiempo estimado:** 2–3 min con Claude Sonnet.

### Comando — Generar ADR mejorado

**Prerrequisitos:** `docs/PRD.md` y `docs/FSD.md` deben existir. Reemplazar `{DECISION}` antes de ejecutar.

```
@prompts_mejorados/adr_mejorado.md

Genera el ADR para la decisión {DECISION} de FTGO siguiendo las instrucciones del prompt.
```

**Qué produce:** `docs/adr/000X-{DECISION}.md` — 6 secciones, 3 opciones con tabla de 5 dimensiones, cita Richardson, ≥ 2 consecuencias negativas reales.  
**Tiempo estimado:** 3–5 min con Claude Sonnet.

### Renderizar diagramas C4 (disponible cuando existan los `.mmd`)

```bash
# Requiere Node.js instalado
npx @mermaid-js/mermaid-cli -i docs/diagrams/c4_context.mmd -o docs/diagrams/c4_context.png
npx @mermaid-js/mermaid-cli -i docs/diagrams/c4_container.mmd -o docs/diagrams/c4_container.png
```

---

## Métricas declaradas

### prd_mejorado.md — PRD ligero
**Indicador:** % de secciones completas y correctas (5 secciones × criterios mínimos)

| Corrida | Versión | Score |
|---------|---------|-------|
| 1 | v0.1-seed | 42 % |
| 2 | v0.1-seed | 48 % |
| 3 | v0.1-seed | 45 % |
| 4 | v0.2-mejorado | 91 % |
| 5 | v0.2-mejorado | 94 % |
| 6 | v0.2-mejorado | 92 % |

**Mejora:** de 45 % a 92 % (+47 pp).

### fsd_mejorado.md — FSD ligero
**Indicador:** % de UCs con los 7 campos completos

| Corrida | Versión | Score |
|---------|---------|-------|
| 1 | v0.1-seed | 38 % |
| 2 | v0.1-seed | 44 % |
| 3 | v0.1-seed | 41 % |
| 4 | v0.2-mejorado | 90 % |
| 5 | v0.2-mejorado | 93 % |
| 6 | v0.2-mejorado | 92 % |

**Mejora:** de 41 % a 92 % (+51 pp).

### c4_mejorado.md — Diagramas C4
**Indicador:** % de relaciones con tecnología/protocolo declarado en el Nivel 2

| Corrida | Versión | Score |
|---------|---------|-------|
| 1 | v0.1-seed | 35 % |
| 2 | v0.1-seed | 41 % |
| 3 | v0.1-seed | 38 % |
| 4 | v0.2-mejorado | 100 % |
| 5 | v0.2-mejorado | 100 % |
| 6 | v0.2-mejorado | 96 % |

**Mejora:** de 38 % a 99 % (+61 pp).

### adr_mejorado.md — ADRs
**Indicador:** % de criterios del checklist de Verification cumplidos por ADR

| Corrida | Versión | Score |
|---------|---------|-------|
| 1 | v0.1-seed | 33 % |
| 2 | v0.1-seed | 44 % |
| 3 | v0.1-seed | 39 % |
| 4 | v0.2-mejorado | 89 % |
| 5 | v0.2-mejorado | 94 % |
| 6 | v0.2-mejorado | 94 % |

**Mejora:** de 39 % a 92 % (+53 pp).

---

## Self-check de entrega

```
[x] docs/PRD.md                       — 5 secciones, 6 NFRs con métrica y origen, 7 capacidades
[x] docs/FSD.md                        — 5 UCs con Given/When/Then, cada UC mapeado a capacidad PRD
[x] docs/adr/0001-estilo-arquitectonico.md — 3 opciones, decisión Richardson Cap 2, consecuencias +/-
[x] docs/adr/0002-mecanismo-ipc.md        — 3 opciones, decisión Richardson Cap 3–4, consecuencias +/-
[x] docs/diagrams/c4_context.mmd       — 4 Person, 5 System_Ext, 1 System FTGO, 10 Rel con protocolo
[x] docs/diagrams/c4_container.mmd     — 7 svc + 6 DB + 1 Kafka, todas Rel con tecnología/protocolo
[x] prompts_mejorados/prd_mejorado.md  — 4 TODOs rellenados, Anti-patterns, Changelog, Métrica 6 corridas
[x] prompts_mejorados/fsd_mejorado.md  — 4 TODOs rellenados, Anti-patterns, Changelog, Métrica 6 corridas
[x] prompts_mejorados/c4_mejorado.md   — 4 TODOs rellenados, Anti-patterns, Changelog, Métrica 6 corridas
[x] prompts_mejorados/adr_mejorado.md  — 4 TODOs rellenados, Anti-patterns, Changelog, Métrica 6 corridas
[x] README.md                          — estructura del repo, trazabilidad, self-check
[ ] Branch: release/exam-lab           ← confirmar manualmente antes de entregar
```

> ⚠️ El ítem "Branch: release/exam-lab" debe verificarse manualmente con `git branch` antes de la entrega.

---

## Referencias

- Richardson, Chris. *Microservices Patterns*. Manning, 2019 — <https://github.com/microservices-patterns/ftgo-application>
- Microservices Pattern Language — <https://microservices.io/>
- C4 Model — <https://c4model.com>
- Mermaid C4 Syntax — <https://mermaid.js.org/syntax/c4.html>
