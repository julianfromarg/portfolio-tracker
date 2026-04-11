# Contexto: Portfolio Dashboard — Balanz / Julian
## Versión: 11/04/2026 v16

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
| `index.html` | El dashboard completo (en repo portfolio-tracker). ~4750 líneas. |
| `MovimientosHistoricos_Completo.xls` | Export de Balanz: Reportes → Movimientos Históricos |
| `contexto.md` | Este archivo |

---

## Estructura del dashboard

### 8 tabs:
- **📊 Portfolio** — Tab unificado con toggle 🇦🇷 Argentina / 🇺🇸 EEUU (USD). Tarjetas de resumen arriba. Tabla de posiciones con date picker para ver el estado en cualquier fecha histórica. Prices live para EEUU. Argentina con toggle de modo **Todo ARS / Todo USD** (el modo "Moneda origen" fue eliminado). FX siempre desde `fx_rates` en Supabase.
- **📋 Transacciones** — Tabla audit completa con filtros multi-valor (cuenta, operación, año, mes — cada uno es un dropdown con checkboxes), agrupación, columnas Fecha Concert. · **Fecha Liq.** · Saldo cash · Ret. NRA · Otros Imp. · Nro. Boleto · Nro. Mov. y botón `↓ CSV`. El filtro Operación incluye opciones explícitas para **Depósito de Fondos** y **Extracción de Fondos** (detectados por prefijo, no por nombre exacto del banco).
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

## Tab Portfolio — diseño ★ REFACTORIZADO v14

### Fuente de datos — CRÍTICO
**El tab Portfolio ya NO lee de `AR[]`/`US[]` (output de `buildPortfolio`).** Lee directamente de `_ledger` a través de `getPortfolioStateAtDate(fecha)`. Esta es la fuente de verdad canónica — los mismos datos que usa el tab Especies.

### Date picker
- `<input type="text" id="portfolio-date">` en la toolbar — placeholder `dd/mm/yyyy`, default: hoy
- Botón "Hoy" para resetear la fecha a hoy
- Botones `‹` y `›` para navegar de a 1 día calendario (`shiftPortfolioDate(delta)`)
- Variable de estado: `_portfolioDate` (string `YYYY-MM-DD` internamente)
- `parseDateDMY(s)` — convierte entrada `dd/mm/yyyy` → `YYYY-MM-DD`
- `fmtDateDMY(iso)` — convierte `YYYY-MM-DD` → `dd/mm/yyyy` para mostrar en el input
- Al cambiar la fecha: los tickers mostrados, balances, precios promedios, precio actual, Gan./Pérd., Valor Tenencia y caja se recalculan para esa fecha

### Toggle AR mode — SOLO 2 MODOS (v11)
El modo "Moneda origen" fue **eliminado**. Solo existen:
- `'ars'` — Todo en ARS
- `'usd'` — Todo en USD (default al abrir)

`_arMode` default: `'usd'`. `setARMode()` solo itera `['ars','usd']`. El botón `armode-origin` ya no existe.

### Tarjetas de resumen — `renderPortfolioCards()`
Función separada llamada desde `renderPortfolioTable()`. Renderiza en `<div id="portfolio-cards">` ubicado entre la toolbar y la tabla.

**Tab Argentina — 5 tarjetas:**
| Tarjeta | Clase CSS | Contenido |
|---|---|---|
| 💵 Caja ARS | `pc-cash-ars` | `$ X` + sub `≈ U$S Y` |
| 💵 Caja USD | `pc-cash-usd` | `U$S X` + sub `≈ $ Y` |
| 🏦 Total Caja | `pc-cash-total` | en modo toggle + sub `FX X` |
| 📦 Costo + Caja | `pc-costo` | `avgNoC × bal` + caja en toggle + sub `Costo: X` |
| 🏦 Portfolio Total | `pc-total` | placeholder hasta Fase 2 |
| 📈 Ganancia | `pc-pl` | placeholder hasta Fase 2 |

**Tab EEUU — 4 tarjetas:**
| Tarjeta | Clase CSS | Contenido |
|---|---|---|
| 💵 Caja USD | `pc-cash-usd` | `U$S X` |
| 📦 Costo + Caja | `pc-costo` | Costo Total + caja + sub `Costo: X` |
| 🏦 Portfolio Total | `pc-total` | Tenencias a mercado + caja + sub `Tenencias: X` |
| 📈 Ganancia | `pc-pl` | `sumPL` en USD + sub `%` |

**Firma:** `renderPortfolioCards(isAr, fecha, historicCash, sumCostoTotal, sumValorTenencia, sumPL, anyPrice)`

**CRÍTICO:** Las filas de caja ya **NO aparecen en el tbody** de la tabla — la info de caja vive únicamente en las tarjetas.

