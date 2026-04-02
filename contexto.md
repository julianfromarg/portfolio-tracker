# Contexto: Portfolio Dashboard — Balanz / Julian
## Versión: 02/04/2026 v1

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
| `index.html` | El dashboard completo (en repo portfolio-tracker). ~3450 líneas. |
| `MovimientosHistoricos_Completo.xls` | Export de Balanz: Reportes → Movimientos Históricos |
| `CONTEXTO_PORTFOLIO_DASHBOARD_v02_04_v1.md` | Este archivo |

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
| P. Prom. s/comis. | Precio promedio sin comisiones, en la moneda del modo activo |
| P. Prom. c/comis. | Precio promedio con comisiones, en la moneda del modo activo |
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
- **Todo USD:** todo convertido a dólares al FX de `_fxLatest`
- FX siempre tomado de `_fxHistory` / `_fxLatest` desde Supabase — no hay input manual
- Posiciones AR valuadas al costo (P. Prom. s/comis.) hasta tener precios BYMA

### Total EEUU:
- Suma posiciones con fallback al costo + caja USD
- Muestra `*` en el footer cuando alguna posición usa precio estimado

### Columnas P. Prom. en modo Argentina:
Las columnas de precio promedio varían según el modo activo (ARS/USD), usando `avgByAcct` por slot:
- **Moneda origen:** muestra cada slot en su moneda nativa (`$ X` / `U$S Y`) separados por `/` si hay ambos
- **Todo ARS:** convierte slot USD multiplicando por FX, promedio ponderado por balance
- **Todo USD:** convierte slot ARS dividiendo por FX, promedio ponderado por balance

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

La retención real puede diferir del 30% calculado automáticamente (ej: IOL a veces liquida la NRA acumulada en un boleto separado con monto anómalo). El usuario puede ingresar la retención real directamente.

```javascript
function calcNRA(t) {
  if(!isNRAApplicable(t)) return 0;
  const key = String(t.nro_mov);
  if(_nraOverrides.has(key)) {
    return _nraOverrideAmounts.get(key) || 0; // monto con signo libre del usuario
  }
  const gross = Math.abs(t.monto||0) + Math.abs(t.comision||0) + Math.abs(t.iva||0);
  return -(gross * NRA_RATE);
}
```

**Dos globals para overrides:**
```javascript
let _nraOverrides = new Set();       // set de nro_mov con override activo
let _nraOverrideAmounts = new Map(); // nro_mov → monto numérico (con signo)
```

**Dónde se aplica `calcNRA`:**
- `buildCash()` — descuenta NRA del balance de `EEUU (USD)`
- `buildCashDaily()` — idem para saldos diarios
- `buildAuditRows()` — agrega campo `nra` a cada fila

**Tabla `nra_overrides` en Supabase:**
```
nro_mov        TEXT
excluir        BOOL          (legacy, se mantiene por compatibilidad)
monto_override NUMERIC       (NULL = usar 30% auto; cualquier valor = usar ese monto)
session_id     TEXT          (filtra overrides por sesión)
PRIMARY KEY (nro_mov, session_id)
```

**UX — Tab 🏛 Ret. NRA:**
- Pill de 3 estados por fila:
  - `30% auto` (naranja) → sin override
  - `Sin ret.` (gris) → override = 0
  - `+/-U$S X.XX` (rojo) → override con monto custom
- Click en pill → popover con input numérico (signo libre), monto auto como referencia, botones Guardar / Limpiar (auto) / Cancelar
- Enter guarda, Escape cierra
- "Limpiar (auto)" → PATCH `monto_override = NULL` (no DELETE, por permisos RLS)
- Botón `↓ CSV` → exporta todas las filas con columnas: fecha, ticker, montos, tipo retención, nro_mov

**Fetch de overrides (al cargar precios):**
```javascript
// Filtra por session_id de la sesión activa
GET /nra_overrides?select=nro_mov,monto_override&session_id=eq.{sessionId}
```

