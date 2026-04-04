# Contexto: Portfolio Dashboard — Balanz / Julian
## Versión: 04/04/2026 v3

---

## Qué es este proyecto

Dashboard HTML single-file para visualizar el portfolio de inversiones en Balanz.
Hosteado en GitHub Pages — URL pública: **https://julianfromarg.github.io/portfolio-tracker**

Repositorio del dashboard: **github.com/julianfromarg/portfolio-tracker** (público)
Repositorio de precios: **github.com/julianfromarg/portfolio-tracker-prices** (privado)

---

## Archivos

| Archivo | Descripción |
|---|---|
| `index.html` | El dashboard completo (en repo portfolio-tracker). ~4000 líneas. |
| `MovimientosHistoricos_Completo.xls` | Export de Balanz: Reportes → Movimientos Históricos |
| `contexto.md` | Este archivo |

---

## Estructura del dashboard

### 8 tabs:
- **📊 Portfolio** — Tab unificado con toggle 🇦🇷 Argentina / 🇺🇸 EEUU (USD). Tabla de posiciones abiertas con precios live + caja + totales. Argentina tiene toggle de modo (moneda origen / todo ARS / todo USD). FX siempre desde `fx_rates` en Supabase — sin input manual.
- **📋 Transacciones** — Tabla audit completa con filtros (cuenta, operación, año, mes), agrupación, columna Saldo cash, columna Ret. NRA y botón `↓ CSV`.
- **💵 Saldo Cash** — Tabla de saldos diarios por cuenta.
- **🏛 Ret. NRA** — Tabla de retenciones NRA estimadas para dividendos EEUU, con filtros, override por transacción y botón `↓ CSV`.
- **📈 Evolución** — Gráfico de línea con 3 series independientes: 🇦🇷 Argentina (azul `#3d7fff`), 🇺🇸 EEUU (rojo `#ef4444`), ∑ Total (violeta `#a78bfa`). Sin fill. Toggle individual por serie. Zoom, pan, selector de rango 1M/3M/6M/1Y/MAX.
- **🔍 Especies** — Tab de auditoría por instrumento. Ver más abajo.
- **💱 Tipo de Cambio** — Tabla con la serie histórica de FX USD/ARS desde Supabase. Columnas: Fecha · ARS/USD · Var. diaria · Var. 30d. Filtros por año y texto. Orden clickeable por fecha o tasa.
- **Botones fijos:** `↑ Importar .xls` (en tabs bar)
- **Header:** selector de sesiones con `＋ Nueva`, `✎ Renombrar`, `🗑 Borrar`

### Tab activo persistente:
`switchTab()` guarda el tab activo en `localStorage('active_tab')`. El `init()` lo restaura al cargar. `switchTab` es defensivo — hace guard si el panel no existe.

### Flujo de datos:
1. Abrir → lee sesión activa de `localStorage` → si vacío, muestra estado vacío
2. Al cargar datos → `fetchLatestPrices()` automático desde Supabase + `fetchPricesHistory()` + `fetchFXHistory()` + `fetchSnapshots()`
3. Importar .xls → `handleFileImport()` detecta HTML vs XLSX → parsea → `mergeTransactions()` → `processAndRefresh()` → guarda en localStorage (key de sesión) → `fetchLatestPrices()`
4. Botón 🔄 Actualizar precios → re-fetchea Supabase manualmente
5. Botón ⟳ Recalcular (tab Evolución) → reconstruye snapshots históricos → upsert en `portfolio_snapshots`
6. Borrar sesión → borra localStorage de esa sesión + overrides NRA en Supabase para esa sesión

---

## Sistema de sesiones

### Concepto
Cada "sesión" representa una cuenta de Balanz independiente. Las sesiones son locales al browser — no hay autenticación todavía (Fase 3 pendiente).

### Estructura en localStorage
```
sessions_index        → [{id, name, createdAt}]
active_session_id     → id de la sesión activa
active_tab            → tab activo (portfolio/audit/cash/nra/evol/especies/fx)
portfolio_txns_{id}   → transacciones de esa sesión
```

### session_id especial: `'default'`
- La primera sesión (migrada desde datos existentes) usa `id = 'default'`
- Los NRA overrides en Supabase que tenían `session_id = 'default'` siguen funcionando
- Las sesiones nuevas generan IDs del tipo `sess_xxxxx_xxxxx`