### `getPortfolioStateAtDate(fecha)`
Itera `Object.keys(_ledger)`, para cada ticker busca el último `_snap` en `t.fecha <= fecha`. Solo incluye tickers con `balAR > 0` o `balEEUU > 0` en ese snapshot. Para fechas intermedias sin movimiento usa el último snap anterior.

### Columnas de la tabla (v14) — SIN comisiones:
| Columna | Sort key | Descripción |
|---|---|---|
| Instrumento | `ticker` | Ticker canonicalizado |
| Cantidad | `net` | Balance del slot a la fecha |
| P. Prom. | `avg` | `snap.avgAR_ars/usd` o `snap.EEUU.avgNoC` — **sin comisiones** |
| Costo Total | `costoTotal` | `avgNoC × bal` en moneda del toggle — **sin comisiones** |
| Precio Actual | — | `getPriceForDate()` — solo EEUU |
| Valor Tenencia | `valorTenencia` | `precio × bal` — solo EEUU con precio real |
| Gan./Pérd. $ | — | En celda separada — solo EEUU con precio real y `!isEstimated` |
| Gan./Pérd. % | — | En celda separada — solo EEUU con precio real y `!isEstimated` |

**CRÍTICO (v14):** La columna "P. Prom. c/comis." fue **eliminada**. `costoTotal` usa `avgNoC_val × bal`, no `avgWithC_val × bal`. Esto garantiza consistencia con `getARValueAtDate` y el tab Evolución. **No restaurar `avgWithC` en la tabla Portfolio.**

AR: Precio Actual, Valor Tenencia, Gan./Pérd. $ y % quedan en `—` (preparado para Fase 2 con precios BYMA).

### Acumuladores en `renderPortfolioTable()`
```javascript
let sumCostoTotal = 0;      // suma de avgNoC × bal por posición (sin comisiones)
let sumValorTenencia = 0;   // suma de precio × bal (solo EEUU con precio real)
let sumPL = 0;              // suma de P&L (solo !isEstimated)
let anyPriceForTotal = false;
```
Estos alimentan tanto la fila "Total posiciones" del footer como `renderPortfolioCards()`.

### Fila "Total posiciones" en el footer
Muestra: Costo Total acumulado · Valor Tenencia acumulado · Gan./Pérd. $ total · Gan./Pérd. % total (vs costo).

### Cauciones en pesos (v13)
Las cauciones en pesos aparecen como filas especiales al final del tbody del tab Argentina. **No aparecen en EEUU.**

**`buildCauciones(rawTxns)`** — matchea pares por `nro_mov`:
- Apertura: `operacion === 'Caución'` AND `ticker === 'Caución en Pesos Arg.'`
- Cierre: `operacion === 'Liquidación de Caución'` AND mismo `nro_mov`
- Si no hay cierre → caución todavía activa

```javascript
{
  nro_mov,
  fechaApertura,  // fecha concert. de la Caución
  fechaCierre,    // fecha concert. de la Liquidación (null si activa)
  monto,          // Math.abs(monto de apertura) — siempre ARS
  label,          // "Caución - DD/MM/YYYY"
}
```

**Visibilidad por fecha:** una caución es visible si `fechaApertura <= _portfolioDate && (fechaCierre === null || fechaCierre > _portfolioDate)`.

**Rendering:** cada caución activa aparece como fila con:
- Costo Total = Valor Tenencia = `monto` en ARS (modo `ars`) o `monto / FX` (modo `usd`)
- Cantidad, P. Prom., Precio Actual, Gan./Pérd. = `—`

**Impacto en acumuladores:** `sumCostoTotal` y `sumValorTenencia` incluyen el monto de cada caución activa → las tarjetas (Costo+Caja, Portfolio Total) se actualizan automáticamente.

**CRÍTICO — cauciones en dólares:** se manejan por separado vía `buildUsdCaucionMovs()` y `isCashSkip()`. `buildCauciones()` solo procesa ARS.

### Avg sin comisiones — fuente: `_snap` del ledger
- **AR modo ARS:** `snap.avgAR_ars`
- **AR modo USD:** `snap.avgAR_usd`
- **EEUU:** `snap.EEUU.avgNoC`

### Precio actual y P&L (EEUU)
- Usa `getPriceForDate(ticker, _portfolioDate, avgRef)`
- `estimated: true` solo cuando no hay **ningún** precio histórico para ese ticker en `_pricesHistory`
- Si hay precio anterior a la fecha (ej: último EOD disponible), se usa y `estimated: false`
- Gan./Pérd. solo se calcula cuando `!isEstimated`

### Cash histórico — CRÍTICO (v14)
```javascript
// Fuente canónica: getCashAtDate(fecha) — lee de _cashDaily
const historicCash = getCashAtDate(fecha);
```
**NUNCA reconstruir `buildCash(txnsUpTo, ...)` inline en `renderPortfolioTable` para obtener el cash histórico.** Usar siempre `getCashAtDate()`. Ver sección "Fuentes canónicas" más abajo.