**CRÍTICO — race condition al inicio resuelta:**
`fetchLatestPrices()` recalcula `buildCash()` y `buildCashDaily()` después de cargar los overrides de Supabase. Sin esto, el cash al abrir la página ignora los overrides.

---

## Estructura del .xls de Balanz

**CRÍTICO: el archivo .xls de Balanz es en realidad HTML disfrazado** (no un binario XLS real).
El dashboard lo detecta por los primeros bytes y lo parsea con `DOMParser` nativo del browser, **NO con SheetJS**.

```
Row 1: Título "Operaciones Historicas del periodo..."
Row 2: Headers — columnas por índice:
  [0]  Nro. de Mov.   → nro_mov  (puede ser 0 para eventos sin boleto)
  [1]  Nro. de Boleto → nro_boleto
  [2]  Tipo Mov.      → "Compra(STRC)", "Venta(AL30D)", "Remuneración de Saldos Líquidos", etc.
  [3]  Concert.       → fecha de concertación — string "dd/mm/yy" en el archivo del broker
  [4]  Liquid.        → fecha de liquidación
  [5]  Est            → estado: "Terminada" | "Cancelada"
  [6]  Cant. titulos  → cantidad
  [7]  Precio         → precio — EN LA MONEDA DE LA CUENTA, con coma decimal AR
  [8]  Comis.         → comisión
  [9]  Iva Com.       → IVA
  [10] Otros Imp.     → otros impuestos
  [11] Monto          → monto neto (ya incluye comisiones, con signo: negativo=egreso)
  [12] Observaciones
  [13] Tipo Cuenta    → "Inversion Estados Unidos Dolares" / "Inversion Argentina Pesos" / "Inversion Argentina Dolares"
```

**Mapeo de cuentas:**
```
"Inversion Estados Unidos Dolares" → "EEUU (USD)"
"Inversion Argentina Pesos"        → "Argentina (ARS)"
"Inversion Argentina Dolares"      → "Argentina (USD)"
```

**CRÍTICO — campo Precio:**
- El campo `precio` está en la moneda de la cuenta
- Usa coma decimal formato AR: `11,7699` = $11.7699 USD
- **Para bonos:** el precio es por cada 100 de valor nominal → costo = `precio × cantidad / 100`
- **Para acciones/CEDEARs:** el precio es por unidad → costo = `precio × cantidad`

**CRÍTICO — campo Monto:**
- Siempre en la moneda de la cuenta
- Ya incluye comisiones (neto)
- Negativo = egreso de caja

---

## Cálculo de precio promedio — `buildInstrument()` + `calcBuyCost()`

### Método: VWAP promedio móvil (moving average net of sells)

El precio promedio se calcula con promedio ponderado móvil. Al vender, el costo acumulado se reduce proporcionalmente — las unidades vendidas salen al precio promedio vigente, no al precio de venta. Si el balance llega a 0, el costo acumulado se resetea a 0 (pizarra en blanco para futuras compras).

```javascript
// En SELL_OPS dentro del loop de buildInstrument:
const sellQty = Math.min(qty, ac.balance);
const ratio = Math.max(0, (ac.balance - sellQty) / ac.balance);
ac.costNoComis   *= ratio;
ac.costWithComis *= ratio;
ac.balance -= qty;
if(ac.balance <= 0) { ac.balance = 0; ac.costNoComis = 0; ac.costWithComis = 0; }
```

### Segregación por cuenta (`acctState`) — 3 slots

**CRÍTICO:** `buildInstrument` trackea balance y costo **por separado para 3 slots**, porque el mismo ticker canonicalizado puede existir en distintas monedas:

```javascript
// 'EEUU'   → cuenta EEUU (USD)       -- avg en USD
// 'AR_ARS' → cuenta Argentina (ARS)  -- avg en ARS
// 'AR_USD' → cuenta Argentina (USD)  -- avg en USD
const acctState = {};
const getAcct = cuenta => {
  const key = cuenta.includes('EEUU') ? 'EEUU' : cuenta.includes('ARS') ? 'AR_ARS' : 'AR_USD';
  if(!acctState[key]) acctState[key] = {balance:0, costNoComis:0, costWithComis:0};
  return acctState[key];
};
```

