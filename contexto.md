# Contexto: Portfolio Dashboard — Balanz / Julian
## Versión: 05/04/2026 v7

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
| `index.html` | El dashboard completo (en repo portfolio-tracker). ~4350 líneas. |
| `MovimientosHistoricos_Completo.xls` | Export de Balanz: Reportes → Movimientos Históricos |
| `contexto.md` | Este archivo |

---

## Estructura del dashboard

### 8 tabs:
- **📊 Portfolio** — Tab unificado con toggle 🇦🇷 Argentina / 🇺🇸 EEUU (USD). Tabla de posiciones con date picker para ver el estado en cualquier fecha histórica. Prices live para EEUU. Argentina con toggle de modo (moneda origen / todo ARS / todo USD). FX siempre desde `fx_rates` en Supabase.
- **📋 Transacciones** — Tabla audit completa con filtros multi-valor (cuenta, operación, año, mes — cada uno es un dropdown con checkboxes), agrupación, columnas Saldo cash · Ret. NRA · Otros Imp. · Nro. Boleto · Nro. Mov. y botón `↓ CSV`.
- **💵 Saldo Cash** — Tabla de saldos diarios por cuenta.
- **🏛 Ret. NRA** — Tabla de retenciones NRA estimadas para dividendos EEUU, con filtros, override por transacción y botón `↓ CSV`.
- **📈 Evolución** — Gráfico de línea con 3 series independientes: 🇦🇷 Argentina (azul `#3d7fff`), 🇺🇸 EEUU (rojo `#ef4444`), ∑ Total (violeta `#a78bfa`). Sin fill. Toggle individual por serie. Zoom, pan, selector de rango 1M/3M/6M/1Y/MAX.
- **🔍 Especies** — Tab de auditoría por instrumento. Selector dividido en 5 dropdowns por categoría: Acciones · Bonos · Letras · FCI · Otros.
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

## Tab Portfolio — diseño ★ REFACTORIZADO v7

### Fuente de datos — CRÍTICO
**El tab Portfolio ya NO lee de `AR[]`/`US[]` (output de `buildPortfolio`).** Lee directamente de `_ledger` a través de `getPortfolioStateAtDate(fecha)`. Esta es la fuente de verdad canónica — los mismos datos que usa el tab Especies.

### Date picker
- `<input type="date" id="portfolio-date">` en la toolbar — default: hoy
- Botón "Hoy" para resetear la fecha a hoy
- Variable de estado: `_portfolioDate` (string `YYYY-MM-DD`)
- Al cambiar la fecha: los tickers mostrados, balances, precios promedios, precio actual, Gan./Pérd., Valor Tenencia y caja se recalculan para esa fecha

### `getPortfolioStateAtDate(fecha)`
Itera `Object.keys(_ledger)`, para cada ticker busca el último `_snap` en `t.fecha <= fecha`. Solo incluye tickers con `balAR > 0` o `balEEUU > 0` en ese snapshot. Para fechas intermedias sin movimiento usa el último snap anterior.

### Columnas de la tabla:
| Columna | Descripción |
|---|---|
| Instrumento | Ticker canonicalizado |
| Cantidad | Balance del slot a la fecha (del `_snap` del ledger) |
| P. Prom. s/comis. | Precio promedio sin comisiones — del `_snap` del ledger a esa fecha |
| P. Prom. c/comis. | Precio promedio con comisiones — del `_snap` del ledger a esa fecha (ver abajo) |
| Precio Actual | Precio de mercado EOD a la fecha seleccionada (EEUU); `—` para AR |
| Gan./Pérd. | (Precio a la fecha − P. Prom. s/comis.) × cantidad + % (solo EEUU con precio real) |
| Valor Tenencia | Precio a la fecha × cantidad (solo EEUU) |

### Avg s/comis — fuente: `_snap` del ledger
Lee directamente del `_snap` del ledger a la fecha seleccionada:
- **AR modo origen:** `snap.AR_ARS.avgNoC` ($) y `snap.AR_USD.avgNoC` (U$S)
- **AR modo ARS:** `snap.avgAR_ars` (blended en $, FX histórico por compra)
- **AR modo USD:** `snap.avgAR_usd` (blended en U$S, FX histórico por compra)
- **EEUU:** `snap.EEUU.avgNoC` (U$S)