### Total Argentina (footer):
- Calculado por `getARValueAtDate(fecha, precomputed)` — **fuente canónica**
- **Todo ARS:** `totalUSD_ar × FX`
- **Todo USD:** `totalUSD_ar` directo
- FX: `getFXForDate(fecha)` — **nunca `getCurrentFX()`**

### Total EEUU (footer):
- Calculado por `getUSValueAtDate(fecha, precomputed)` — **fuente canónica**

### Re-render triggers
`renderPortfolioTable()` se llama al final de:
- `fetchLatestPrices()` — siempre
- `fetchFXHistory()` — si `_historicalTxns` cargado
- `fetchPricesHistory()` — si `_historicalTxns` cargado
- `rebuildBlendedAvgs()` — siempre

---

## ★★★ FUENTES CANÓNICAS — CASH Y VALOR DE PORTFOLIO ★★★

Esta es la sección más importante del documento. Toda discrepancia entre tabs en el pasado se originó en calcular cash o valor de portfolio en múltiples lugares con lógica ligeramente diferente. Está **terminantemente prohibido** calcular cualquiera de estos valores fuera de las funciones canónicas.

### Las tres funciones canónicas

```javascript
getCashAtDate(fecha)
  → { 'Argentina (ARS)': n, 'Argentina (USD)': n, 'EEUU (USD)': n }

getARValueAtDate(fecha, precomputedData?)
  → number  // valor total Argentina en USD

getUSValueAtDate(fecha, precomputedData?)
  → number  // valor total EEUU en USD
```

### Qué calcula cada una

**`getCashAtDate(fecha)`**
- Lee de `_cashDaily` (construido UNA SOLA VEZ en `processAndRefresh`)
- Busca el último registro con `fecha <= fecha pedida` (array ordenado desc → iterar desde el inicio)
- Es exactamente el mismo dato que muestra el tab Saldo Cash
- **Si no hay movimiento en esa fecha exacta, usa el registro anterior** — correcto porque el cash no cambia sin movimiento

**`getARValueAtDate(fecha, precomputedData?)`**
- Tenencias: `snap.avgAR_usd × balAR` para cada posición AR abierta a esa fecha (via `getPortfolioStateAtDate`)
- Cash: `getCashAtDate(fecha)` — USD directo + ARS / FX
- Cauciones ARS activas a esa fecha: `monto / FX`
- FX: `getFXForDate(fecha)` — siempre histórico, nunca `getCurrentFX()`
- `precomputedData`: objeto opcional `{ balances, cauciones }` para evitar reconstruir en loops

**`getUSValueAtDate(fecha, precomputedData?)`**
- Tenencias: `getPriceForDate(ticker, fecha, avgNoC) × balEEUU` para cada posición EEUU abierta
- Cash: `getCashAtDate(fecha)` — solo `EEUU (USD)`
- `precomputedData`: objeto opcional `{ balances }` para evitar reconstruir en loops

### Quién consume estas funciones

| Consumidor | Función usada |
|---|---|
| `renderPortfolioTable()` footer AR | `getARValueAtDate(fecha, precomputed)` |
| `renderPortfolioTable()` footer EEUU | `getUSValueAtDate(fecha, precomputed)` |
| `renderPortfolioTable()` tarjetas | `getCashAtDate(fecha)` → pasa como `historicCash` |
| `recalcSnapshots()` loop por fecha | `getARValueAtDate(date, precomputed)` + `getUSValueAtDate(date, precomputed)` |
| Tab Saldo Cash | `_cashDaily` directamente (es la misma fuente que `getCashAtDate`) |

### Regla de performance en loops

`recalcSnapshots()` itera ~300-400 fechas. Para evitar reconstruir cash N veces, pre-computa `getCashAtDate(date)` por fecha y lo pasa como `precomputed.balances`. Las cauciones también se pre-computan una sola vez antes del loop.

```javascript
// CORRECTO — en recalcSnapshots
const cauciones = buildCauciones(_historicalTxns);  // una vez, fuera del loop
for(const date of dates) {
  const precomputed = { balances: getCashAtDate(date), cauciones };
  const ar_usd = getARValueAtDate(date, precomputed);
  const us_usd = getUSValueAtDate(date, precomputed);
}
```

### Lo que está PROHIBIDO

```javascript
// ❌ NUNCA HACER ESTO — calcular cash inline fuera de getCashAtDate
const txnsUpTo = _historicalTxns.filter(t => t.fecha <= fecha);
const { balances } = buildCash(txnsUpTo, usdCaucionMovs);

// ❌ NUNCA HACER ESTO — calcular valor AR inline fuera de getARValueAtDate
let ar_usd = 0;
for(const [, inst] of Object.entries(_ledger)) { ... snap.avgAR_usd × snap.balAR ... }

// ❌ NUNCA HACER ESTO — calcular valor EEUU inline fuera de getUSValueAtDate
let us_usd = balances['EEUU (USD)'];
for(const d of usCards) { us_usd += getPriceForDate(...) × d.net; }
```