### Funciones clave
- `getSessions()` / `saveSessions()` — CRUD del índice
- `getActiveSessionId()` / `setActiveSessionId()` — sesión activa
- `getSessionTxnsKey(id)` → `portfolio_txns_{id}`
- `switchSession(id)` — carga transacciones y overrides de otra sesión
- `newSessionPrompt()` — crea sesión nueva con nombre
- `renameSessionPrompt()` — renombra sesión activa
- `clearAllData()` — borra sesión activa (localStorage + Supabase NRA overrides)
- `renderSessionDropdown()` — refresca el `<select>` del header

### Migración automática
Al cargar la nueva versión por primera vez, si hay `portfolio_txns` en localStorage sin sesión, se migra automáticamente a `portfolio_txns_default` con sesión "Mi cuenta".

---

## Tab Portfolio — diseño

### Columnas de la tabla:
| Columna | Descripción |
|---|---|
| Instrumento | Ticker canonicalizado |
| Cantidad | Posición neta (solo abiertas, net > 0) |
| P. Prom. s/comis. | Precio promedio sin comisiones ni impuestos, en la moneda del modo activo |
| P. Prom. c/comis. | Precio promedio con comisiones e impuestos, en la moneda del modo activo |
| Precio Actual | Precio de mercado EOD desde Supabase, o precio promedio s/comis si no hay precio (marcado con `*` en naranja) |
| Gan./Pérd. | (Precio actual − P. Prom. s/comis.) × cantidad + % (solo cuando hay precio de mercado) |
| Valor Tenencia | Precio actual × cantidad |

### Lógica de precio con fallback:
- `getPriceForDate(ticker, date, avgCost)` — busca precio en `_pricesHistory[ticker][date]`, luego el más reciente anterior a esa fecha, y si no hay nada usa `avgCost` (P. Prom. s/comis.)
- Precios estimados al costo aparecen con `*` en color naranja (`var(--usd)`)
- P&L solo se calcula cuando hay precio de mercado real (no estimado)

### Caja:
- Panel Argentina: dos filas — Caja ARS y Caja USD
- Panel EEUU: una fila — Caja USD
- Aparece al pie de las posiciones, antes del total

### Total Argentina — toggle 3 modos:
- **Moneda origen:** ARS en ARS + USD en USD (dos líneas separadas)
- **Todo ARS:** todo convertido a pesos al FX de `_fxLatest`
- **Todo USD:** usa `AR_BLENDED_USD.avg` (FX histórico por compra) cuando está disponible; fallback a conversión con FX actual para instrumentos de moneda única
- FX siempre tomado de `_fxHistory` / `_fxLatest` desde Supabase — no hay input manual
- Posiciones AR valuadas al costo (P. Prom. s/comis.) hasta tener precios BYMA

### Total EEUU:
- Suma posiciones con fallback al costo + caja USD
- Muestra `*` en el footer cuando alguna posición usa precio estimado

### Columnas P. Prom. en modo Argentina:
Las columnas de precio promedio varían según el modo activo (ARS/USD), usando `avgByAcct` por slot:
- **Moneda origen:** muestra cada slot en su moneda nativa (`$ X` / `U$S Y`) separados por `/` si hay ambos
- **Todo ARS:** convierte slot USD multiplicando por FX, promedio ponderado por balance
- **Todo USD:** usa `AR_BLENDED_USD` cuando hay mezcla de monedas (histórico); fallback a FX actual para moneda única

---

## Retención NRA — `calcNRA(t)`

**Contexto:** Balanz no reporta la retención NRA del 30% para dividendos de no residentes en cuenta EEUU. El dashboard la estima y la descuenta del cash.

**Fórmula default:**
```javascript
// Solo aplica a: operacion === 'Pago de Dividendos' AND cuenta === 'EEUU (USD)'
const gross = Math.abs(monto) + Math.abs(comision) + Math.abs(iva);
const nra = -(gross * 0.30); // negativo = egreso de caja
```

**Sistema de overrides — monto ad-hoc por transacción:**

La retención real puede diferir del 30% calculado automáticamente. El usuario puede ingresar la retención real directamente.

```javascript
function calcNRA(t) {
  if(!isNRAApplicable(t)) return 0;
  const key = String(t.nro_mov);
  if(_nraOverrides.has(key)) {
    return _nraOverrideAmounts.get(key) || 0;
  }
  const gross = Math.abs(t.monto||0) + Math.abs(t.comision||0) + Math.abs(t.iva||0);
  return -(gross * NRA_RATE);
}
```

