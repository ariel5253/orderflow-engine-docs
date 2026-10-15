# orderflow-engine-docs

[English](README.md)

Documentación de **Orderflow Engine**: el motor de precios y la confirmación idempotente de pedidos de un marketplace B2B de bebidas que opera en varios países. El enunciado original está en [`assignment/`](assignment/). El código entregable, su README y el ADR viven en el repositorio `orderflow-engine-api`. Los demás repositorios se citan por nombre, sin enlace, para que cada uno funcione clonado solo.

> **¿Abres una sesión nueva? Empieza por [`handoff/HANDOFF.md`](handoff/HANDOFF.md)** (en inglés). Tiene el estado verificado, los comandos para retomar, las decisiones abiertas y las trampas ya encontradas.

```
orderflow-engine-docs/
├─ README.md · README.es.md   este archivo: el mapa y el glosario de todo el espacio de trabajo
├─ LICENSE                    MIT
├─ handoff/                   el único lugar donde vive el estado verificado, con la gráfica de avance
├─ spec/                      la especificación viva (inglés)
├─ assignment/                el enunciado original, intocable
├─ diagrams/                  figuras SVG generadas, numeradas por orden de aparición
├─ governance/                regla de oro, ramas, pull requests, commits, reglas de documentación
└─ deliverables/             ADR en Word y PDF (generados) y manifest.json con el origen de cada uno
```

**La regla de orden de esta carpeta, en una línea: en la raíz solo están los README y la licencia.** Todo lo demás vive en una carpeta, y el glosario de abajo dice quién edita cada una.

## Glosario de carpetas

| Carpeta o archivo | Qué es | ¿Se edita a mano? |
|-------------------|--------|-------------------|
| [`handoff/HANDOFF.md`](handoff/HANDOFF.md) | **Empieza aquí.** Estado, comandos, decisiones abiertas, trampas | Sí, y **tiene dueño** |
| [`spec/constitution.md`](spec/constitution.md) | Los principios innegociables, cada uno con cómo se verifica | Sí |
| [`spec/SPEC.md`](spec/SPEC.md) | Qué y por qué: alcance, reglas, política de dinero, consistencia, criterios de aceptación | Sí |
| [`spec/plan.md`](spec/plan.md) | Decisiones técnicas con sus alternativas descartadas, y registro de riesgos | Sí |
| [`spec/TASKS.md`](spec/TASKS.md) | Tareas en orden de dependencia, cada una con su verificación | Sí |
| [`spec/api-contract.md`](spec/api-contract.md) | El contrato: petición, respuesta y errores de cada operación | Sí |
| [`spec/data-model.md`](spec/data-model.md) | Tablas, restricciones, índices y bloqueos | Sí |
| [`spec/architecture.md`](spec/architecture.md) | Topología, hexágono, flujos y riesgos, con las figuras | Sí |
| [`assignment/`](assignment/) | El enunciado original | **No.** Es el original |
| [`diagrams/`](diagrams/) | Las figuras SVG | **No.** Las genera y verifica `orderflow-engine-tools` |
| [`governance/`](governance/) | Regla de oro, ramas, pull requests, commits, reglas de documentación, plantillas, configuración del check | Sí |
| [`deliverables/`](deliverables/) | El ADR en Word y PDF, en inglés y español, y `manifest.json` con la fuente y el hash de cada uno | **No.** Los genera `orderflow-engine-tools` |

## Mapa de documentos

Un documento que no aparece aquí no existe (ver las [reglas de documentación](governance/documentation-rules.md), en inglés).