### Avg c/comis — fuente: `_snap` del ledger (desde v7)
También viene del `_snap` del ledger. Los campos nuevos en `snapTotals()`:
- **AR modo origen:** `snap.AR_ARS.avgWithC` ($) y `snap.AR_USD.avgWithC` (U$S)
- **AR modo ARS:** `snap.avgAR_ars_wc` (blended en $, FX histórico por compra)
- **AR modo USD:** `snap.avgAR_usd_wc` (blended en U$S, FX histórico por compra)
- **EEUU:** `snap.avgEEUU_usd_wc` (U$S)

**CRÍTICO:** el avg c/comis YA NO viene de `_instruments[ticker].avgByAcct`. Viene del snap del ledger. Funciona para fechas históricas y para tickers cerrados hoy que tenían posición en el pasado.

### Precio actual y P&L (EEUU)
- Usa `getPriceForDate(ticker, _portfolioDate, avgRef)` — precio a la fecha seleccionada, no siempre "hoy"
- Si `_portfolioDate` es una fecha pasada, muestra el precio de ese día

### Cash histórico
En `renderPortfolioTable()` el cash también se recalcula a la fecha seleccionada:
```javascript
const txnsUpTo = _historicalTxns.filter(t => t.fecha <= fecha);
const { balances } = buildCash(txnsUpTo, usdCaucionMovs);
```
Refleja el saldo de caja real en esa fecha, no el de hoy.

### Total Argentina — toggle 3 modos:
- **Moneda origen:** ARS en ARS (suma `snap.AR_ARS.avgNoC × bal`) + USD en USD (suma `snap.AR_USD.avgNoC × bal`)
- **Todo ARS:** `snap.avgAR_ars × balAR` + caja en ARS + caja USD convertida
- **Todo USD:** `snap.avgAR_usd × balAR` + caja en USD + caja ARS convertida
- FX para conversiones: `getCurrentFX()` (del input o `_fxLatest`)
- El label del footer muestra la fecha si no es hoy

### Total EEUU:
- `getPriceForDate(ticker, _portfolioDate, avgRef)` por posición + caja USD a la fecha
- `*` cuando alguna posición usa precio estimado al costo

---

## Instrument Ledger — `buildLedger()` ★ FUENTE DE VERDAD