**Tabla `nra_overrides` en Supabase:**
```
nro_mov        TEXT
excluir        BOOL          (legacy)
monto_override NUMERIC       (NULL = auto 30%; valor = usar ese monto)
session_id     TEXT
PRIMARY KEY (nro_mov, session_id)
```

**CRÍTICO — race condition al inicio resuelta:**
`fetchLatestPrices()` recalcula `buildCash()` y `buildCashDaily()` después de cargar los overrides de Supabase.

---

## Estructura del .xls de Balanz

**CRÍTICO: el archivo .xls de Balanz es en realidad HTML disfrazado** — parsear con `DOMParser`, nunca con SheetJS.

```
[0]  Nro. de Mov.   → nro_mov
[1]  Nro. de Boleto → nro_boleto
[2]  Tipo Mov.      → "Compra(STRC)", "Venta(AL30D)", etc.
[3]  Concert.       → fecha "dd/mm/yy"
[5]  Est            → "Terminada" | "Cancelada"
[6]  Cant. titulos  → cantidad
[7]  Precio         → en moneda de la cuenta, coma decimal AR
[8]  Comis.
[9]  Iva Com.
[10] Otros Imp.
[11] Monto          → neto con signo (negativo=egreso)
[13] Tipo Cuenta    → "Inversion Estados Unidos Dolares" / "Inversion Argentina Pesos" / "Inversion Argentina Dolares"
```

**CRÍTICO — doble fila por operación:**
Cuando un instrumento opera fuera de su cuenta nativa, el broker registra DOS filas con el mismo `nro_mov`:
- **Fila 1** en la cuenta real del instrumento: títulos + monto
- **Fila 2** en Argentina (ARS): solo comisiones en pesos — NO mueve títulos

Ejemplos:
- Venta de AY24C (EEUU): fila EEUU (USD) = real + fila Argentina (ARS) = comisión
- Compra de AY24D: fila Argentina (USD) = real + fila Argentina (ARS) = comisión

---

## Instrument Ledger — `buildLedger()` ★ NUEVA ARQUITECTURA

### Concepto
`buildLedger(rawTxns, fxHistory)` es la **fuente de verdad para balances de títulos**. Opera sobre `rawTxns` crudos (pre-`deduplicateMEP`, pre-`canonicalizeTickers`). Convive con `buildInstrument`/`_instruments` durante la transición.

### Global
```javascript
let _ledger = {};  // { ticker_canonicalizado: LedgerInstrument }
```

### Estructura de `_ledger[ticker]`
```javascript
{
  ticker: 'AY24',
  txns: [...],        // todas las txns ordenadas cronológicamente, con _snap por fila
  slots: {
    AR_ARS: { bal, costARS, costUSD, costWithC },
    AR_USD: { bal, costUSD, costWithC },
    EEUU:   { bal, costUSD, costWithC },
  },
  totals: { balAR, balEEUU, balTotal, avgAR_usd, avgAR_ars, avgEEUU_usd },
  meta: { isClosed, isStale, isBond, firstDate, lastDate, opCount },
}
```

### Cada txn tiene `_snap` con el estado completo después de esa operación:
```javascript
{
  AR_ARS: { bal, avgNoC },
  AR_USD: { bal, avgNoC },
  EEUU:   { bal, avgNoC },
  balAR, balEEUU, balTotal,
  avgAR_usd, avgAR_ars, avgEEUU_usd,
  fx,
}
```

### Regla `movesTitle(ticker_raw, cuenta)` — qué fila mueve títulos
```javascript
// Sufijo C (ej: AY24C) → solo mueve títulos en cuenta EEUU (USD)
// Sufijo D (ej: AY24D) → solo mueve títulos en cuenta Argentina (USD)
// Sin sufijo            → siempre mueve títulos
```
Las filas de comisión (misma operación, cuenta Argentina ARS) devuelven `_movesTitle: false` y no tocan balances.

### Regla de slot
```
cuenta EEUU (USD)      → slot 'EEUU'
cuenta Argentina (ARS) → slot 'AR_ARS'
cuenta Argentina (USD) → slot 'AR_USD'
```

