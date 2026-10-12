# Reporte Inventario pro — Proyecto Power BI (PBIP) con setup agéntico

Proyecto Power BI en formato **PBIP** (reporte PBIR + modelo semántico TMDL) preparado para ser operado por un agente de IA (Claude Code) con un alcance acotado: **visuales, medidas DAX y consultas de lectura**.

Este README documenta sobre todo la **configuración técnica del setup agéntico**: qué archivos lo componen, cómo se conectan y cómo se debe usar.

---

## 1. Estructura del repositorio

```text
Clase/
├── CLAUDE.md                          # Orquestador: alcance, enrutamiento, reglas (instrucciones del agente)
├── .claude/
│   ├── SKILL.md                       # Skill `powerbi-report-cli` (despachador de modos)
│   ├── apm.yml                        # Manifiesto APM: dependencias de la skill + servidor MCP
│   └── references/                    # Guías detalladas (authoring, design, planning, management)
│       ├── authoring*.md              # Guías del modo authoring (+ authoring/*.md por tema)
│       ├── design*.md                 # Guías del modo design (+ design/*.md, archetypes/)
│       ├── planning*.md               # (no se usan en este proyecto)
│       └── management*.md             # (no se usan en este proyecto)
├── Reporte Inventario pro.pbip        # Punto de entrada del proyecto (abrir en Power BI Desktop)
├── Reporte Inventario pro.Report/     # Reporte en formato PBIR
│   ├── definition.pbir                # Enlace al modelo (byPath → ../Reporte Inventario pro.SemanticModel)
│   ├── definition/                    # report.json, version.json, pages/…
│   └── StaticResources/
├── Reporte Inventario pro.SemanticModel/   # Modelo semántico en TMDL
│   ├── definition.pbism
│   ├── definition/                    # model.tmdl, relationships.tmdl, tables/*.tmdl, cultures/
│   └── TMDLScripts/
└── .gitignore                         # Ignora .pbi/localSettings.json y .pbi/cache.abf
```

> **Nota:** `CLAUDE.md` se refiere a la carpeta de la skill como `,claude/` (con coma), pero en disco se llama **`.claude/`** (con punto). Las rutas de `CLAUDE.md` deberían corregirse a `.claude/` para que el agente encuentre `SKILL.md` y `references/`.

---

## 2. Componentes del setup agéntico

### 2.1 `CLAUDE.md` — orquestador

Se carga automáticamente en cada sesión de Claude Code y define:

| Aspecto | Definición |
|---|---|
| **Rol** | Asistente de un único proyecto Power BI PBIP; responde siempre en español |
| **Alcance** | Solo 3 áreas: **A.** Visuales y gráficos · **B.** Medidas DAX · **C.** Consultas (solo lectura) |
| **Fuera de alcance** | Publicar/descargar de Fabric, rebind, ETL/Power Query, columnas calculadas, cambios de esquema/relaciones, RLS, análisis de negocio, temas ajenos a Power BI. Se rechaza con una plantilla fija |
| **Clasificación** | Por *intención* (no por archivo abierto), con reglas evaluadas en orden: fuera de alcance → consulta → DAX → visual |
| **Peticiones compuestas** | Orden obligatorio **C → B → A**, con validación entre fases y anuncio de cada paso. Si una fase falla, se detiene |
| **Enrutamiento** | Tabla de necesidad → guía concreta en `.claude/references/` |
| **Reglas transversales** | Cambios mínimos, sin datos sintéticos, confirmar antes de borrar, cambios solo locales (nunca publicar) |

### 2.2 `.claude/SKILL.md` — skill `powerbi-report-cli` (v1.0.5)

Despachador de **cuatro modos**; cada uno exige leer su guía completa antes del primer comando:

| Modo | Uso | ¿Se usa aquí? |
|---|---|---|
| `authoring` | Editar/validar/previsualizar PBIR local | ✅ Sí (área A y C) |
| `design` | Criterio estético: tipo de gráfico, layout, color, tipografía, accesibilidad | ✅ Sí, de apoyo (`brownfield.md` para reportes existentes) |
| `planning` | Reportes nuevos desde cero con spec y aprobación | ❌ Fuera de alcance |
| `management` | Publicar/descargar/rebind en Fabric | ❌ Fuera de alcance |