`buildInstrument` retorna `avgByAcct` con hasta 3 entradas, cada una con `{avg, avgWithCom, balance}`. `makeCard` preserva `avgByAcct` en el objeto final. El footer y las columnas de precio promedio usan `avgByAcct` directamente — **nunca el `avg` global mezclado**.

### `calcBuyCost(t, withComissions)`

Dos versiones del precio promedio:

**Sin comisiones** (= precio del broker, comparable con precio de mercado):
- Acciones/CEDEARs: `(|monto| - comision - iva) / cantidad`
- Bonos: `precio × cantidad / 100` (el precio ya es sin comisiones en los bonos)

**Con comisiones** (= costo real de adquisición):
- Acciones/CEDEARs: `|monto| / cantidad`
- Bonos: igual que sin comisiones

**Detección de bonos — `isBond(ticker)`:**
```javascript
const BOND_TICKERS = new Set([
  'AL30','AY24','GD30','CO26','AO20','BDC20','AS13','TO26',
  'AL30D','AY24D','GD30D','AO20D','PARYD',
  'AL30C','AY24C',
]);
```

---

## Pipeline de procesamiento

### 1. Parse — `handleFileImport()` → `parseBalanzRowsFromHTML()` o `parseBalanzRows()`

**CRÍTICO — detección del tipo de archivo:**
```javascript
const isHtmlXls = /<html|<table|<tr|<!DOCTYPE/i.test(textBuf);
```
- Si es HTML → `parseBalanzRowsFromHTML()` con `DOMParser`
- Si es XLSX → `parseBalanzRows()` con SheetJS

**`numArg()` — tres casos:**
```javascript
// 1. Formato AR con decimal: "1.234,56" → 1234.56
if(/^-?[\d.]+,\d+$/.test(str)) return parseFloat(str.replace(/\./g,'').replace(',','.'));
// 2. Entero con separador de miles: "677.225" → 677225
if(/^-?\d{1,3}(\.\d{3})+$/.test(str)) return parseFloat(str.replace(/\./g,''));
// 3. Número JS directo o decimal estándar
```

### 2. Merge — `mergeTransactions()`

Clave primaria via `primaryKey()`:
- Si `nro_mov != '0'`: `mov|fecha|cuenta|nro_mov`
- Fallback: `fb|fecha|cuenta|ticker|operacion|cantidad`

### 3. Deduplicación MEP — `deduplicateMEP()`

**CRÍTICO:** solo deduplicar si hay filas del mismo grupo en cuentas DISTINTAS (`accounts.size > 1`).
**CRÍTICO:** usar siempre `rawClean` (pre-dedup) para calcular cash.

### 4. Canonicalización de tickers — `canonicalizeTickers()`

```javascript
const INSTRUMENT_GROUPS = {
  AL30:['AL30','AL30D','AL30C'], AY24:['AY24','AY24D','AY24C'],
  GD30:['GD30','GD30D'], AO20:['AO20','AO20D'],
  MELI:['MELI','MELID'], VIST:['VIST','VISTD'],
  SPY:['SPY','SPYD'], PARY:['PARY','PARYD'],
};
```

### 5. Cálculo de posiciones — `buildPortfolio()`

**BUY_OPS:** `Compra`, `Suscripción Primaria`, `Suscripción FCI`, `Transferencia de Titulos IN -`
**SELL_OPS:** `Venta`, `Rescate FCI`, `Pago de Amortización` (cantidad < 0)

**Detección de posiciones cerradas — `isEffectivelyClosed()`:**
1. Stale: `Math.max(...years) <= 2020`
2. Fully amortized: buys + Pago de Amortización ≥ 80% del costo, sin sells

**CRÍTICO:** `buildPortfolio()` es pura — no muta globals. Segura para llamar con subsets.

### 6. Exclusión de opciones — `isOption()`

```javascript
const OPTION_DECIMAL_RE = /^[A-Za-z]{2,6}\d+\.\d+[A-Za-z]{1,2}$/;
const OPTION_PREFIX_RE  = /^(TPPC|TPPV|ALUC|ALUP|GFGV|TECC|PBRC|EDNC|EDNA|EDND)/;
```