### Lógica de ventas — CRÍTICO
**Venta desde AR (AY24 o AY24D):**
1. Consumir del slot propio primero
2. Si no alcanza (ej: MEP donde se vende AY24 pero los títulos están en AR_USD), consumir el remanente del otro slot AR

**Venta desde EEUU (AY24C):**
1. Transferencia implícita AR → EEUU proporcional (preserva precio promedio): salen de AR_ARS y AR_USD en proporción a sus balances respectivos, entran en EEUU al avg USD de AR
2. Venta normal desde EEUU

**Por qué no usar pool proporcional para ventas AR:** en un MEP, AY24 (AR_ARS) y AY24D (AR_USD) se compran/venden en el mismo día — si se usa pool proporcional, la venta de AY24 con AR_ARS=0 varía AR_USD cuando no debería.

### Consulta de balance a fecha dada
```javascript
function getBalanceAtDate(ticker, date) {
  const inst = _ledger[ticker];
  if(!inst) return null;
  let lastSnap = null;
  for(const t of inst.txns) {
    if(t.fecha <= date) lastSnap = t._snap;
    else break;
  }
  return lastSnap; // { balAR, balEEUU, balTotal, avgAR_usd, avgAR_ars, ... }
}
```

### Orden de construcción en `processAndRefresh`
**CRÍTICO:** `_ledger` se construye **antes** de `buildPortfolio`, porque `isEffectivelyClosed` consulta `_ledger` para decidir si un instrumento está cerrado.

```javascript
function processAndRefresh(rawMerged) {
  _historicalTxns = rawMerged;
  _ledger = buildLedger(rawMerged, _fxHistory);  // PRIMERO
  const deduped = deduplicateMEP(rawMerged);
  const clean = canonicalizeTickers(deduped);
  const { usCards, arCards, ... } = buildPortfolio(clean, _fxHistory);  // DESPUÉS
  ...
}
```

### Rebuild en `fetchFXHistory`
Cuando llega el FX history, se reconstruye el ledger y luego se reconstruye el portfolio completo (para que `isEffectivelyClosed` re-evalúe con el ledger correcto):
```javascript
_ledger = buildLedger(_historicalTxns, _fxHistory);
// rebuild portfolio + rebuildEspeciesDropdown
```

### `isEffectivelyClosed` — modificado
Antes de declarar stale por año, consulta `_ledger`:
```javascript
if(years.length && Math.max(...years) <= STALE_CUTOFF_YEAR) {
  if(_ledger && _ledger[ticker] && _ledger[ticker].totals.balTotal > 0) return false;
  return true;
}
```

---

## Tab 🔍 Especies — migrado al ledger

### Fuente de datos: `_ledger` (no `_instruments`)
`renderEspecies()` ahora lee de `_ledger[ticker]` en lugar de `_instruments[ticker]`. Los balances por fila se leen de `t._snap` del ledger — son correctos porque `buildLedger` maneja MEPs y dobles filas correctamente.

### Columna nueva: Bal Total
Sección **Total** al final de la tabla (violeta), con una columna `Bal Total = balAR + balEEUU`. Refleja el balance global del instrumento en cada fecha.

### Dropdown — fuente: `_ledger`
`rebuildEspeciesDropdown` usa `Object.keys(_ledger)` como fuente para los tickers históricos. Esto incluye instrumentos que `buildPortfolio` descarta (ej: AY24 con balance 0 — aparece en "Históricas / cerradas").

---

## Cálculo de precio promedio — `buildInstrument()` + `calcBuyCost()`

### Método: VWAP promedio móvil
El precio promedio se calcula con promedio ponderado móvil. Al vender, el costo acumulado se reduce proporcionalmente.

### Segregación por cuenta (`acctState`) — 3 slots
`buildInstrument` trackea balance y costo por separado para `EEUU`, `AR_ARS`, `AR_USD`. El slot `AR_ARS` acumula `costNoComis_usd` usando FX histórico por compra.

### `calcBuyCost(t, withComissions)`
- Sin comisiones: `|monto| - comision - iva - otros_imp` (acciones) / `precio × cantidad / 100` (bonos)
- Con comisiones: `|monto|` (acciones) / igual (bonos)

### `AR_BLENDED_USD`
Avg blended en USD para instrumentos con mezcla AR_ARS + AR_USD. Race condition resuelta con `rebuildBlendedAvgs()` al final de `fetchFXHistory`.

---

## Pipeline de procesamiento