### Cuándo llamar a `buildCash` directamente

`buildCash()` y `buildCashDaily()` solo se llaman en dos lugares para **construir** `_cashDaily`:
- `processAndRefresh()` — al importar o cargar datos
- `saveNRAOverride()` — al cambiar un override (reconstruye `_cashDaily` con los nuevos valores)

En cualquier otro lugar, **usar `getCashAtDate()`**.

### Qué hacer después de cambiar la fórmula de valuación

Si se modifica `getARValueAtDate`, `getUSValueAtDate` o `getCashAtDate`:
1. Deployar el cambio
2. El tab Portfolio refleja el cambio inmediatamente (tiempo real)
3. El tab Evolución muestra datos **stale** hasta que el usuario presione **⟳ Recalcular**
4. Recordar avisar al usuario que debe recalcular

---

## Precios históricos — `fetchPricesHistory()` ★ CORREGIDO v11

### Paginación correcta
```javascript
// CORRECTO — solo Range header, sin limit en URL
const r = await fetch(
  `${SUPABASE_URL}/rest/v1/prices?select=ticker,date,close&order=ticker.asc,date.asc`,
  { headers: { ..., 'Range-Unit': 'items', 'Range': `${from}-${from+999}` } }
);
```
**CRÍTICO:** No usar `limit` en la URL junto con `Range` header — da error 416 y corta el resultado (solo traía EWZ). Mismo patrón que `fetchFXHistory` y `fetchSnapshots`.

### Estructura
```javascript
_pricesHistory = { 'NU': { '2026-04-02': 14.15, ... }, 'EWZ': {...}, ... }
```
11 tickers activos: `EWZ, IREN, IVV, MELI, MSTR, NU, SHV, SPY, STRC, VGSH, VIST`

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
Cuando llega el FX history, se reconstruye el ledger y el portfolio completo, y se llama `renderPortfolioTable()`.

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

**Nota:** `rebuildBlendedAvgs()` actualiza `AR_BLENDED_USD` en el global `_instruments` (sistema pre-v7). Desde v7 el tab Especies y el tab Portfolio leen del `_ledger` directamente — `rebuildBlendedAvgs()` ya no afecta el rendering de ninguno de los dos tabs. Se mantiene en el código pero no debe confundirse con la fuente de datos activa.

---

## Tab 📋 Transacciones — detalle

### Columnas de la tabla (v11):
Fecha Concert. · **Fecha Liq.** · Cuenta · Instrumento · Operación · Cantidad · Precio · Monto · Comisión · IVA · Otros Imp. · Ret. NRA · Saldo Cash · Nro. Boleto · Nro. Mov.

**Fecha Liq.** viene de `t.fecha_liq` tal como llega del broker. Se muestra siempre para todas las operaciones. `buildAuditRows()` la incluye en el objeto de cada fila como `fecha_liq`.

### Filtros multi-valor (dropdown con checkboxes):
- **Cuenta** — `ms-panel-cuenta`
- **Operación** — `ms-panel-op`: lista fija + **Depósito de Fondos** + **Extracción de Fondos** + opción "Otras" (mapea a `OTHER_OPS`)
- **Año** — `ms-panel-year`: poblado dinámicamente por `rebuildYearDropdown()`
- **Mes** — `ms-panel-month`

**Funciones:** `msGetSelected(key)`, `msUpdateBadge(key)`, `msDrillToggle(key)`, `msChange(key)`

**Lógica OR** dentro de cada filtro, **AND** entre filtros distintos.

**Depósitos y extracciones** se detectan por prefijo con `isDeposito(op)` y `isExtraccion(op)`:
```javascript
function isDeposito(op)   { return op && op.startsWith('Depósito de Fondos'); }
function isExtraccion(op) { return op && op.startsWith('Extracción de Fondos'); }
```
Esto cubre todas las variantes bancarias (COMAFI, SUPERVIELLE, BBVA, etc.) sin listarlas.

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
[4]  Liquidación    → fecha_liq "dd/mm/yy"
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

