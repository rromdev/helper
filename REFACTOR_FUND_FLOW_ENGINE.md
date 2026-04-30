# REFACTOR: `middle_man_detector.py` → `fund_flow_engine`

**Tipo:** Refactor mayor con coexistencia controlada
**Estado:** Spec aprobado, pendiente de implementación
**Owner:** @inflex
**Estimación:** 8-15 horas de trabajo dirigido por Claude Code, repartido en fases

---

## 1. CONTEXTO DEL PROYECTO

### Stack INFLEX (producción)

- **Servidor:** Hetzner CAX11 ARM64, 4GB RAM, Ubuntu, IP 89.167.101.71
- **Runtimes:** Python 3.12.3, Node v24.14.0
- **API:** Flask en `127.0.0.1:5001` (~3738 LOC)
- **UI:** React 19 + Vite en `:5174`
- **DB principal:** SQLite en `/inflex/inflex_final_product/data/knowledge.db`
- **Sin Docker, sin nginx, sin MongoDB, sin Redis garantizado**

### Módulos Python relevantes (NO duplicar lógica existente)

- `rpc_pool.py` (519 LOC) — pool rotativo SUI con health checks, circuit breakers, fallback. **USAR SIEMPRE** para queries RPC, nunca llamar endpoints directamente.
- `sui_graphql.py` (771 LOC) — cliente GraphQL para queries complejas del indexer.
- `sui_client.py` (211 LOC) — cliente async básico SUI RPC.
- `wallet_investigator.py` (944 LOC) — motor de investigación. **Importa de `middle_man_detector`** (línea ~233 usa `MiddleManResult`). No tocar en este refactor.
- `wallet_cluster.py` (1037 LOC) — clustering por heurísticas. No tocar en este refactor.
- `middle_man_detector.py` (908 LOC) — **este es el módulo que se refactoriza.**

### Endpoints API que dependen de este módulo (mantener compatibilidad)

```
GET /api/wallet/middle-man/:addr
GET /api/wallet/cluster/:addr
GET /api/wallet/cluster/extended/:addr
GET /api/wallet/cez/:addr
GET /api/wallet/linked/:addr
```

Estos endpoints consumen `MiddleManResult` y los outputs actuales de `detect()` y `find_middle_man()`. **El refactor no debe romperlos.**

---

## 2. PROBLEMA QUE RESOLVEMOS

El `middle_man_detector.py` actual tiene cinco problemas críticos:

1. **Scoring binario hardcoded.** Usa thresholds arbitrarios (`+30`, `+20`, `+25`, threshold 40). Un MM real es un fenómeno probabilístico, no un sí/no.
2. **No usa temporalidad.** El indicador más fuerte de MM es el *dwell time* (tiempo entre recibir y reenviar). El código actual lo ignora completamente.
3. **Falsos positivos altos.** "Recibe de 5+ wallets +30" hace que cualquier wallet popular se marque como MM.
4. **Solo detecta 1 nivel.** El patrón rugger real es multi-hop: `rug → MM1 → CEX → withdrawal a wallet nueva → opera otra vez`. El código actual no captura cadenas.
5. **Lento.** Sin cache de features ni de edges, cada análisis re-quetea RPC desde cero.

### Lo que debe detectar el sistema nuevo

- **Patrones de ruggers:** rug → MM → CEX → wallet nueva (reactivación)
- **Insiders/KOLs operando con múltiples wallets** del mismo owner
- **Mapeo visual de redes de wallets** del mismo owner con confidence
- **Hacks tipo Aftermath:** drain → fragmentación a múltiples MMs → CEX
- **CEX bridging:** detectar cuando un owner cruza fondos a través de CEX para romper trazabilidad on-chain directa

---

## 3. ESTRATEGIA DE COEXISTENCIA (CRÍTICO)

`wallet_investigator.py` y `wallet_cluster.py` ya importan de `middle_man_detector`. Reemplazar todo de golpe = 2889 LOC tocados simultáneamente sin tests previos. Inviable.

### Plan en tres etapas

**Etapa A — Construcción en paralelo**
- Se crea `fund_flow_engine/` desde cero.
- `middle_man_detector.py` sigue siendo la fuente de verdad activa.
- Endpoints siguen llamando al código viejo.
- Se generan golden tests del comportamiento actual.