| ID | Documento | Pregunta que responde | Idioma | Estado |
|----|-----------|-----------------------|--------|--------|
| README-API | `orderflow-engine-api/README.es.md` | ¿Cómo se ejecuta y cuáles son el orden de palancas, la política de redondeo y el modelo de consistencia? | en, es | aceptado |
| ADR-0001 | `orderflow-engine-api/docs/adr/0001-order-pricing-and-confirmation.es.md` | ¿Qué se decidió, qué se dejó fuera, qué revisar y qué riesgo queda? | en, es | propuesto |
| HANDOFF | [Traspaso](handoff/HANDOFF.md) | ¿En qué punto está el trabajo y cómo lo retomo? | en | aceptado |
| CONSTITUTION | [Constitución](spec/constitution.md) | ¿Qué principios son innegociables y cómo se verifica cada uno? | en | aceptado |
| SPEC | [Especificación](spec/SPEC.md) | ¿Qué debe hacer el sistema y cómo se verifica cada regla? | en | aceptado |
| PLAN | [Plan técnico](spec/plan.md) | ¿Qué decisiones técnicas se tomaron y con qué riesgos? | en | aceptado |
| TASKS | [Plan de tareas](spec/TASKS.md) | ¿En qué orden se construye y cómo se verifica cada paso? | en | aceptado |
| API-CONTRACT | [Contrato de la API](spec/api-contract.md) | ¿Qué acepta, devuelve y rechaza cada operación? | en | aceptado |
| DATA-MODEL | [Modelo de datos](spec/data-model.md) | ¿Qué tablas, restricciones y bloqueos sostienen las invariantes? | en | aceptado |
| ARCHITECTURE | [Arquitectura](spec/architecture.md) | ¿Qué forma tiene Orderflow Engine y dónde están sus riesgos? | en | aceptado |
| GOV-GOLDEN-RULE | [Regla de oro](governance/golden-rule.md) | ¿A qué regla única sirven todas las demás? | en | aceptado |
| GOV-BRANCHES | [Ramas y ambientes](governance/branches-and-environments.md) | ¿Qué ramas existen y cómo se protegen? | en | aceptado |
| GOV-PULL-REQUESTS | [Reglas de pull request](governance/pull-request-rules.md) | ¿Cómo se abre, verifica, integra y limpia un pull request? | en | aceptado |
| GOV-COMMITS | [Convenciones de commits](governance/commit-conventions.md) | ¿Cómo se escribe un mensaje de commit? | en | aceptado |
| GOV-INTEGRITY | [Integridad de repositorios](governance/repository-integrity.md) | ¿Qué debe contener, ignorar y nunca contener cada repositorio? | en | aceptado |
| GOV-001 | [Reglas de documentación](governance/documentation-rules.md) | ¿Cómo se escriben, ubican y verifican los documentos? | en | aceptado |
| GOV-002 | [Configuración del check](governance/docs-governance.json) | ¿Qué rutas, pares, encabezados y límites aplica la verificación? | en | aceptado |
| TPL-ADR | [Plantilla de ADR](governance/templates/adr-template.md) | ¿Cómo se registra una decisión de arquitectura? | en | plantilla |
| TPL-DOC | [Plantilla de documento](governance/templates/doc-template.md) | ¿Cómo empieza un documento nuevo? | en | plantilla |
| README-WEB | `orderflow-engine-web/README.es.md` | ¿Qué es la pantalla y cómo se ejecuta? | en, es | planeado |
| README-INFRA | `orderflow-engine-infra/README.es.md` | ¿Cómo se levanta el stack completo y cómo se ejecutan todas las pruebas? | en, es | planeado |
| README-TOOLS | `orderflow-engine-tools/README.es.md` | ¿Qué genera y verifica cada herramienta, y cómo se ejecuta? | en, es | aceptado |
| README-APP | `orderflow-engine-app/README.es.md` | ¿Por qué esta carpeta está reservada? | en, es | reservado |

## Códigos que verás citados

Los documentos se citan entre sí por código, no por página. Cada familia de códigos tiene un prefijo distinto.

| Código | Identifica | Dónde vive |
|--------|-----------|------------|
| `G-1`…`G-4` | Garantías del sistema | SPEC §0 |
| `P-01`…`P-15` | Principios de la constitución | constitution |
| `REQ-…` | Requisitos del enunciado | SPEC §14.1 |
| `BR-00`…`BR-07` | Reglas de negocio | SPEC §3 |
| `RP-1`…`RP-5` | Puntos de redondeo | SPEC §4 |
| `I-201` | La invariante del 201 | SPEC §7.2 |
| `GT-01`…`GT-06` | Los seis golden tests del enunciado | SPEC §11 |
| `A-01`…`A-10` | Supuestos | SPEC §13 |
| `AC-01`…`AC-48` | Criterios de aceptación. Las filas eliminadas conservan su hueco | SPEC §14.2 |
| `DEC-…`, `RISK-…` | Decisiones técnicas, riesgos | plan |
| `T-00`…`T-17`, `GC-…`, `RF-…` | Tareas, restricciones globales, foco de revisión | TASKS |
| `E-01`…`E-06`, `BD-…`, `GAP-…` | Fichas de operación, decisiones de comportamiento, huecos conocidos | api-contract |
| `DM-…` | Notas del modelo de datos | data-model |
| `ADR-0001` | El registro de decisión de arquitectura | `orderflow-engine-api/docs/adr/` |
| `DR-1`…`DR-6` | Reglas de documentación | governance |
| `OD-…`, `TR-…` | Decisiones abiertas, trampas | handoff |

## Verificar la documentación

```bash
node orderflow-engine-tools/docs-check/check-docs.mjs
```

Se ejecuta desde la raíz del espacio de trabajo. Las herramientas viven en el repositorio `orderflow-engine-tools`; esta carpeta solo guarda sus resultados. Falla ante cualquiera de estos casos:
- archivos Markdown sin indexar o fuera de lugar
- pares de idioma faltantes
- enlaces relativos rotos, incluidas las figuras
- front matter inválido
- falta de un encabezado obligatorio, como los cuatro temas del README que exige el enunciado
- un ADR de más de una página
- claves i18n que no coinciden