### Concepto
`buildLedger(rawTxns, fxHistory)` es la **fuente de verdad para balances y precios promedio**. Opera sobre `rawTxns` crudos (pre-`deduplicateMEP`, pre-`canonicalizeTickers`). Es la fuente que usan tanto el tab Especies como el tab Portfolio.

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
    AR_ARS: { bal, costARS, costUSD, costWithC, costARS_withC, costUSD_withC },
    AR_USD: { bal, costUSD, costWithC },
    EEUU:   { bal, costUSD, costWithC },
  },
  totals: { balAR, balEEUU, balTotal, avgAR_usd, avgAR_ars, avgEEUU_usd },
  meta: { isClosed, isStale, isBond, firstDate, lastDate, opCount },
}
```

### Nuevos campos en slot AR_ARS (v7)
```javascript
costARS_withC   // costo acumulado en ARS con comisiones
costUSD_withC   // costo acumulado en USD con comisiones (usando FX al momento de cada compra)
```
Ambos se acumulan en BUY, se reducen proporcionalmente en SELL (todos los pasos), XFER_IN y Pago de Amortización — igual que `costARS` y `costUSD`.

### `snapTotals()` — campos del _snap (v7 actualizado)
Cada txn del ledger guarda `_snap` con el estado completo después de esa operación:
```javascript
{
  AR_ARS: { bal, avgNoC, avgWithC },      // avgWithC = costARS_withC / bal
  AR_USD: { bal, avgNoC, avgWithC },      // avgWithC = costWithC / bal (en USD)
  EEUU:   { bal, avgNoC, avgWithC },      // avgWithC = costWithC / bal (en USD)
  balAR, balEEUU, balTotal,
  avgAR_usd,     // blended AR sin comis, en USD (FX histórico por compra)
  avgAR_ars,     // blended AR sin comis, en ARS
  avgAR_usd_wc,  // blended AR con comis, en USD (FX histórico por compra)
  avgAR_ars_wc,  // blended AR con comis, en ARS
  avgEEUU_usd,   // EEUU sin comis, en USD
  avgEEUU_usd_wc,// EEUU con comis, en USD
  fx,            // FX a esa fecha
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
**Venta desde AR:**
1. Consumir del slot propio primero
2. Si no alcanza, consumir del otro slot AR
3. Si todavía no alcanza, transferencia implícita EEUU → AR

**Venta desde EEUU (sufijo C, ej AY24C):**
1. Transferencia implícita AR → EEUU proporcional (solo si `isMEP_C`)
2. Venta normal desde EEUU

**Reducción proporcional en ventas — AR_ARS:** se reducen `costARS`, `costUSD`, `costWithC`, `costARS_withC`, `costUSD_withC` todos con el mismo ratio. Reset a 0 si balance llega a 0.

### Consulta de balance a fecha dada
```javascript
// Patrón usado por getPortfolioStateAtDate y Especies
let lastSnap = null;
for(const t of inst.txns) {
  if(t.fecha > fecha) break;
  if(t._snap) lastSnap = t._snap;
}
// lastSnap contiene estado completo a esa fecha
```

### Orden de construcción en `processAndRefresh`
**CRÍTICO:** `_ledger` se construye **antes** de `buildPortfolio`.
```javascript
_ledger = buildLedger(rawMerged, _fxHistory);  // PRIMERO
const { usCards, arCards, ... } = buildPortfolio(clean, _fxHistory);  // DESPUÉS
```

### Rebuild en `fetchFXHistory`
Cuando llega el FX history, se reconstruye el ledger y el portfolio completo.

---

## Tab 🔍 Especies — detalle de arquitectura

### Fuente de datos: `_ledger` (no `_instruments`)
`renderEspecies()` lee de `_ledger[ticker]`. Los balances por fila se leen de `t._snap` del ledger.

### Selectores por categoría — 5 dropdowns
`rebuildEspeciesDropdown()` puebla 5 selectores independientes: `esp-sel-acciones`, `esp-sel-bonos`, `esp-sel-letras`, `esp-sel-fci`, `esp-sel-otros`. Elegir en uno limpia los demás (`espCatChange(cat)`). El ticker activo se obtiene con `espGetSelectedTicker()`.

**Filtro "instrumento real":** solo aparecen tickers que tienen al menos una txn con `BUY_OPS` en `_ledger[tk].txns`.

**Clasificación `espCategorizeTicker(tk)` — en orden de prioridad:**
1. **Letras** — ticker contiene `-`
2. **Bonos** — ticker está en `BOND_TICKERS`
3. **FCI** — tiene txn con `Suscripción FCI` o `Rescate FCI`
4. **Acciones** — alphanumeric puro `/^[A-Z0-9]+$/`
5. **Otros** — fallback

### Layout de la tabla — columnas dinámicas:
Flags: `hasAR`, `hasEEUU`, `hasAR_ARS`, `hasAR_USD`.

**Grupo 🇦🇷 Argentina** (solo si `hasAR`):
- Sub-sección **$** (solo si `hasAR_ARS`): C $ · V $ · Precio $ · Monto $
- Sub-sección **U$S** (solo si `hasAR_USD`): C U$S · V U$S · Precio U$S · Monto U$S
- Columnas compartidas: Xfer. · Amort. · Bal AR · **Avg $** · **Avg U$S** · Valoriz. ARS · Valoriz. USD

**Grupo 🇺🇸 EEUU** (solo si `hasEEUU`):
- Compras · **Xfer.** · Ventas · Amort. · Bal EEUU · Precio USD · Monto USD · Avg USD · Valoriz. USD

**Columna Xfer.**:
- Transferencia implícita del ledger (`_xferAR`/`_xferEEUU`): `(N)` rojo si sale, `+N` verde si entra
- Sin transferencia implícita: XFER_IN real del broker o `—`

**Grupo Total** (siempre): Bal Total (violeta) · FX

### Export XLS (`exportEspeciesXLS()`):
- Lee de `_especiesRows` y `_especiesState` — NO scrapea el DOM
- Columnas condicionales según `hasAR_ARS`, `hasAR_USD`, `hasEEUU`

### Globals de estado:
```javascript
let _especiesRows  = [];  // rows del último renderEspecies (para XLS export)
let _especiesState = {};  // {ticker, hasAR, hasEEUU, hasAR_ARS, hasAR_USD, opsLen}
```

### Funciones clave:
- `rebuildEspeciesDropdown()` — repuebla los 5 selectores, fuente: `_ledger`
- `espCatChange(cat)` — limpia los otros 4 selectores y llama `renderEspecies()`
- `espGetSelectedTicker()` — escanea los 5 selectores para encontrar el valor activo
- `renderEspecies()` — lee de `_ledger[ticker].txns`, agrupa por fecha en `dateBuckets`
- `exportEspeciesXLS()` — genera y descarga el XLSX desde `_especiesRows`
- `rebuildBlendedAvgs()` — recalcula `AR_BLENDED_USD` en `_instruments` tras `fetchFXHistory`

---

## Tab 📋 Transacciones — detalle

### Columnas de la tabla:
Fecha · Cuenta · Instrumento · Operación · Cantidad · Precio · Monto · Comisión · IVA · **Otros Imp.** · Ret. NRA · Saldo Cash · Nro. Boleto · **Nro. Mov.**

### Filtros multi-valor (dropdown con checkboxes):
- **Cuenta** — `ms-panel-cuenta`
- **Operación** — `ms-panel-op`: lista fija + opción "Otras" (mapea a `OTHER_OPS`)
- **Año** — `ms-panel-year`: poblado dinámicamente por `rebuildYearDropdown()`
- **Mes** — `ms-panel-month`

**Funciones:** `msGetSelected(key)`, `msUpdateBadge(key)`, `msDrillToggle(key)`, `msChange(key)`

**Lógica OR** dentro de cada filtro, **AND** entre filtros distintos.

**CRÍTICO:** `rebuildYearDropdown()` puebla `ms-panel-year` con checkboxes — no hay `<select id="aYear">`.

---

## Cálculo de avg — resumen

### Sin comisiones (`avgNoC`)
```
Acciones/CEDEARs: |monto| - comision - iva - otros_imp
Bonos:            precio × cantidad / 100
```

### Con comisiones (`avgWithC`)
```
Acciones/CEDEARs: |monto| (incluye todo)
Bonos:            precio × cantidad / 100 (igual — las comisiones van separadas)
```

### VWAP móvil (venta reduce costo proporcionalmente)
```
ratio = (balance_antes - qty_vendida) / balance_antes
costXXX *= ratio  // para todos los campos de costo del slot
```
Si balance = 0 → reset completo de todos los campos del slot.

### AR_ARS — tracking dual de costo
El slot AR_ARS trackea costos en **dos monedas** simultáneamente:
- `costARS` / `costARS_withC` — en pesos (para avg $)
- `costUSD` / `costUSD_withC` — en USD usando FX al momento de cada compra (para avg U$S blended histórico)

---

## Retención NRA — `calcNRA(t)`

Solo aplica a: `operacion === 'Pago de Dividendos' AND cuenta === 'EEUU (USD)'`

```javascript
const gross = Math.abs(monto) + Math.abs(comision) + Math.abs(iva);
const nra = -(gross * 0.30);
```

Overrides por `nro_mov` en tabla `nra_overrides` de Supabase (PK: `nro_mov, session_id`). PATCH para limpiar (no DELETE — RLS no permite DELETE con anon key).

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
- **Fila 1** en la cuenta real: títulos + monto
- **Fila 2** en Argentina (ARS): solo comisiones — NO mueve títulos

---

## Pipeline de procesamiento

### 1. Parse → `parseBalanzRowsFromHTML()` o `parseBalanzRows()`
### 2. Merge → `mergeTransactions()`
Clave primaria: `mov|fecha|cuenta|nro_mov` si `nro_mov != '0'`

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
Función pura. `isEffectivelyClosed()`:
1. Stale: `Math.max(...years) <= 2020` — pero si `_ledger[ticker].totals.balTotal > 0`, no es stale
2. Fully amortized: buys + Pago de Amortización ≥ 80% del costo, sin sells

---

## Cálculo de cash — `buildCash()` y `buildAuditRows()`

**CASH_SKIP_OPS:** `Transferencia de Titulos IN -`, `Depósito por transferencia interna`, `Extracción por Transferencia interna`

**Cauciones en Dólares:** `buildUsdCaucionMovs()` + `isCashSkip()`

**CRÍTICO — `buildAuditRows(clean, rawClean, usdCaucionMovs)`:**
- `clean`: deduped + canonicalized → para mostrar en tabla
- `rawClean`: pre-dedup + canonicalized → para calcular saldo cash correcto

---

## Precios y datos externos — Supabase + GitHub Actions

- **Proyecto:** `ghxisfkgwfqqxndfpmgp.supabase.co`
- **Anon key:** `sb_publishable_s5hodNiLD-8_OwX8NWHfRw_S1uylN2Y`
- **Tablas:** `instruments`, `prices`, `nra_overrides`, `fx_rates`, `portfolio_snapshots`

### Fetch en el dashboard:
- `fetchLatestPrices()` — último EOD + NRA overrides + recálculo cash post-overrides
- `fetchPricesHistory()` — histórico completo
- `fetchFXHistory()` — histórico FX + `rebuildBlendedAvgs()` + rebuild ledger + rebuild portfolio
- `fetchSnapshots()` — snapshots guardados

**`fetchFXHistory` usa `order=date.desc`** — intencional (cap Supabase 1000 filas, trae los más recientes).

**CRÍTICO — orden de carga:** `fetchFXHistory()` debe completar antes de `recalcSnapshots()`.

### GitHub Actions
- **Cron:** lunes a viernes 21:00 UTC (18:00 ART)
- **Tickers US activos:** `NU, STRC, VGSH, SHV, EWZ, IREN, MSTR, MELI, IVV, SPY, VIST`

---

## Tab 📈 Evolución

### `recalcSnapshots()`:
Por cada fecha en `_cashDaily`: reconstruye posiciones usando `avgByAcct` por slot, valoriza, obtiene FX, guarda snapshot. Requiere `_fxHistory` cargado.

### `renderEvolChart()`:
- 🇦🇷 `arUSD = ar_ars / fx_rate + ar_usd` — color `#3d7fff`
- 🇺🇸 `usUSD = us_usd` — color `#ef4444`
- ∑ `totalUSD = arUSD + usUSD` — color `#a78bfa`

---

## Tab 💱 Tipo de Cambio

Filtra por año y texto. Orden por fecha o tasa. Var. diaria y Var. 30d calculadas contra el histórico completo (no el subconjunto filtrado). `fetchFXHistory` usa `order=date.desc` para traer los más recientes dentro del cap.

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
- AY24 balance total: 0 ✅ (MEP 11/09/19 cierra correctamente en 0)

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
| Números con miles truncados | SheetJS trunca `"10.000,00"` → `10` | Usar `DOMParser` |
| Cantidades con punto de miles | `numArg` no reconocía enteros con miles | Regex `/^-?\d{1,3}(\.\d{3})+$/` |

### Posiciones y balances

| Bug | Causa | Fix |
|---|---|---|
| AY24 no aparecía en Especies | `isEffectivelyClosed` stale incorrecto | Consultar `_ledger` antes de declarar stale |
| Precio promedio bonos incorrecto | Bonos cotizan por lámina de 100 | `precio × cantidad / 100` |
| Tickers viejos con posición abierta | Historial incompleto | `isEffectivelyClosed()`: stale `<= 2020` + check ledger |
| ADRDOLA no aparece | Cantidad `"677.225"` parseada mal | Fix `numArg()` |
| Precio promedio incorrecto tras ventas parciales | VWAP ignoraba ventas | VWAP móvil proporcional |
| Precio promedio incorrecto tickers split AR/EEUU | Mezclaba montos ARS y USD | 3 slots: `EEUU`, `AR_ARS`, `AR_USD` |
| Avg blended incorrecto (race condition) | `buildInstrument` corría con `_fxHistory={}` | `rebuildBlendedAvgs()` al final de `fetchFXHistory` |
| GLOB con transferencia implícita incorrecta | Xfer aplicaba a toda venta EEUU | Solo si sufijo C (`isMEP_C`) |

### Cash

| Bug | Causa | Fix |
|---|---|---|
| Transferencias internas duplican cash | Aparecen en dos cuentas | `CASH_SKIP_OPS` |
| Caución en Dólares distorsiona saldos | Broker registra -USD y +ARS | `buildUsdCaucionMovs()` + `isCashSkip()` |
| Cash EEUU incorrecto (factor ~10x) | NRA no descontada | `calcNRA()` en `buildCash()` |
| Cash incorrecto al abrir | Race condition con overrides | `fetchLatestPrices()` recalcula cash post-overrides |

### NRA overrides

| Bug | Causa | Fix |
|---|---|---|
| "Limpiar" no persistía | DELETE sin permisos RLS | PATCH `monto_override = NULL` |
| Override no impactaba cash | `saveNRAOverride` llamaba función incorrecta | Cambiar a `cFilter()` |

### UI / Tab

| Bug | Causa | Fix |
|---|---|---|
| Tab activo no persiste | `act-portfolio` hardcodeado | `switchTab` guarda en `localStorage('active_tab')` |
| Dropdown Especies vacío | Solo se llamaba en `processAndRefresh` | También en `switchTab` cuando key === `'especies'` |
| Instrumentos cerrados no aparecen en dropdown | Leía de `AR`/`US` | Fuente: `Object.keys(_ledger)` |
| Tab FX solo mostraba hasta 2023 | `order=date.asc` traía los más antiguos | `order=date.desc` con `Range: 0-9999` |
| Fechas XLS exportadas como número | `parseFloat("28/03/2026")` → `28` | Leer de `_especiesRows` directamente |

---

## Cómo pedir cambios en un chat nuevo

Pegá este documento + el HTML al inicio del chat.

**Reglas para cambios:**
- Siempre editar el archivo existente con `str_replace` — no reescribir desde cero
- Leer `contexto.md` y las secciones relevantes del HTML antes de cualquier cambio
- Identificar todos los call sites de cualquier función a modificar
- El dashboard NO funciona en el sandbox de Claude (CSP bloquea Supabase) — siempre testear desde GitHub Pages
- Antes de cualquier fix de cash, validar con Python usando el .xls directamente
- El archivo del broker es HTML disfrazado — **nunca procesar con SheetJS directamente**
- Ofrecer actualizar `contexto.md` después de cada sesión de cambios

**Reglas críticas de arquitectura — NUNCA violar:**
- **`_ledger` es la fuente de verdad para balances y avgs** — Tab Portfolio y Tab Especies leen de `_ledger`, no de `AR[]`/`US[]`
- **`renderPortfolioTable()` usa `getPortfolioStateAtDate(_portfolioDate)`** — no leer de `AR[]`/`US[]` directamente
- **El avg c/comis en Portfolio viene del `_snap` del ledger** — no de `_instruments[ticker].avgByAcct`
- **`buildLedger` se llama ANTES de `buildPortfolio` en `processAndRefresh`** — no cambiar este orden
- **`buildLedger` opera sobre rawTxns (pre-dedup)** — nunca pasarle txns ya deduplicadas
- **`renderEspecies()` lee de `_ledger[ticker]`** — no de `_instruments[ticker]`
- **`rebuildEspeciesDropdown()` usa `Object.keys(_ledger)` como fuente** — no `AR`/`US`
- **`rebuildEspeciesDropdown()` puebla 5 selectores** — no hay `esp-ticker-select` único
- **`renderEspecies()` obtiene ticker con `espGetSelectedTicker()`** — no leer de un selector fijo
- **`rebuildYearDropdown()` puebla `ms-panel-year` con checkboxes** — no hay `<select id="aYear">`
- **Los filtros de Transacciones son multi-valor** — `aFilter()` usa `msGetSelected()`, no `.value`
- **`isEffectivelyClosed` consulta `_ledger` antes de declarar stale** — no eliminar ese check
- **`fetchFXHistory` reconstruye `_ledger` + portfolio + dropdown al resolverse** — no eliminar
- **`movesTitle()` determina qué fila mueve títulos** — la fila de comisión (cuenta ARS) NO mueve
- **Ventas desde AR:** consumir slot propio → otro slot AR → EEUU→AR
- **Ventas desde EEUU:** xfer implícita AR→EEUU proporcional SOLO si sufijo C (`isMEP_C`)
- **`_xferAR` y `_xferEEUU` en cada txn de venta** — los usa `renderEspecies`, no eliminar
- **`fetchLatestPrices()` recalcula cash post-overrides** — no eliminar
- **`fetchFXHistory()` llama `rebuildBlendedAvgs()` al final** — no eliminar
- **FX siempre desde `_fxHistory`/`_fxLatest`** — no agregar inputs manuales de FX
- **`exportEspeciesXLS()` lee de `_especiesRows`/`_especiesState`** — no del DOM
- **`calcBuyCost(t, false)` descuenta comis + IVA + otros_imp. `calcBuyCost(t, true)` = `|monto|`**
- **`buildPortfolio(clean, fxHistory)` es pura** — no muta globals, segura para llamar con subsets
- **`recalcSnapshots()` requiere `_fxHistory` cargado** — el guard lo verifica
- **Cash histórico en Portfolio** — se recalcula filtrando `_historicalTxns` por `t.fecha <= _portfolioDate`