**Etapa B — Activación con feature flag**
- `middle_man_detector.py` se convierte en **paquete** (`middle_man_detector/`).
- El código actual se mueve LITERAL a `middle_man_detector/_legacy_impl.py` con `git mv` para preservar historia.
- `middle_man_detector/__init__.py` se vuelve un router que decide:

```python
# middle_man_detector/__init__.py
from config.feature_flags import USE_NEW_FUND_FLOW_ENGINE
from . import _legacy_impl

if USE_NEW_FUND_FLOW_ENGINE:
    from fund_flow_engine.api import detect_compat as detect
    from fund_flow_engine.api import find_middle_man_compat as find_middle_man
    from fund_flow_engine.types import MiddleManResult
else:
    from ._legacy_impl import detect, find_middle_man, MiddleManResult
```

- `wallet_investigator.py` y `wallet_cluster.py` no se tocan — siguen importando `MiddleManResult` de `middle_man_detector`, que sigue resolviéndose correctamente.

**Etapa C — Eliminación del legacy (deadline duro: 3 semanas tras Etapa B)**
- Si el flag `USE_NEW_FUND_FLOW_ENGINE=true` lleva 2 semanas en producción sin regresiones, se migran `wallet_investigator.py` y `wallet_cluster.py` para importar directamente de `fund_flow_engine`.
- Se elimina `_legacy_impl.py`.
- Se elimina el feature flag.

### Compatibilidad de tipos

`MiddleManResult` debe ser un **superset** del actual (mismos campos + nuevos opcionales con defaults). Así cuando `wallet_investigator.py:233` se migre, sigue funcionando solo cambiando el import path.

---

## 4. ESTRUCTURA DE CARPETAS

```
inflex_final_product/
├── middle_man_detector/                # antes era middle_man_detector.py (archivo)
│   ├── __init__.py                     # facade público con routing por feature flag
│   ├── _legacy_impl.py                 # código actual movido aquí literal (git mv)
│   └── _routing.py                     # decide legacy vs new según feature flag
│
├── fund_flow_engine/
│   ├── __init__.py
│   ├── api.py                          # detect_compat, find_middle_man_compat, analyze_wallet, trace_flow, find_owner_cluster, detect_cex_bridges
│   ├── types.py                        # WalletFeatures, WalletAnalysis, FlowGraph, OwnerCluster, BridgeMatch, MiddleManResult
│   ├── features/
│   │   ├── __init__.py
│   │   ├── extractor.py                # extract(address) -> WalletFeatures
│   │   ├── temporal.py                 # dwell_time, activity_burst, lifetime
│   │   ├── topology.py                 # fan_in/out, sender overlap, common funding
│   │   └── behavioral.py               # token diversity, liquid_ratio, cex_proximity
│   ├── classifiers/
│   │   ├── __init__.py
│   │   ├── bayesian.py                 # framework de likelihood ratios reutilizable
│   │   ├── middle_man.py               # classify_middle_man(features) -> (prob, evidence)
│   │   ├── cex.py                      # classify_cex(features) -> (prob, evidence)
│   │   └── passthrough.py              # classify_passthrough_edge(edge_features)
│   ├── patterns/
│   │   ├── __init__.py
│   │   ├── cex_bridge.py               # detect_cex_bridge(deposit_tx, cex_addr)
│   │   ├── rug_pattern.py              # state machine para rug → MM → CEX
│   │   └── hack_pattern.py             # fragmentación tipo Aftermath
│   ├── clustering/
│   │   ├── __init__.py
│   │   ├── owner_cluster.py            # agrupación por features compartidas
│   │   └── propagation.py              # BFS multi-nivel con propagación de probabilidad
│   ├── storage/
│   │   ├── __init__.py
│   │   ├── feature_cache.py            # SQLite con WAL, TTL adaptativo
│   │   ├── edge_index.py               # SQLite con índices from/to/timestamp
│   │   └── migrations.py               # crear tablas si no existen, idempotente
│   └── README.md                       # arquitectura, decisiones, cómo extender
│
├── config/
│   └── feature_flags.py                # nuevo si no existe; USE_NEW_FUND_FLOW_ENGINE
│
├── tests/
│   ├── fixtures/
│   │   ├── golden_wallets.json         # 10 wallets seleccionadas con metadata
│   │   └── golden_outputs.json         # output del legacy sobre esas 10 (baseline)
│   ├── test_fund_flow_golden.py        # corre el nuevo y compara contra golden
│   ├── regression_check.py             # script standalone con reporte legible
│   └── test_features.py                # unit tests de extractores
│
└── scripts/
    ├── generate_golden_outputs.py      # corre legacy sobre las 10, serializa
    └── seed_cex_wallets.py             # añade hot wallets de CEX a KB
```