---

## Cálculo de cash — `buildCash()` y `buildAuditRows()`

**CASH_SKIP_OPS:**
```javascript
new Set([
  'Transferencia de Titulos IN -',
  'Depósito por transferencia interna',
  'Extracción por Transferencia interna',
])
```

**Cauciones en Dólares:** `buildUsdCaucionMovs()` identifica `nro_mov` de cauciones en dólares y excluye sus filas del cash.

**NRA en cash:**
- `buildCash()` y `buildCashDaily()` llaman `calcNRA(t)` para cada dividendo EEUU
- El resultado (negativo) se suma al balance de `EEUU (USD)`

**CRÍTICO — `buildAuditRows(clean, rawClean, usdCaucionMovs)`:**
- `clean`: deduped + canonicalized → para mostrar en tabla
- `rawClean`: pre-dedup + canonicalized → para calcular saldo cash correcto

---

## Precios y datos externos — Supabase + GitHub Actions

### Supabase
- **Proyecto:** portfolio-tracker (`ghxisfkgwfqqxndfpmgp.supabase.co`)
- **Anon key:** `sb_publishable_s5hodNiLD-8_OwX8NWHfRw_S1uylN2Y`
- **Tablas:**
  - `instruments` — catálogo de instrumentos
  - `prices` — precios EOD históricos
  - `nra_overrides` — overrides de retención NRA (`nro_mov`, `excluir`, `monto_override`, `session_id`) — PK: `(nro_mov, session_id)`
  - `fx_rates` — tipo de cambio USD/ARS diario (fuente de verdad — no hay input manual)
  - `portfolio_snapshots` — valor diario del portfolio

### Tickers US activos:
`NU, STRC, VGSH, SHV, EWZ, IREN, MSTR, MELI, IVV, SPY, VIST`

### GitHub Actions
- **Repo:** github.com/julianfromarg/portfolio-tracker-prices (privado)
- **Cron:** lunes a viernes 21:00 UTC (18:00 ART)
- **Script:** `update_prices.py` — fetcha Yahoo Finance, upsert en Supabase

### Fetch en el dashboard (al cargar + botón 🔄):
- `fetchLatestPrices()` — último EOD + NRA overrides de la sesión activa + **recálculo de cash post-overrides**
- `fetchPricesHistory()` — histórico completo
- `fetchFXHistory()` — histórico FX + setea `_fxLatest`. **Orden `date.desc`, limit 10000, Range 0-9999** — Supabase cappea en 1000 filas por defecto; con `desc` trae las 1000 más recientes.
- `fetchSnapshots()` — snapshots guardados

**CRÍTICO — orden de carga:** `fetchFXHistory()` debe completar antes de `recalcSnapshots()`.

---

## Tab 📈 Evolución

### `recalcSnapshots()`:
1. Verifica `_historicalTxns` y `_fxHistory` cargados
2. Por cada fecha en `_cashDaily`: reconstruye posiciones usando `avgByAcct` por slot, valoriza, obtiene FX, guarda snapshot
3. AR: `ar_ars += avgByAcct.AR_ARS.avg × balance`, `ar_usd += avgByAcct.AR_USD.avg × balance`
4. Upsert en `portfolio_snapshots` en batches de 200
5. Barra de progreso durante el cálculo

### `renderEvolChart()`:
- 3 series independientes (no apiladas, sin fill):
  - 🇦🇷 Argentina: `arUSD = ar_ars / fx_rate + ar_usd`, color `#3d7fff`, borderWidth 2.5
  - 🇺🇸 EEUU: `usUSD = us_usd`, color `#ef4444`, borderWidth 2.5
  - ∑ Total: `totalUSD = arUSD + usUSD`, color `#a78bfa`, borderWidth 3
- Chart.js con zoom plugin
- Selector de rango: 1M / 3M / 6M / 1Y / MAX
- Toggle de series individual — cada serie es independiente, Total no es redundante