Reglas relevantes de la skill: usar el CLI para metadatos PBIR en vez de adivinar, mantener los cambios locales salvo petición explícita, y exigir el header de telemetría `x-ms-fabric-skill: powerbi-report-cli` en cualquier llamada a `api.fabric.microsoft.com` (no aplica normalmente, porque Fabric está fuera de alcance).

### 2.3 `.claude/apm.yml` — manifiesto APM

Generado por `build/gen_apm_manifests.py` (no editar a mano). Equivale a:

```bash
apm install microsoft/skills-for-fabric --skill powerbi-report-cli
```

| Campo | Valor |
|---|---|
| `name` | `powerbi-report-cli` |
| `version` | `0.2.9` |
| `author` / `license` | Microsoft Fabric / MIT |
| Dependencia MCP | `powerbi-modeling-mcp` |
| Transporte | `stdio` |
| Comando | `npx -y @microsoft/powerbi-modeling-mcp@latest --start` |
| Tools | `*` (todas) |

### 2.4 Herramientas que usa el agente

| Herramienta | Para qué | Fuente de verdad de… |
|---|---|---|
| **CLI `powerbi-report-author`** | Catálogo de visuales, roles, propiedades de formato, enums, selectores, `validate`, `preview-*` | Esquemas y propiedades PBIR (nunca de memoria) |
| **MCP `powerbi-modeling`** | Leer/escribir el modelo: `measure_operations`, `dax_query_operations`, `table_operations`, etc. | Tablas, columnas, medidas, relaciones |
| **Skill `semantic-model-authoring`** | Guía para trabajo de modelo y DAX (no está en `.claude/`) | Buenas prácticas de modelo/DAX |

---

## 3. Flujo de trabajo del agente

```text
Petición del usuario
        │
        ▼
¿Fuera de alcance? ──sí──▶ Rechazo amable (plantilla) + alternativa
        │ no
        ▼
Clasificar intención ──▶ C (consulta) · B (DAX) · A (visual)
        │
        ▼
Petición compuesta:  C ──▶ B ──▶ A   (validando entre fases)
```

Ejemplo: *"Crea el margen % y muéstralo en un gráfico de barras por categoría"*
1. **C** — leer modelo/reporte y confirmar nombres reales.
2. **B** — crear la medida `Margen %` y probarla con una consulta DAX.
3. **A** — crear el gráfico de barras enlazado a la medida.

### Compuerta de finalización para visuales

1. Obtener roles y propiedades con el CLI (`catalog describe`, `formatting describe-object`).
2. `powerbi-report-author validate "Reporte Inventario pro.Report"` tras cada lote de cambios.
3. Cargar en Desktop (`preview ... --host desktop --reload`), capturar **cada página afectada** (`--all-pages` si el cambio es global) y revisar con `authoring/screenshot-review.md`.
4. Corregir y repetir. Si no se puede capturar, la validación visual se reporta como **bloqueada** (no se declara éxito).

### Flujo para medidas DAX

1. Cargar `semantic-model-authoring`.
2. Conectar el MCP al modelo (Desktop o carpeta TMDL).
3. Leer tablas/columnas/medidas existentes.
4. Escribir la medida, evaluarla con una consulta DAX de prueba y verificar.
5. Si cambió el TMDL en disco y hay visuales afectados: `preview ... --reload-with-model`.

Buenas prácticas: medidas (no columnas calculadas), `VAR`/`RETURN`, `DIVIDE` en lugar de `/`, evitar iteradores innecesarios, `formatString` y carpeta de visualización.

### Consultas (solo lectura)

`preview-pages`, `preview-visuals`, `preview-filters`, `preview-themes`, `validate` y `preview ... --host desktop --status`. Una consulta **nunca** modifica archivos, ejecuta `--reload` ni escribe en el modelo.

---

## 4. Requisitos y puesta en marcha