### Notas sobre la estructura

- `middle_man_detector.py` (archivo) se convierte en `middle_man_detector/` (paquete). Python resuelve `from middle_man_detector import X` igual.
- Tablas SQLite del fund_flow_engine van en `data/fund_flow_cache.db` separadas de `knowledge.db`. **No contaminar la KB con datos volátiles.**
- Usar `git mv middle_man_detector.py middle_man_detector/_legacy_impl.py` para preservar historia.

---

## 5. FASES DE IMPLEMENTACIÓN

**ORDEN ESTRICTO. No saltar fases. Validar cada una antes de la siguiente.**

### Fase 0 — Golden Tests (BLOQUEANTE, hacer primero)

Sin baseline no podemos garantizar que el refactor no degrada precisión.

**Tareas:**

1. Crear `tests/fixtures/golden_wallets.json` con 8-12 wallets seleccionadas de la KB. Criterio:
   - 3-4 **positivos fuertes** (Ruggers, MMs conocidos, KOLs con multi-wallet conocido)
   - 2-3 **negativos** (Trusted, wallets personales)
   - 2-3 **ambiguos** (KOLs activos que podrían dar falso positivo)
   - Todas con actividad on-chain suficiente (>50 txs ideal)

2. SELECT sugerido (ajustar según schema real de la KB):

```sql
-- KOL con más actividad
SELECT address, tag, metadata FROM wallets
WHERE tag = 'KOL'
ORDER BY (SELECT COUNT(*) FROM edges WHERE from_address = wallets.address OR to_address = wallets.address) DESC
LIMIT 3;

-- Ruggers conocidos
SELECT address, tag, metadata FROM wallets
WHERE tag = 'Rugger'
ORDER BY (SELECT COUNT(*) FROM edges WHERE from_address = wallets.address OR to_address = wallets.address) DESC
LIMIT 3;

-- CEX (si hay; si no, saltar y añadir en Fase 8)
SELECT address, tag, metadata FROM wallets WHERE tag = 'CEX' LIMIT 2;

-- Trusted (control negativo)
SELECT address, tag, metadata FROM wallets WHERE tag = 'Trusted' LIMIT 2;
```

**SI LA KB NO TIENE TABLA `edges` INDEXADA:** ordenar por `last_seen` o equivalente y dejarlo documentado. La Fase 6 (storage) se vuelve crítica desde el inicio en ese caso.

3. Shape de `golden_wallets.json`:

```json
{
  "version": 1,
  "generated_at": "ISO8601",
  "wallets": [
    {
      "address": "0x...",
      "tag": "KOL",
      "expected_classification": "not_mm",
      "notes": "KOL muy activo, podría dar falso positivo si el detector es ingenuo",
      "tx_count_at_capture": 247
    }
  ]
}
```

`expected_classification` es **juicio humano** (lo que el sistema debería decir). `golden_outputs.json` captura lo que el legacy dice. La regresión check los compara con lo nuevo y muestra los tres.

4. Crear `scripts/generate_golden_outputs.py`:
   - Lee `golden_wallets.json`
   - Corre `middle_man_detector.detect()` y `find_middle_man()` legacy sobre cada una
   - Serializa outputs a `tests/fixtures/golden_outputs.json`
   - **Concurrencia limitada**: `asyncio.Semaphore(5)` para no saturar rpc_pool
   - **Idempotente**: si `golden_outputs.json` ya existe, no regenerar salvo `--force`

**Presupuesto RPC:** ~500 calls (10 wallets × ~50 calls). Vía `rpc_pool` en mainnet, en horario de baja carga. OK.