### 1. Parse → `parseBalanzRowsFromHTML()` o `parseBalanzRows()`
- Broker HTML → `DOMParser` (preserva strings AR como "10.000,00")
- XLSX → SheetJS con `raw:true`

### 2. Merge → `mergeTransactions()`
Clave primaria: `mov|fecha|cuenta|nro_mov` si `nro_mov != '0'`, sino fallback por campos.

### 3. Deduplicación MEP → `deduplicateMEP()`
Solo deduplicar si hay filas del mismo grupo en cuentas DISTINTAS (`accounts.size > 1`).

### 4. Canonicalización → `canonicalizeTickers()`
```javascript
const INSTRUMENT_GROUPS = {
  AL30:['AL30','AL30D','AL30C'], AY24:['AY24','AY24D','AY24C'],
  GD30:['GD30','GD30D'], AO20:['AO20','AO20D'],
  MELI:['MELI','MELID'], VIST:['VIST','VISTD'],
  SPY:['SPY','SPYD'], PARY:['PARY','PARYD'],
};
```

### 5. Posiciones → `buildPortfolio(clean, fxHistory)`
**CRÍTICO:** función pura — no muta globals. Segura para llamar con subsets.

**`isEffectivelyClosed()`:**
1. Stale: `Math.max(...years) <= 2020` — pero si `_ledger[ticker].totals.balTotal > 0`, no es stale
2. Fully amortized: buys + Pago de Amortización ≥ 80% del costo, sin sells

---

## Cálculo de cash — `buildCash()` y `buildAuditRows()`

**CASH_SKIP_OPS:** `Transferencia de Titulos IN -`, `Depósito por transferencia interna`, `Extracción por Transferencia interna`

**Cauciones en Dólares:** `buildUsdCaucionMovs()` identifica `nro_mov` de cauciones en dólares y excluye sus filas del cash.

**CRÍTICO — `buildAuditRows(clean, rawClean, usdCaucionMovs)`:**
- `clean`: deduped + canonicalized → para mostrar en tabla
- `rawClean`: pre-dedup + canonicalized → para calcular saldo cash correcto

---

## Precios y datos externos — Supabase + GitHub Actions

### Supabase
- **Proyecto:** portfolio-tracker (`ghxisfkgwfqqxndfpmgp.supabase.co`)
- **Anon key:** `sb_publishable_s5hodNiLD-8_OwX8NWHfRw_S1uylN2Y`
- **Tablas:** `instruments`, `prices`, `nra_overrides`, `fx_rates`, `portfolio_snapshots`

### Fetch en el dashboard:
- `fetchLatestPrices()` — último EOD + NRA overrides + recálculo cash post-overrides
- `fetchPricesHistory()` — histórico completo
- `fetchFXHistory()` — histórico FX + `rebuildBlendedAvgs()` + rebuild ledger + rebuild portfolio
- `fetchSnapshots()` — snapshots guardados

**`fetchFXHistory` usa `order=date.desc`** — intencional para traer los más recientes dentro del cap de Supabase (1000 filas).

---

## Tab 📈 Evolución

### `recalcSnapshots()`:
Por cada fecha en `_cashDaily`: reconstruye posiciones usando `avgByAcct` por slot, valoriza, obtiene FX, guarda snapshot. Requiere `_fxHistory` cargado.

### `renderEvolChart()`:
- 🇦🇷 `arUSD = ar_ars / fx_rate + ar_usd` — color `#3d7fff`
- 🇺🇸 `usUSD = us_usd` — color `#ef4444`
- ∑ `totalUSD = arUSD + usUSD` — color `#a78bfa`

---

## Checkpoints de validación

- 31/12/2011: ARS 42.412,60 ✅ | USD 0,00 ✅
- 11/09/2019: ARS 72,58 ✅ | USD 1,35 ✅
- NU precio promedio s/comis: ~11,77 USD ✅
- Argentina USD cash final (25/03/2026): ~U$S 261,51 ✅
- ADRDOLA posición abierta: ~297.952 cuotapartes ✅
- BDC20: cerrada (stale) ✅
- SPY dividendo 5/8/20: monto neto 20,32 → NRA estimada 6,15 ✅
- MELI precio promedio s/comis (cuenta EEUU): ~2.006,50 USD ✅
- GLOB avg AR (2 compras ARS, 88+229 unidades): ~$9.484 ✅
- VIST avg blended USD (153 unidades ARS + 103 unidades AR_USD, 24/10/25): ~U$S 13,5x ✅
- **AY24 balance total: 0 ✅** (MEP 11/09/19 cierra correctamente en 0)