| Requisito | Detalle |
|---|---|
| Power BI Desktop | Con soporte de proyectos PBIP / PBIR habilitado |
| Claude Code | CLI o extensión de VS Code, abierto en la raíz del repo |
| Node.js | Para ejecutar el MCP vía `npx` |
| CLI `powerbi-report-author` | Debe estar en el `PATH`. Verificar con `powerbi-report-author doctor` |
| (Opcional) APM | Para reinstalar la skill: `apm install microsoft/skills-for-fabric --skill powerbi-report-cli` |

Pasos:

1. Abrir `Reporte Inventario pro.pbip` en Power BI Desktop (para previsualización y capturas).
2. Abrir esta carpeta con Claude Code; `CLAUDE.md` se carga solo.
3. Aprobar/conectar el servidor MCP `powerbi-modeling` cuando Claude Code lo solicite.
4. Ejecutar `powerbi-report-author doctor` si el CLI no responde.

---

## 5. Contenido actual del proyecto

### Modelo semántico (TMDL)

| Tipo | Tablas |
|---|---|
| Dimensiones | `Dim Clientes`, `Dim Date`, `Dim SKU`, `Dim Tiendas` |
| Hechos | `Fact Ventas`, `Fact Inventario` |
| Medidas | `Medidas` |

**Relaciones** (hechos → dimensiones):

| Desde | Hacia |
|---|---|
| `Fact Ventas[store_id]` | `Dim Tiendas[store_id]` |
| `Fact Inventario[store_id]` | `Dim Tiendas[store_id]` |
| `Fact Ventas[sku_id]` | `Dim SKU[sku_id]` |
| `Fact Inventario[sku_id]` | `Dim SKU[sku_id]` |
| `Fact Ventas[customer_id]` | `Dim Clientes[cust_id]` |
| `Fact Ventas[date]` | `Dim Date[Date]` |

**Medidas definidas:** `Ventas`, `Cantidad Productos`, `Tickets`, `Costo de Ventas`, `Margen Bruto`, `Margen %`, `Stock Disponible`, `Ventas(M)`.

### Reporte (PBIR)

Tres páginas de 1280×720 (orden en `pages.json`): `Página 1`, `Página 2` y **`Resumen de ventas`** (activa, carpeta `resumen`).

Visuales de `resumen`: `banner-encabezado`, `txt-titulo-pagina`, `kpi-ventas`, `kpi-margen-pct`, `kpi-tickets`, `kpi-stock-disponible`, `chart-ventas-categoria`, `chart-ventas-evolucion`.

El enlace reporte → modelo está en `definition.pbir` (`byPath` relativo: `../Reporte Inventario pro.SemanticModel`).

---

## 6. Convenciones y límites

- **Idioma:** el agente responde siempre en español.
- **Nunca inventar** tablas, columnas, medidas, páginas ni datos sintéticos.
- **Cambios mínimos:** conservar estructura, esquemas y convenciones existentes.
- **Solo local:** nada se publica a Fabric.
- **Confirmación previa** para borrar visuales, páginas o medidas.
- **No editar JSON de PBIR a ciegas:** usar el CLI y validar.
- **Informe final** conciso: qué se hizo, qué se validó (esquema y captura) y problemas encontrados.
- **Git:** `.pbi/localSettings.json` y `.pbi/cache.abf` están ignorados.

---

## 7. Ejemplos de uso

| Petición | Clasificación | Resultado |
|---|---|---|
| "¿Cuántas páginas y visuales tiene el reporte?" | C | Resumen en tabla, sin modificar nada |
| "Corrige la medida `Margen %` para evitar divisiones por cero" | B | Edita vía MCP, prueba con DAX |
| "Cambia el color de las tarjetas KPI y alinéalas" | A | CLI + `validate` + captura |
| "Crea una medida de ticket promedio y muéstrala en una tarjeta" | C → B → A | Flujo compuesto con validación por fase |
| "Publica el reporte en mi workspace" | Fuera de alcance | Rechazo con alternativa |
| "Agrega una columna calculada" | Fuera de alcance | Rechazo (solo medidas) |