**CRÍTICO — gap fecha concert. vs fecha liq. en Pago de Renta:**
El broker puede registrar el Pago de Renta con `fecha` (concertación) anterior a `fecha_liq` (liquidación). El dinero no está disponible hasta `fecha_liq`. Ver sección buildCash para el tratamiento correcto.

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
  I15G8:['I15G8','I15G8-1505','I15G8-1906'],
};
```

### 5. Posiciones → `buildPortfolio(clean, fxHistory)`
Función pura. `isEffectivelyClosed()`:
1. Stale: `Math.max(...years) <= 2020` — pero si `_ledger[ticker].totals.balTotal > 0`, no es stale
2. Fully amortized: buys + Pago de Amortización ≥ 80% del costo, sin sells

---

## Cálculo de cash — `buildCash()` y `buildCashDaily()` ★ ACTUALIZADO v11

**CASH_SKIP_OPS:** `Transferencia de Titulos IN -`, `Depósito por transferencia interna`, `Extracción por Transferencia interna`

**Cauciones en Dólares:** `buildUsdCaucionMovs()` + `isCashSkip()`

### Pago de Renta — uso de fecha_liq (v11)
**CRÍTICO:** Para operaciones `Pago de Renta` donde `fecha_liq !== fecha`, tanto `buildCash` como `buildCashDaily` usan `fecha_liq` como fecha de impacto en la caja:
```javascript
const fechaCash = (t.operacion === 'Pago de Renta' && t.fecha_liq && t.fecha_liq !== t.fecha)
  ? t.fecha_liq : t.fecha;
```
Esto evita picos artificiales en el valor del portfolio histórico cuando el Pago de Renta llega (concertación) 1-2 días antes que la Amortización (liquidación).

El mismo criterio aplica al filtro `txnsUpTo` en `renderPortfolioTable()` para el cash histórico.

**CRÍTICO — `buildAuditRows(clean, rawClean, usdCaucionMovs)`:**
- `clean`: deduped + canonicalized → para mostrar en tabla
- `rawClean`: pre-dedup + canonicalized → para calcular saldo cash correcto

---

## Precios y datos externos — Supabase + GitHub Actions

- **Proyecto:** `ghxisfkgwfqqxndfpmgp.supabase.co`
- **Anon key:** `sb_publishable_s5hodNiLD-8_OwX8NWHfRw_S1uylN2Y`
- **Tablas:** `instruments`, `prices`, `nra_overrides`, `fx_rates`, `portfolio_snapshots`

### Políticas RLS por tabla
| Tabla | SELECT | INSERT | UPDATE | DELETE |
|---|---|---|---|---|
| `prices` | ✅ public | — | — | — |
| `fx_rates` | ✅ public | ✅ public | ✅ public | — |
| `nra_overrides` | ✅ public | ✅ public | ✅ public | ❌ no permitido (usar PATCH con `monto_override=null`) |
| `portfolio_snapshots` | ✅ public | ✅ public | ✅ public | ✅ public (agregado v12) |

**CRÍTICO:** El DELETE en `nra_overrides` no está permitido con anon key — siempre usar PATCH con `monto_override = null` para "limpiar" un override.

### Fetch en el dashboard:
- `fetchLatestPrices()` — último EOD + NRA overrides + recálculo cash post-overrides + `renderPortfolioTable()`
- `fetchPricesHistory()` — histórico completo paginado de a 1000 + `renderPortfolioTable()` si hay txns
- `fetchFXHistory()` — histórico FX + `rebuildBlendedAvgs()` + rebuild ledger + rebuild portfolio + `renderPortfolioTable()`
- `fetchSnapshots()` — snapshots guardados

**`fetchFXHistory` usa `order=date.desc`** — intencional (cap Supabase 1000 filas, trae los más recientes).

**CRÍTICO — orden de carga:** `fetchFXHistory()` debe completar antes de `recalcSnapshots()`.

### GitHub Actions
- **Cron:** lunes a viernes 21:00 UTC (18:00 ART)
- **Tickers US activos:** `NU, STRC, VGSH, SHV, EWZ, IREN, MSTR, MELI, IVV, SPY, VIST`

---

## Tab 📈 Evolución

### Invariante de consistencia
**Cada punto del gráfico debe coincidir exactamente con el total que mostraría Portfolio en modo "Todo USD" para esa fecha.** Portfolio, Especies y Evolución leen todos del mismo `_ledger`.

### Series del gráfico
El gráfico es de tipo mixto (line + bar). Todos los datasets declaran `yAxisID`:
- 🇦🇷 `arUSD = r.ar_usd` — eje Y izquierdo — color `#3d7fff`
- 🇺🇸 `usUSD = r.us_usd` — eje Y izquierdo — color `#ef4444`
- ∑ `totalUSD = arUSD + usUSD` — eje Y izquierdo — color `#a78bfa`
- 💰 **Flujos** — barras verdes (depósitos) y rojas (extracciones) — eje Y izquierdo — prendido por default
- 📈 **Índice base 100** — línea punteada naranja — eje Y derecho (`y2`) — apagado por default
- 💵 **Dep. Netos** — línea punteada blanca (`#ffffff`) — eje Y izquierdo — apagado por default — serie acumulada de depósitos netos en USD (ARS convertidos al FX del día, USD directo)

### Toggles
`_evolSeries` es un `Set` que controla qué series se renderizan. Estado inicial: `['ar','us','total','flujos']`.
Función: `evolToggleSeries(series)`.