---

## Cómo pedir cambios en un chat nuevo

Pegá este documento + el HTML al inicio del chat.

**Reglas para cambios:**
- Siempre editar el archivo existente (no reescribir desde cero)
- Antes de cualquier fix de cash, validar con Python usando el .xls directamente
- El archivo del broker es HTML disfrazado — **nunca procesar con SheetJS directamente**
- `buildPortfolio(clean, fxHistory)` es pura — segura para llamar con subsets; siempre pasar `_fxHistory`
- `recalcSnapshots()` requiere `_fxHistory` cargado (el guard lo verifica)
- El dashboard NO funciona en el sandbox de Claude (CSP bloquea Supabase). Siempre testear desde GitHub Pages
- El PATCH (no DELETE) es intencional para limpiar overrides — RLS no permite DELETE con anon key
- **`buildLedger` se llama ANTES de `buildPortfolio` en `processAndRefresh` — no cambiar este orden**
- **`renderEspecies()` lee de `_ledger[ticker]`, no de `_instruments[ticker]` — no revertir**
- **`rebuildEspeciesDropdown()` usa `Object.keys(_ledger)` como fuente — no `AR`/`US`**
- **`isEffectivelyClosed` consulta `_ledger` antes de declarar stale — no eliminar ese check**
- **`fetchFXHistory` reconstruye `_ledger` + portfolio + dropdown al resolverse — no eliminar**
- **`buildLedger` opera sobre rawTxns (pre-dedup) — nunca pasarle txns ya deduplicadas**
- **`movesTitle()` determina qué fila mueve títulos — la fila de comisión (misma op, cuenta ARS) NO mueve**
- **Ventas desde AR: consumir del slot propio primero, luego del otro slot AR si no alcanza**
- **Ventas desde EEUU: transferencia implícita AR→EEUU proporcional antes de la venta**
- **`avgByAcct` incluye hasta 4 slots: `AR_ARS`, `AR_USD`, `EEUU`, `AR_BLENDED_USD`**
- **El avg en `makeCard` viene de `d.avgByAcct[acctKey]` — no cambiar esto**
- **`fetchLatestPrices()` recalcula cash post-overrides — no eliminar**
- **`fetchFXHistory()` llama `rebuildBlendedAvgs()` al final — no eliminar**
- **FX siempre desde `_fxHistory`/`_fxLatest` — no agregar inputs manuales**
- **`exportEspeciesXLS()` lee de `_especiesRows`/`_especiesState`, no del DOM**
- **`fetchFXHistory` usa `order=date.desc` — intencional**
- **`calcBuyCost(t, false)` descuenta comis + IVA + otros_imp. `calcBuyCost(t, true)` = `|monto|`**

---


## Tab 🔍 Especies — detalle de arquitectura

### Layout de la tabla — columnas dinámicas:

El layout se adapta según qué sub-cuentas tenga el instrumento. Flags: `hasAR`, `hasEEUU`, `hasAR_ARS`, `hasAR_USD`.

**Grupo 🇦🇷 Argentina** (solo si `hasAR`):
- Sub-sección **$** (solo si `hasAR_ARS`): C $ · V $ · Precio $ · Monto $
- Sub-sección **U$S** (solo si `hasAR_USD`): C U$S · V U$S · Precio U$S · Monto U$S
- Columnas compartidas: Xfer. · Amort. · Bal AR · **Avg $** · **Avg U$S** · Valoriz. ARS · Valoriz. USD

**Grupo 🇺🇸 EEUU** (solo si `hasEEUU`):
- Compras · Ventas · Xfer. · Amort. · Bal EEUU · Precio USD · Monto USD · Avg USD · Valoriz. USD

**Grupo Total** (siempre): Bal Total (violeta)

**FX** — columna final

### Separadores visuales:
- **`esp-divider-soft`** (`1px rgba(107,122,153,.35)`): entre sub-secciones internas
- **`esp-divider-strong`** (`2px solid var(--muted)`): entre grupos Argentina / EEUU / Total

### Monto en Especies:
- **Compras:** `calcBuyCost(t, true)` = `|monto|` del broker (incluye comisión, IVA y otros impuestos)
- **Ventas:** `Math.abs(t.monto)` directamente