5. Crear `tests/regression_check.py`:
   - Carga `golden_outputs.json` (legacy) y corre el nuevo motor sobre las mismas wallets
   - Reporte humano-legible: tabla con `legacy vs new vs expected` por wallet
   - Detecta regresiones (wallets que el legacy detectaba bien y el nuevo falla)
   - Detecta mejoras (falsos positivos del legacy que el nuevo elimina)

### Fase 1 — Configuración y feature flag

1. Crear `config/feature_flags.py`:

```python
import os

USE_NEW_FUND_FLOW_ENGINE = os.getenv("INFLEX_NEW_FFE", "false").lower() == "true"
```

2. Por defecto `false`. Solo se activa explícitamente.

### Fase 2 — Storage layer (`fund_flow_engine/storage/`)

**Hacer antes que features porque todo lo demás depende de esto.**

`feature_cache.py` — SQLite con WAL mode:

```sql
CREATE TABLE wallet_features (
    address TEXT PRIMARY KEY,
    features_json TEXT NOT NULL,
    computed_at INTEGER NOT NULL,
    ttl_seconds INTEGER NOT NULL DEFAULT 1800
);
CREATE INDEX idx_computed_at ON wallet_features(computed_at);
```

`edge_index.py`:

```sql
CREATE TABLE edges (
    from_address TEXT NOT NULL,
    to_address TEXT NOT NULL,
    tx_digest TEXT NOT NULL,
    amount_sui REAL NOT NULL,
    timestamp_ms INTEGER NOT NULL,
    token_type TEXT,
    PRIMARY KEY (tx_digest, from_address, to_address)
);
CREATE INDEX idx_from ON edges(from_address, timestamp_ms);
CREATE INDEX idx_to ON edges(to_address, timestamp_ms);
CREATE INDEX idx_timestamp ON edges(timestamp_ms);
```

**TTL adaptativo de features:**
- CEX y wallets muy activas (>100 txs/día): 5 min
- Wallets normales: 30 min
- Wallets dormidas (sin actividad >7d): 24h

`migrations.py` debe ser idempotente. DB en `data/fund_flow_cache.db`.

### Fase 3 — Feature Extractor (`fund_flow_engine/features/`)

`WalletFeatures` dataclass con estas categorías:

**Temporal (las más importantes):**
- `dwell_time_median_seconds`: tiempo mediano entre recibir y reenviar (>90% del monto)
- `dwell_time_p25`, `dwell_time_p75`
- `activity_burst_score`: 0-1, qué tan en ráfagas opera
- `lifetime_days`, `first_seen_ms`, `last_seen_ms`

**Topología:**
- `unique_senders`, `unique_receivers` (últimas 50 txs)
- `fan_in_ratio`, `fan_out_ratio`
- `sender_creation_overlap`: senders creados en ventana ±7 días entre sí
- `common_funding_source_count`: senders con fuente común a 2 hops

**Comportamiento:**
- `token_diversity`: tokens distintos manejados
- `liquid_token_ratio`: % volumen en SUI/USDC/USDT
- `inflow_outflow_retention`: 1 - (outflow/inflow)
- `cex_proximity_hops`: distancia mínima a CEX conocida
- `dex_interaction_count`: txs con DEXs conocidos

**REGLA CRÍTICA:** `dwell_time_median > 24h` → fuerte señal de **NO es MM** sin importar otras features. Un MM real tiene dwell <2h en >70% de sus passthroughs.

### Fase 4 — Classifiers (`fund_flow_engine/classifiers/`)

Reemplazar scoring `+30/+20/threshold 40` por **sistema bayesiano**:

```python
def classify_middle_man(features: WalletFeatures) -> tuple[float, dict]:
    """
    Returns (probability [0,1], evidence_dict).
    Evidence dict permite explicabilidad: qué features contribuyeron y cuánto.
    """
    # Prior: P(MM) = 0.05 (base rate empírico)
    # Cada feature aporta likelihood ratio (LR)
    # posterior = (prior * LR_total) / (1 - prior + prior * LR_total)
```

**Likelihood ratios iniciales (calibrar con golden tests):**