**`buildDepositSeries(dates)`** — calcula la serie acumulada de Dep. Netos on-the-fly desde `_historicalTxns` + `_fxHistory`. No se guarda en Supabase. Recibe array de fechas ISO del gráfico (ventana filtrada), devuelve array de valores acumulados en USD.

### Zoom
`mode: 'xy'` en zoom y pan — permite zoom y pan en ambos ejes con scroll del mouse y click+drag.

### `recalcSnapshots()` ★ REFACTORIZADO v10, ACTUALIZADO v13
Por cada fecha en `_cashDaily`:
1. Reconstruye posiciones EEUU con `buildPortfolio` → `getPriceForDate`
2. Calcula AR en USD desde `_ledger`: `snap.avgAR_usd × snap.balAR` para cada ticker con `balAR > 0`
3. Suma cash AR: `bUSD_AR + bARS / getFXForDate(date)`
4. **Suma cauciones en pesos activas a esa fecha:** `c.monto / fx` para cada caución con `fechaApertura <= date && (fechaCierre === null || fechaCierre > date)`
5. Guarda `{ date, ar_ars: 0, ar_usd, us_usd, fx_rate }`

`buildCauciones(_historicalTxns)` se llama **una sola vez antes del loop** y se reutiliza en cada iteración.

Después del loop, calcula flujos e índice:
5. **Flujos por fecha**: itera `_historicalTxns`, filtra por `isDeposito()`/`isExtraccion()`, convierte a USD con `getFXForDate`. ARS → divide por FX. USD → directo.
6. **Índice base 100** (Modified Dietz simplificado):
   - `retorno = (total_hoy - flujo_hoy) / total_ayer - 1`
   - `idx_hoy = idx_ayer × (1 + retorno)`
   - Huecos en la serie: el retorno acumulado se aplica de una vez (correcto matemáticamente)
7. Guarda `idx` en cada snapshot → **DELETE de todos los snapshots existentes** → upsert a Supabase (columna `idx float8` en `portfolio_snapshots`)

**CRÍTICO — DELETE antes del upsert (v12):** `recalcSnapshots()` hace DELETE de todos los snapshots antes de insertar los nuevos. Esto evita que queden fechas huérfanas cuando `_cashDaily` cambia (ej: por el fix de `fecha_liq` en Pago de Renta que desplaza fechas). Sin el DELETE, los snapshots viejos persisten en Supabase y se cargan al abrir, mostrando datos incorrectos.

```javascript
// DELETE antes del upsert
await fetch(`${SUPABASE_URL}/rest/v1/portfolio_snapshots?date=gte.2000-01-01`, {
  method: 'DELETE',
  headers: { apikey: SUPABASE_ANON, Authorization: `Bearer ${SUPABASE_ANON}` }
});
```

**CRÍTICO — política RLS requerida:** la tabla `portfolio_snapshots` necesita política DELETE habilitada. Sin ella el DELETE falla silenciosamente. Las políticas actuales de la tabla son:
```sql
-- SELECT, INSERT, UPDATE ya existían
-- DELETE agregado en v12:
CREATE POLICY "public delete" ON portfolio_snapshots
FOR DELETE TO public USING (true);
```

Requiere `_fxHistory` cargado. **Después de cualquier cambio en Portfolio o Especies, hay que hacer ⟳ Recalcular.**

### `ar_ars` en snapshots siempre es 0 desde v9
`ar_usd` ya contiene el total AR en USD.

### `fetchFXHistory()` — paginación
```javascript
while(true) {
  // Range: from-(from+999)
  // Si rows.length < 1000 → última página, break
}
```
**No usar `limit` en la URL junto con `Range` header — da error 416.**

### `fetchSnapshots()` — ídem paginación
Mismo patrón que `fetchFXHistory`.

### FX histórico en Supabase
5571 filas cargadas en tabla `fx_rates`, desde 2011-01-03 hasta 2026-04-06.

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