### Export XLS (`exportEspeciesXLS()`):
- Botón `↓ XLS` aparece solo cuando hay ticker seleccionado
- **Lee de `_especiesRows` y `_especiesState`** — NO scrapea el DOM
- Columnas condicionales según `hasAR_ARS`, `hasAR_USD`, `hasEEUU`
- Nombre del archivo: `Especies_TICKER_YYYY-MM-DD.xlsx`

### Globals de estado:
```javascript
let _especiesRows  = [];  // rows del último renderEspecies (para XLS export)
let _especiesState = {};  // {ticker, hasAR, hasEEUU, hasAR_ARS, hasAR_USD, opsLen}
```

### Funciones clave:
- `rebuildEspeciesDropdown()` — repuebla el selector con optgroups, fuente: `_ledger`
- `renderEspecies()` — lee de `_ledger[ticker].txns`, agrupa por fecha en `dateBuckets`, renderiza tabla
- `exportEspeciesXLS()` — genera y descarga el XLSX desde `_especiesRows`
- `rebuildBlendedAvgs()` — recalcula `AR_BLENDED_USD` en `_instruments` tras `fetchFXHistory`

---

## Tab 💱 Tipo de Cambio

Tab con la serie histórica de FX USD/ARS desde la tabla `fx_rates` de Supabase.

### Columnas:
- **Fecha** — `dd/mm/yyyy`
- **ARS / USD** — tasa de cambio
- **Var. diaria** — variación % vs. la entrada anterior en el histórico completo
- **Var. 30d** — variación % vs. la entrada más cercana hace 30 días o más

### UX:
- Filtro por año (dropdown) y búsqueda por texto en fecha
- Orden clickeable por Fecha o ARS/USD, con toggle asc/desc
- Se popula automáticamente cuando `fetchFXHistory()` termina y al hacer click en el tab

### CRÍTICO — límite de filas Supabase:
Supabase cappea en 1000 filas por request. `fetchFXHistory` usa `order=date.desc` para traer las 1000 entradas **más recientes**. Las variaciones se calculan siempre contra el histórico completo cargado.

---

## GitHub Actions

- **Repo:** github.com/julianfromarg/portfolio-tracker-prices (privado)
- **Cron:** lunes a viernes 21:00 UTC (18:00 ART)
- **Script:** `update_prices.py` — fetcha Yahoo Finance, upsert en Supabase
- **Tickers US activos:** `NU, STRC, VGSH, SHV, EWZ, IREN, MSTR, MELI, IVV, SPY, VIST`

---

## Fase 2 — pendiente

- **Precios BYMA (Argentina):** conectar IOL API para CEDEARs, acciones y bonos AR
- **Backfill BYMA:** histórico desde 2020
- **FX automático:** automatizar ingreso diario del tipo de cambio en `fx_rates`

## Fase 3 — pendiente

- **Autenticación de usuarios:** Supabase Auth
- **Multi-usuario real:** `user_id` en Supabase, sesiones ligadas al usuario
- La arquitectura actual (session_id local) está diseñada para migrar a esto sin reescribir

---

## Bugs resueltos — no repetir

### Parseo y pipeline

| Bug | Causa | Fix |
|---|---|---|
| Fechas invertidas | SheetJS con `raw:false` | Usar `raw:true` + `XLSX.SSF.parse_date_code()` |
| "Remuneración de Saldos Líquidos" no aparece | Filtro por `nro_mov` | Cambiar a `if(!row[COL.tipo_mov])` |
| Transacciones canceladas contabilizan | Estado "Cancelada" no filtrado | `if(estado === 'Cancelada') continue` |
| Dos ventas TVPP descartadas | `deduplicateMEP` sin chequeo de cuenta | Solo deduplicar si `accounts.size > 1` |
| Cash ARS incorrecto | `buildAuditRows` usaba `clean` post-dedup | Pasar `rawClean` pre-dedup |
| Números con miles truncados | SheetJS trunca `"10.000,00"` → `10` | Usar `DOMParser` (`parseBalanzRowsFromHTML()`) |
| Cantidades con punto de miles | `numArg` no reconocía enteros con miles | Agregar regex `/^-?\d{1,3}(\.\d{3})+$/` |

### Posiciones y balances