| Feature | Condición | LR |
|---|---|---|
| `dwell_time_median` | < 1h | 8.0 |
| `dwell_time_median` | 1-6h | 3.0 |
| `dwell_time_median` | 6-24h | 1.0 |
| `dwell_time_median` | > 24h | 0.1 |
| `inflow_outflow_retention` | < 0.1 | 4.0 |
| `fan_in >= 5 AND fan_out <= 2` | — | 3.5 |
| `liquid_token_ratio` | > 0.9 | 2.0 |
| `token_diversity` | > 8 | 0.3 |

**Para CEX (`cex.py`)** — señales específicas, no solo "mucho volumen":
- `withdrawal_batching_score`: txs salientes a >5 destinos en <60s
- Actividad 24/7 (sin gaps >6h en 7 días)
- `deposit_consolidation_pattern`: recibe de >50 senders únicos en 7d, todos pequeños
- Presencia en lista seedeada de CEX (Fase 8)

### Fase 5 — CEX Bridge Detection (`fund_flow_engine/patterns/cex_bridge.py`)

**Pieza diferenciadora del producto.** Para una tx que deposita en CEX, busca withdrawals que matcheen:

```python
def detect_cex_bridge(
    deposit_tx: Transaction,
    cex_address: str,
    time_window_hours: int = 24
) -> list[BridgeMatch]:
    """
    Match criteria:
    - Withdrawal de cex_address en ventana [deposit_time + 1min, deposit_time + 24h]
    - Cantidad: ±5% del deposit_amount (tolerar fees)
    - Si destination wallet creada en últimas 24h: confidence boost

    Returns: [BridgeMatch(deposit_tx, withdrawal_tx, confidence, dest_wallet)]
    """
```

**Confidence scoring:**
- Match exacto (±1%) en <1h: 0.85
- Match aproximado (±5%) en <6h: 0.65
- Destination wallet nueva (<24h): +0.15
- Múltiples matches simultáneos (deposit dividido): -0.10 por ambigüedad

**Output:** enriquece el grafo con edges tipo `cex_bridged` que conectan wallets sin tx directa.

### Fase 6 — Multi-nivel (`fund_flow_engine/clustering/propagation.py`)

BFS limitado en profundidad con propagación de probabilidad:

```python
def trace_fund_flow(
    start_address: str,
    max_depth: int = 5,
    min_probability: float = 0.3
) -> FlowGraph:
    """
    BFS desde start_address. Para cada hop:
    - P(este nodo es MM/CEX/owner_network | features)
    - Propaga: P(camino) = P(nodo_1) * P(nodo_2) * ...
    - Corta cuando P(camino) < min_probability
    - Prioriza caminos de mayor probabilidad acumulada (priority queue)
    """
```

### Fase 7 — Owner Clustering (`fund_flow_engine/clustering/owner_cluster.py`)

**Antes de implementar, leer `wallet_cluster.py` (1037 LOC) y proponer plan ajustado.** Si ya implementa parte de esto, **extender** en lugar de duplicar. Si tiene su propia interfaz, decidir si vale la pena unificar o si conviven.

Algoritmo:
- Inputs: co-funding, common destination, temporal correlation, behavioral fingerprint
- Output: clusters de wallets con confidence ("estas 4 wallets son del mismo usuario, 87%")

### Fase 8 — API Facade (`fund_flow_engine/api.py`)

**Compat layer (firma EXACTA del legacy):**

```python
def detect_compat(address: str) -> dict:  # mismo output shape
def find_middle_man_compat(address: str) -> dict:  # mismo output shape
```

**Nuevas funciones:**

```python
def analyze_wallet(address: str) -> WalletAnalysis
def trace_flow(address: str, depth: int = 5) -> FlowGraph
def find_owner_cluster(address: str) -> OwnerCluster
def detect_cex_bridges(address: str) -> list[BridgeMatch]
```

### Fase 9 — Seed CEX Hot Wallets (`scripts/seed_cex_wallets.py`)

Añade a la KB hot wallets de: Binance, OKX, Bybit, Kucoin, Gate, MEXC, Bitget en SUI.

```python
# Etiquetar en KB:
{
    'category': 'wallets',
    'label': 'CEX',
    'metadata': {'exchange': 'binance', 'type': 'hot_wallet'}
}
```

Verificar direcciones con `sui_graphql.py` antes de insertar.

### Fase 10 — Regression & Validation final

