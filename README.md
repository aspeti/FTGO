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
│   └── FSD.md                    ← Artefacto 2: FSD con 5 UCs Given/When/Then
└── prompts_mejorados/
    ├── prd_mejorado.md           ← Artefacto 7a: Prompt PRD mejorado (v0.2)
    └── fsd_mejorado.md           ← Artefacto 7b: Prompt FSD mejorado (v0.2)
```

> Artefactos pendientes antes de la entrega:
> `docs/adr/0001-*.md` · `docs/adr/0002-*.md` ·
> `docs/diagrams/c4_context.mmd` · `docs/diagrams/c4_container.mmd` ·
> `prompts_mejorados/adr_mejorado.md` · `prompts_mejorados/c4_mejorado.md`

---

## Trazabilidad de artefactos

| Artefacto | Fuente principal | Trazabilidad |
|-----------|-----------------|--------------|
| `docs/PRD.md` | Brief §A.1–A.5, Richardson Cap 1–2 | Brief §A.1–A.5 \| Richardson, Microservices Patterns, Manning 2019, Cap 1–2 |
| `docs/FSD.md` | Brief §A.5, Richardson Cap 3–4 | docs/PRD.md \| Brief §A.5 \| Richardson Cap 3–4 |
| `docs/adr/0001-*.md` | — | _archivo pendiente_ |
| `docs/adr/0002-*.md` | — | _archivo pendiente_ |
| `docs/diagrams/c4_context.mmd` | — | _archivo pendiente_ |
| `docs/diagrams/c4_container.mmd` | — | _archivo pendiente_ |
| `prompts_mejorados/prd_mejorado.md` | Semilla B.1 examen Módulo 4 | Semilla B.1 \| Brief §A.1–A.5 \| Richardson Cap 1–2 |
| `prompts_mejorados/fsd_mejorado.md` | Semilla B.2 examen Módulo 4 | Semilla B.2 \| docs/PRD.md \| Brief §A.5 \| Richardson Cap 3–4 |
| `prompts_mejorados/adr_mejorado.md` | — | _archivo pendiente_ |

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

---

## Self-check de entrega

```
[x] docs/PRD.md                       — 5 secciones, 6 NFRs con métrica y origen, 7 capacidades
[x] docs/FSD.md                        — 5 UCs con Given/When/Then, cada UC mapeado a capacidad PRD
[ ] docs/adr/0001-*.md                 — ≥3 opciones, decisión con Richardson, consecuencias +/-
[ ] docs/adr/0002-*.md                 — ≥3 opciones, decisión con Richardson, consecuencias +/-
[ ] docs/diagrams/c4_context.mmd       — ≥1 Person, ≥2 System_Ext, 1 System FTGO
[ ] docs/diagrams/c4_container.mmd     — ≥5 contenedores, relaciones con tecnología/protocolo
[x] prompts_mejorados/prd_mejorado.md  — 4 TODOs rellenados, Anti-patterns, Changelog, Métrica 6 corridas
[x] prompts_mejorados/fsd_mejorado.md  — 4 TODOs rellenados, Anti-patterns, Changelog, Métrica 6 corridas
[ ] prompts_mejorados/adr_mejorado.md  — ≥2 TODOs rellenados, 1 sección nueva, Changelog, Métrica 3 corridas
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