---

## Tab 🔍 Especies

Tab de auditoría de precio promedio por instrumento. Permite ver la evolución fila por fila del VWAP.

### Dropdown con agrupación:
El selector de ticker usa `<optgroup>` para separar posiciones abiertas de cerradas:
- **● Posiciones abiertas (N)** — tickers con `net > 0`, muestra el balance entre paréntesis (suma AR + EEUU)
- **○ Históricas / cerradas (N)** — tickers con historial pero sin balance

### Estructura de la tabla:
Dos grupos de columnas generados dinámicamente según qué cuentas tenga la especie. Layout fijo (sin toggle):

**Grupo 🇦🇷 Argentina** (colspan 9, si tiene ops en cuentas Argentina):
- Compras · Ventas · Xfer. · Amort. · Bal AR · Precio ARS · **Avg ARS AR** · **Valoriz. ARS AR** · **Valoriz. USD AR**

**Grupo 🇺🇸 EEUU** (colspan 8, si tiene ops en cuenta EEUU):
- Compras · Ventas · Xfer. · Amort. · Bal EEUU · Precio USD · **Avg USD EEUU** · **Valoriz. USD EEUU**

**FX** — columna final compartida

Si la especie solo existe en un lado, solo aparece ese grupo.

### Lógica de valorización (fija, sin toggle):
- **Valoriz. ARS AR** = posición AR_ARS × avgARS + posición AR_USD × avgARUSD × FX (todo en pesos)
- **Valoriz. USD AR** = posición AR_ARS × avgARS / FX + posición AR_USD × avgARUSD (todo en dólares)
- **Valoriz. USD EEUU** = posición EEUU × avgEEUU (en dólares)
- **Avg ARS AR** = avgARS del slot AR_ARS (nativo en pesos)
- **Avg USD EEUU** = avgEEUU del slot EEUU (nativo en dólares)

### Lógica VWAP en Especies:
Mismo algoritmo que `buildInstrument()` pero recalculado fila por fila en `renderEspecies()` para mostrar la evolución. Usa los mismos slots `AR_ARS`, `AR_USD`, `EEUU`.

### Export XLS (`exportEspeciesXLS()`):
- Botón `↓ XLS` aparece en la toolbar solo cuando hay ticker seleccionado
- **Lee de `_especiesRows` y `_especiesState`** (globals guardados al final de `renderEspecies`) — NO scrapea el DOM
- Formato del archivo:
  - Fila 1: headers de grupo con merge de celdas (🇦🇷 Argentina / 🇺🇸 EEUU)
  - Fila 2: headers de columna
  - Filas de datos: fechas como `dd/mm/yyyy` (string), números como valores numéricos con formato `#,##0` o `#,##0.00`
  - Fila de footer con posición final
  - Freeze de primeras 2 filas
  - Anchos de columna ajustados por tipo
  - Celdas vacías (compras/ventas que no aplican) como `null`, no como `0`
- Nombre del archivo: `Especies_TICKER_YYYY-MM-DD.xlsx`

### Globals de estado para export:
```javascript
let _especiesRows  = [];  // rows del último renderEspecies (para XLS export)
let _especiesState = {};  // {ticker, hasAR, hasEEUU, opsLen}
```

### Funciones clave:
- `rebuildEspeciesDropdown()` — repuebla el selector con optgroups
- `renderEspecies()` — renderiza la tabla y guarda `_especiesRows` / `_especiesState`
- `exportEspeciesXLS()` — genera y descarga el XLSX desde `_especiesRows`

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
- Contador de registros
- Se popula automáticamente cuando `fetchFXHistory()` termina y al hacer click en el tab

### Funciones clave:
- `rebuildFXYearDropdown()` — popula el dropdown de años desde `_fxHistory`
- `renderFXTab()` — renderiza la tabla con filtros y variaciones
- `fxTabSort(col)` — toggle de orden
- `fxTabClear()` — limpia filtros