1. Correr `tests/regression_check.py` contra golden tests
2. Reporte:
   - **Precision:** wallets correctamente clasificadas (no perder detecciones del legacy)
   - **Falsos positivos eliminados** (wallets que el legacy marcaba mal y el nuevo no)
   - **Regresiones** (wallets que el legacy clasificaba bien y el nuevo falla) — **bloqueante para merge si hay alguna**
3. Tabla comparativa por wallet: legacy_output | new_output | expected | delta

---

## 6. CONSTRAINTS DE PERFORMANCE

- **Hardware:** Hetzner CAX11 ARM64, 4GB RAM. NO Neo4j, NO scikit-learn entrenando modelos pesados, NO cargar grafos enormes en memoria.
- **RPC:** Toda query SUI debe pasar por `rpc_pool.py`. Nunca llamar endpoints directamente.
- **Concurrencia:** Paralelizar con `asyncio` (precedente en el código). Limitar concurrencia con `Semaphore` para no saturar pool.
- **Cache agresivo:** Asumir que el mismo address se consulta múltiples veces. TTL adaptativo según volatilidad.
- **DB:** SQLite con WAL mode. Índices en columnas de query frecuente.

---

## 7. ENTREGABLES

1. `fund_flow_engine/` módulo completo
2. `middle_man_detector/` paquete con routing y `_legacy_impl.py`
3. `config/feature_flags.py`
4. `tests/test_fund_flow_golden.py` + fixtures
5. `tests/regression_check.py` con reporte humano-legible
6. `scripts/generate_golden_outputs.py`
7. `scripts/seed_cex_wallets.py`
8. `fund_flow_engine/README.md` documentando arquitectura, decisiones y cómo extender con nuevos patterns

---

## 8. ANTES DE EMPEZAR

**Leer estos archivos del repo y proponer plan ajustado si encuentras decisiones que no encajan con el código real:**

1. `middle_man_detector.py` (908 LOC) — entender lógica actual
2. `wallet_investigator.py` (944 LOC) — específicamente línea ~233 y uso de `MiddleManResult`
3. `wallet_cluster.py` (1037 LOC) — para Fase 7, evaluar overlap
4. `rpc_pool.py` (519 LOC) — interfaz de uso
5. `sui_graphql.py` (771 LOC) — qué queries están disponibles
6. Schema de `data/knowledge.db` — confirmar si hay tabla `edges` o equivalente

**Pregunta crítica a confirmar antes de Fase 0:**

> ¿La KB tiene tabla `edges` indexada o las txs se queryan siempre vía RPC al vuelo?

Si NO hay edges cacheada:
- El SELECT de golden wallets no puede ordenar por activity count → ordenar por `last_seen`
- Fase 2 (storage/edge_index.py) se vuelve **crítica desde el inicio**, no opcional

---

## 9. DECISIONES ABIERTAS (preguntar al owner durante implementación)

- Valores exactos de likelihood ratios en classifiers — los del spec son educated guesses, calibrar con golden tests
- Si `find_owner_cluster` vive en `fund_flow_engine` o se integra en `wallet_cluster.py` (decidir tras leer `wallet_cluster.py`)
- Lista exacta de DEXs conocidos en SUI para `dex_interaction_count` (Cetus, Turbos, Aftermath, Kriya — confirmar)
- Direcciones exactas de CEX hot wallets en SUI para Fase 9 (búsqueda manual + verificación con sui_graphql)

---

## 10. CRITERIOS DE ACEPTACIÓN

El refactor se considera completo cuando:

- [ ] Todos los endpoints existentes (`/api/wallet/*`) responden idénticamente con flag OFF
- [ ] Con flag ON, golden tests pasan sin regresiones
- [ ] Con flag ON, al menos el 50% de los falsos positivos del legacy se eliminan
- [ ] Multi-nivel (depth >= 3) funciona y propaga probabilidades correctamente
- [ ] CEX bridge detection encuentra al menos 1 caso real en las wallets golden si aplica
- [ ] Cache de features funciona (segundo análisis del mismo address es >10x más rápido)
- [ ] `fund_flow_engine/README.md` documenta arquitectura para futuros mantenedores
- [ ] No hay regresiones en wallets que el legacy clasificaba correctamente

---

**Fin del spec. Empezar por Fase 0 tras confirmar la pregunta crítica de la sección 8.**