- **Precios BYMA (Argentina):** conectar IOL API para CEDEARs, acciones y bonos AR — cuando esté, las tarjetas y columnas de Gan./Pérd. y Portfolio Total para AR dejarán de ser placeholders
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
| I15G8-1505 e I15G8-1906 aparecían como posiciones abiertas | El broker registró las Suscripciones Primarias con nombres distintos al Pago de Amortización (`I15G8`), generando tres buckets separados en `buildLedger` | Agregar `I15G8:['I15G8','I15G8-1505','I15G8-1906']` a `INSTRUMENT_GROUPS`; vaciar `IMPLICIT_CLOSED` |
| Cantidades negativas (Pago de Amortización) no se mostraban en tab Transacciones ni en export CSV | Condición `cantidad > 0` excluía valores negativos | Cambiar a `cantidad !== 0` en `aRow()` y `exportAuditCSV()` |

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
| Cauciones en pesos generaban "bache" en portfolio total | El monto inmovilizado salía de caja pero no figuraba como tenencia | `buildCauciones()` matchea pares por `nro_mov`; cauciones activas se muestran como filas en Portfolio y se suman a `ar_usd` en `recalcSnapshots()` |
| Picos (spikes) en serie Argentina de Evolución | `buildCashDaily` y `buildCash` usaban `fecha_liq` para registrar el impacto en caja de "Pago de Renta", mientras el ledger usa `fecha` (concertación) — generando double counting en fechas intermedias | Eliminado el tratamiento especial de `fechaCash` / `fecha_liq` en ambas funciones. Ahora toda operación (incluyendo Pago de Renta) usa `t.fecha` para impactar caja. Cash refleja "Disponible Operable" (concertación), consistente con el ledger. |

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
| Evolución AR no coincidía con Portfolio | `recalcSnapshots` usaba `buildPortfolio` → `avgByAcct` en vez del ledger | Leer `_ledger` → `snap.avgAR_usd × balAR` |
| Portfolio modo "Todo USD" no coincidía con Evolución en fechas históricas | `renderPortfolioTable` usaba `getCurrentFX()` | Reemplazado por `getFXForDate(fecha)` |
| Índice base 100 inflado | Fórmula incorrecta | Corregida: Modified Dietz `(total - flujo) / total_ayer - 1` |
| FX history solo desde 2023 | `fetchFXHistory` traía solo 1000 filas con `limit` en URL | Paginación con `Range` header — **no combinar `limit` en URL con `Range` header (error 416)** |
| Evolución se reseteaba al reabrir | `recalcSnapshots()` hacía upsert pero no DELETE — fechas huérfanas persistían en Supabase con valores viejos | DELETE de todos los snapshots antes del upsert. Requirió agregar política RLS DELETE en `portfolio_snapshots` |
| Datos de Portfolio AR vacíos al abrir | `renderPortfolioTable` corría antes de `fetchFXHistory` | `renderPortfolioTable()` al final de `fetchFXHistory` |
| Precios EEUU no cargaban (solo EWZ) | `fetchPricesHistory` combinaba `limit` en URL + `Range` header → error 416 | Paginación correcta sin `limit` en URL |
| P&L y tarjeta Ganancia vacíos al abrir | `renderPortfolioTable` corría antes de `fetchPricesHistory` | `renderPortfolioTable()` al final de `fetchPricesHistory` si hay txns |
| Date picker Portfolio en formato mm/dd/yyyy | `<input type="date">` usa locale del browser | Reemplazado por `<input type="text">` con `parseDateDMY`/`fmtDateDMY` y botones `‹`/`›` para navegar ±1 día |

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
- **Plan primero, código después** — presentar plan completo para confirmación antes de escribir código
- **Preguntar dudas antes de implementar** — no a mitad del cambio

**Reglas críticas de arquitectura — NUNCA violar:**

**► FUENTES CANÓNICAS DE CASH Y VALOR (v14) — LAS MÁS IMPORTANTES:**
- **`getCashAtDate(fecha)` es la ÚNICA fuente de cash histórico** — nunca reconstruir `buildCash(txnsUpTo, ...)` inline para consultar cash de una fecha puntual
- **`getARValueAtDate(fecha)` es la ÚNICA definición del valor AR** — nunca calcular `snap.avgAR_usd × balAR + cash` inline fuera de esta función
- **`getUSValueAtDate(fecha)` es la ÚNICA definición del valor EEUU** — nunca calcular `precio × balEEUU + cash` inline fuera de esta función
- **`buildCash()` y `buildCashDaily()` solo se llaman en `processAndRefresh()` y `saveNRAOverride()`** — solo para construir `_cashDaily`, nunca para consultar una fecha puntual
- **Todo cambio de fórmula de valuación va EXCLUSIVAMENTE en las funciones canónicas** — después de deployar, el usuario debe presionar ⟳ Recalcular para actualizar Evolución
- **`getCashAtDate` itera `_cashDaily` desde el inicio (índice 0 = más reciente)** — el array está ordenado desc; iterar desde el final daría siempre el valor más antiguo
- **`buildCash()` y `buildCashDaily()` usan `t.fecha` para TODAS las operaciones** — incluyendo "Pago de Renta". No existe tratamiento especial por `fecha_liq` ni variable `fechaCash`. El cash refleja "Disponible Operable" (fecha de concertación), consistente con el ledger.