### CRÍTICO — límite de filas Supabase:
Supabase cappea en 1000 filas por request independientemente del `limit` enviado. `fetchFXHistory` usa `order=date.desc` para traer las 1000 entradas **más recientes** (no las más antiguas). Las variaciones se calculan siempre contra el histórico completo cargado (`allDates`), no contra el subconjunto filtrado.

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

### Posiciones y precios

| Bug | Causa | Fix |
|---|---|---|
| Precio promedio incorrecto EEUU | Formato AR con SheetJS | `(|monto| - comis - iva) / cantidad` |
| Precio promedio bonos incorrecto | Bonos cotizan por lámina de 100 | `calcBuyCost`: `precio × cantidad / 100` |
| Tickers viejos con posición abierta | Historial incompleto | `isEffectivelyClosed()`: stale `<= 2020` |
| BDC20 aparece como abierta | Check stale era `< 2020` | Cambiar a `<= 2020` |
| Bonos amortizados con posición abierta | `Pago de Amortización` sin cantidad | Criterio amortización ≥ 80% |
| Opciones con posición abierta | Vencen sin cierre registrado | `isOption()`: excluir completamente |
| ADRDOLA no aparece | Cantidad `"677.225"` parseada mal | Fix en `numArg()` para enteros con miles |
| Precio promedio incorrecto tras ventas parciales | `avg = totalCostNoComis / bought` ignoraba ventas | VWAP móvil: al vender, `costNoComis *= (balance - sellQty) / balance`. Si balance = 0, reset a 0. |
| Precio promedio incorrecto para tickers split AR/EEUU (ej: MELI) | `buildInstrument` mezclaba montos ARS y USD en el mismo acumulador `'AR'` | 3 slots: `'EEUU'`, `'AR_ARS'`, `'AR_USD'`. `avgByAcct` incluye `balance` por slot. `makeCard` preserva `avgByAcct`. Footer y columnas usan slots directamente. |
| Total Argentina (USD) mostraba ~U$S 32M | Footer filtraba por `accounts.includes('USD')` capturando CEDEARs con avg en pesos | `calcArSlots()` itera `avgByAcct.AR_ARS` y `avgByAcct.AR_USD` directamente — sin filtro por `accounts` |

### Cash

| Bug | Causa | Fix |
|---|---|---|
| Transferencias internas duplican cash | Aparecen en dos cuentas | Agregar a CASH_SKIP_OPS |
| Caución en Dólares distorsiona saldos | Broker registra -USD y +ARS | `buildUsdCaucionMovs()` + `isCashSkip()` |
| Cash Argentina USD negativo | SheetJS trunca `"10.000,00"` | Usar `DOMParser` |
| Cash EEUU incorrecto (factor ~10x) | NRA no descontada | `calcNRA()` en `buildCash()` y `buildCashDaily()` |
| Override NRA no impacta cash | `cFilter()` no se llamaba tras guardar override | Llamar `cFilter()` en lugar de `cSortApply()` en `saveNRAOverride()` |
| Cash incorrecto al abrir (overrides ignorados) | Race condition: `processAndRefresh` corre antes de que `fetchLatestPrices` cargue los overrides de Supabase | `fetchLatestPrices()` recalcula `buildCash()` + `buildCashDaily()` + `cFilter()` después de cargar los overrides |

### NRA overrides

| Bug | Causa | Fix |
|---|---|---|
| "Limpiar" no persistía (override volvía en F5) | DELETE sin permisos RLS con anon key | Reemplazar DELETE por PATCH `monto_override = NULL` |
| Override no impactaba cash tab | `saveNRAOverride` llamaba `cSortApply` (no refrescaba `cData`) | Cambiar a `cFilter()` |

### Snapshots / Evolución

| Bug | Causa | Fix |
|---|---|---|
| `recalcSnapshots` llama función inexistente | Referencia a `buildPortfolioLocal` | Usar `buildPortfolio()` directamente |
| Atribución incorrecta posiciones AR | `d.accounts.some(a => a.includes('ARS'))` mezclaba | Usar `avgByAcct.AR_ARS` y `avgByAcct.AR_USD` directamente |
| Snapshots con `fx_rate = 1` | `fetchFXHistory` async no terminaba | Guard en `recalcSnapshots()` |