| Bug | Causa | Fix |
|---|---|---|
| AY24 no aparecía en tab Especies | `isEffectivelyClosed` marcaba como stale porque última op <= 2020 | Consultar `_ledger` antes de declarar stale |
| Bal AR = 0 en todas las filas de AY24 | `renderEspecies` leía de `_instruments` que tenía balance 0 por `deduplicateMEP` | Migrar `renderEspecies` a `_ledger` |
| MEP 11/09/19: venta de AY24 vaciaba AR_USD | Lógica de pool proporcional para ventas AR consumía de ambos slots | Consumir del slot propio primero, fallback al otro slot AR |
| Precio promedio bonos incorrecto | Bonos cotizan por lámina de 100 | `calcBuyCost`: `precio × cantidad / 100` |
| Tickers viejos con posición abierta | Historial incompleto | `isEffectivelyClosed()`: stale `<= 2020` + check ledger |
| BDC20 aparece como abierta | Check stale era `< 2020` | Cambiar a `<= 2020` |
| ADRDOLA no aparece | Cantidad `"677.225"` parseada mal | Fix en `numArg()` para enteros con miles |
| Precio promedio incorrecto tras ventas parciales | `avg = totalCostNoComis / bought` ignoraba ventas | VWAP móvil proporcional |
| Precio promedio incorrecto para tickers split AR/EEUU | `buildInstrument` mezclaba montos ARS y USD | 3 slots: `EEUU`, `AR_ARS`, `AR_USD` |
| Total Argentina (USD) mostraba ~U$S 32M | Footer filtraba por `accounts.includes('USD')` | `calcArSlots()` itera `avgByAcct.AR_ARS` y `avgByAcct.AR_USD` directamente |
| Avg blended incorrecto (race condition) | `buildInstrument` corría con `_fxHistory={}` vacío | `rebuildBlendedAvgs()` al final de `fetchFXHistory` |

### Snapshots / Evolución

| Bug | Causa | Fix |
|---|---|---|
| `recalcSnapshots` llama función inexistente | Referencia a `buildPortfolioLocal` | Usar `buildPortfolio()` directamente |
| Atribución incorrecta posiciones AR | `d.accounts.some(a => a.includes('ARS'))` mezclaba | Usar `avgByAcct.AR_ARS` y `avgByAcct.AR_USD` directamente |
| Snapshots con `fx_rate = 1` | `fetchFXHistory` async no terminaba | Guard en `recalcSnapshots()` |

### Cash

| Bug | Causa | Fix |
|---|---|---|
| Transferencias internas duplican cash | Aparecen en dos cuentas | Agregar a CASH_SKIP_OPS |
| Caución en Dólares distorsiona saldos | Broker registra -USD y +ARS | `buildUsdCaucionMovs()` + `isCashSkip()` |
| Cash EEUU incorrecto (factor ~10x) | NRA no descontada | `calcNRA()` en `buildCash()` y `buildCashDaily()` |
| Cash incorrecto al abrir (overrides ignorados) | Race condition con `fetchLatestPrices` | `fetchLatestPrices()` recalcula cash post-overrides |

### NRA overrides

| Bug | Causa | Fix |
|---|---|---|
| "Limpiar" no persistía | DELETE sin permisos RLS | PATCH `monto_override = NULL` |
| Override no impactaba cash | `saveNRAOverride` llamaba `cSortApply` | Cambiar a `cFilter()` |

### UI / Tab

| Bug | Causa | Fix |
|---|---|---|
| Tab activo no persiste | `act-portfolio` hardcodeado | `switchTab` guarda en `localStorage('active_tab')` |
| Dropdown Especies vacío | Solo se llamaba en `processAndRefresh` | También en `switchTab` cuando key === `'especies'` |
| Instrumentos cerrados no aparecen en dropdown | Leía de `AR`/`US` | Cambiar fuente a `Object.keys(_ledger)` |

### FX / Supabase

| Bug | Causa | Fix |
|---|---|---|
| Tab FX solo mostraba hasta 2023 | `order=date.asc&limit=5000` traía las más antiguas | `order=date.desc` con `limit=10000` y `Range: 0-9999` |

### Export Especies XLS

| Bug | Causa | Fix |
|---|---|---|
| Fechas exportadas como número | `parseFloat` convertía `"28/03/2026"` → `28` | Leer de `_especiesRows` directamente |
| Montos exportados como texto | `"U$S 1.789,74"` no parseaba | Leer valores numéricos crudos |