**► LEDGER Y PORTFOLIO:**
- **`_ledger` es la fuente de verdad para balances y avgs** — Tab Portfolio y Tab Especies leen de `_ledger`, no de `AR[]`/`US[]`
- **`renderPortfolioTable()` usa `getPortfolioStateAtDate(_portfolioDate)`** — no leer de `AR[]`/`US[]` directamente
- **`costoTotal` en la tabla Portfolio usa `avgNoC_val × bal`** — sin comisiones; no restaurar `avgWithC`; la columna "P. Prom. c/comis." fue eliminada en v14
- **`buildLedger` se llama ANTES de `buildPortfolio` en `processAndRefresh`** — no cambiar este orden
- **`buildLedger` opera sobre rawTxns (pre-dedup)** — nunca pasarle txns ya deduplicadas
- **`recalcSnapshots()` usa `getARValueAtDate` y `getUSValueAtDate`** — no `buildPortfolio` → `avgByAcct`, no loops inline sobre `_ledger`
- **`recalcSnapshots()` usa `getCashAtDate(date)` por fecha** — no `buildCash(txnsUpTo, ...)`

**► ESPECIES:**
- **`renderEspecies()` lee de `_ledger[ticker]`** — no de `_instruments[ticker]`
- **`rebuildEspeciesDropdown()` usa `Object.keys(_ledger)` como fuente** — no `AR`/`US`
- **`rebuildEspeciesDropdown()` puebla 5 selectores** — no hay `esp-ticker-select` único
- **`renderEspecies()` obtiene ticker con `espGetSelectedTicker()`** — no leer de un selector fijo

**► TRANSACCIONES:**
- **`rebuildYearDropdown()` puebla `ms-panel-year` con checkboxes** — no hay `<select id="aYear">`
- **Los filtros de Transacciones son multi-valor** — `aFilter()` usa `msGetSelected()`, no `.value`
- **`isDeposito()`/`isExtraccion()` detectan por prefijo** — no por nombre exacto del banco
- **Depósitos/Extracciones fuera de `OTHER_OPS`** — tienen pseudo-valores `__depositos__`/`__extracciones__` en el dropdown

**► LEDGER INTERNO:**
- **`isEffectivelyClosed` consulta `_ledger` antes de declarar stale** — no eliminar ese check
- **`movesTitle()` determina qué fila mueve títulos** — la fila de comisión (cuenta ARS) NO mueve
- **Ventas desde AR:** consumir slot propio → otro slot AR → EEUU→AR
- **Ventas desde EEUU:** xfer implícita AR→EEUU proporcional SOLO si sufijo C (`isMEP_C`)
- **`_xferAR` y `_xferEEUU` en cada txn de venta** — los usa `renderEspecies`, no eliminar

**► FX Y PRECIOS:**
- **FX siempre desde `_fxHistory`/`_fxLatest`** — no agregar inputs manuales de FX
- **`renderPortfolioTable()` usa `getFXForDate(fecha)`** — nunca `getCurrentFX()` para cálculos de conversión
- **`recalcSnapshots()` usa `getFXForDate(date)`** — no `getCurrentFX()`
- **`fetchFXHistory` reconstruye `_ledger` + portfolio + dropdown + `renderPortfolioTable` al resolverse** — no eliminar
- **`fetchFXHistory()` llama `rebuildBlendedAvgs()` al final** — no eliminar

**► SUPABASE Y PAGINACIÓN:**
- **Paginación Supabase: usar solo `Range` header, sin `limit` en URL** — combinarlos da error 416
- **`fetchFXHistory`, `fetchSnapshots` y `fetchPricesHistory` paginan de a 1000** — loop con `Range: from-(from+999)`, break cuando `rows.length < 1000`
- **`recalcSnapshots()` hace DELETE de todos los snapshots antes del upsert** — evita fechas huérfanas; requiere política RLS DELETE en `portfolio_snapshots`

**► MISC:**
- **`fetchLatestPrices()` recalcula cash post-overrides** — no eliminar
- **`exportEspeciesXLS()` lee de `_especiesRows`/`_especiesState`** — no del DOM
- **`calcBuyCost(t, false)` descuenta comis + IVA + otros_imp. `calcBuyCost(t, true)` = `|monto|`**
- **`buildPortfolio(clean, fxHistory)` es pura** — no muta globals
- **`recalcSnapshots()` requiere `_fxHistory` cargado** — el guard lo verifica
- **`ar_ars` en snapshots siempre es 0 desde v9** — `ar_usd` ya contiene el total AR en USD
- **`buildCauciones()` solo procesa cauciones en ARS** — las cauciones en dólares se manejan por `buildUsdCaucionMovs()`
- **`buildCauciones()` matchea por `nro_mov`** — no por `nro_boleto`
- **`buildCauciones()` se llama una vez antes del loop en `recalcSnapshots()`** — no dentro del loop por fecha
- **`_arMode` solo tiene valores `'ars'` y `'usd'`** — el modo `'origin'` fue eliminado en v11
- **Las filas de caja NO están en el tbody de la tabla Portfolio** — están en las tarjetas `renderPortfolioCards()`
- **`renderPortfolioCards()` recibe `sumPL` como parámetro** — no recalcular `sumValorTenencia - sumCostoTotal` internamente
- **`renderPortfolioTable()` se llama al final de `fetchPricesHistory` y `fetchFXHistory`** — necesario para resolver race conditions de carga