### UI / Tab

| Bug | Causa | Fix |
|---|---|---|
| Tab activo no persiste al refrescar | `act-portfolio` hardcodeado en HTML, sin persistencia | `switchTab` guarda en `localStorage('active_tab')`, `init` restaura |
| `switchTab` explota si panel no existe | `getElementById(...).classList` sobre null | Guard: `const panel = getElementById(...); if(!panel) return;` |
| Dropdown Especies vacío al abrir tab | `rebuildEspeciesDropdown` solo se llamaba en `processAndRefresh` | También se llama en `switchTab` cuando key === `'especies'` |

### FX / Supabase

| Bug | Causa | Fix |
|---|---|---|
| Tab FX solo mostraba hasta 2023 | `fetchFXHistory` con `order=date.asc&limit=5000` — Supabase cappea en 1000 filas, traía las más antiguas | Cambiar a `order=date.desc` con `limit=10000` y `Range: 0-9999` — trae las 1000 más recientes |

### Export Especies XLS

| Bug | Causa | Fix |
|---|---|---|
| Fechas exportadas como número entero (solo el día) | `exportEspeciesXLS` scrapeaba el DOM y `parseFloat` convertía `"28/03/2026"` → `28` | Reescribir para leer de `_especiesRows` directamente; fechas como string `dd/mm/yyyy` |
| Montos exportados como texto con prefijo | `"U$S 1.789,74"` no parseaba a número | Leer valores numéricos crudos desde `_especiesRows`, aplicar formato Excel nativo |

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

---

## Cómo pedir cambios en un chat nuevo

Pegá este documento + el HTML al inicio del chat. Ejemplos:

> "Tengo este contexto [pegar doc]. Quiero conectar precios BYMA para la cuenta Argentina."

> "Tengo este contexto [pegar doc]. El saldo cash al 31/12/2020 no cuadra."

> "Tengo este contexto [pegar doc]. El gráfico de evolución no muestra datos."

**Reglas para cambios:**
- Siempre editar el archivo existente (no reescribir desde cero)
- Antes de cualquier fix de cash, validar con Python usando el .xls directamente
- Antes de cambiar `deduplicateMEP`, verificar con casos concretos
- `buildCash()` y saldo en `buildAuditRows()` deben usar siempre `rawClean` (pre-dedup)
- Para agregar tickers nuevos: INSERT en `instruments`, correr backfill desde GitHub Actions
- El archivo del broker es HTML disfrazado — **nunca procesar con SheetJS directamente**
- `buildPortfolio()` es pura — segura para llamar con subsets
- `recalcSnapshots()` requiere `_fxHistory` cargado (el guard lo verifica)
- El dashboard NO funciona en el sandbox de Claude (CSP bloquea Supabase). Siempre testear desde GitHub Pages
- Los NRA overrides se filtran por `session_id` — cada sesión tiene sus propios overrides
- El PATCH (no DELETE) es intencional para limpiar overrides — RLS no permite DELETE con anon key
- **`buildInstrument` trackea `acctState` con 3 slots: `EEUU`, `AR_ARS`, `AR_USD` — nunca mezclar montos de distintas monedas**
- **`avgByAcct` incluye `balance` por slot — usarlo siempre para footer y columnas de precio en AR**
- **El avg en `makeCard` viene de `d.avgByAcct[acctKey]`, no de `d.avg` global — no cambiar esto**
- **`fetchLatestPrices()` recalcula cash post-overrides — este recálculo es intencional, no eliminarlo**
- **FX siempre desde `_fxHistory`/`_fxLatest` — no agregar inputs manuales de FX en el Portfolio tab**
- **`exportEspeciesXLS()` lee de `_especiesRows`/`_especiesState`, no del DOM — no cambiar esto**
- **Tab Especies: no hay toggle ARS/USD — el layout es fijo. No agregar `_espValMode` de vuelta**
- **`fetchFXHistory` usa `order=date.desc` — intencional para traer los más recientes dentro del cap de Supabase**
