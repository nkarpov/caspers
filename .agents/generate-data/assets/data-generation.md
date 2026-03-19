# Recipe: Data Generation

How to generate coherent business data. This recipe can be used standalone or as part of the `build-business` skill.

## The Three Data Types

A business demo needs some combination of:

1. **Seed data** — static dimensional tables (entities that exist in the world)
2. **Event stream** — things that happen over time (the core simulation)
3. **Unstructured documents** — PDFs, images, text files (for document intelligence / RAG)

Not every business needs all three. A pure analytics demo might only need seed data + events. A document intelligence demo might skip the event stream.

---

## 1. Seed Data (Dimensions)

Seed data defines the world. Small, static tables loaded once.

### Pattern

Each entity becomes a parquet file with:
- A primary key (integer or string ID)
- Descriptive fields (names, types, categories)
- Numeric fields (prices, capacities, whatever the domain needs)
- Foreign keys to other entities

### Relationships

Entities form a graph. Common patterns:

```
# Hierarchy (1:many)
brand → items          (airline → routes, hospital → departments)

# Junction table (many:many with attributes)
brand_locations: which brands operate at which locations, with time windows

# 1:1 extension
brand → menu           (airline → fleet_assignment)
```

### Key Principle: Referential Integrity

Every foreign key in every table — and every reference in every event body — must resolve to a real row in a dimension table. This is the foundation of coherence. If an event body contains `{"brand_id": 7}`, brand 7 must exist with a name and related data.

### How to Generate

The LLM should generate realistic seed data using its world knowledge (real airport codes, realistic route networks, sensible pricing). Keep seed data small (tens to hundreds of rows) — the event generator creates volume.

For a new business, identify:
1. **What entities exist?** (airlines, airports, aircraft, passengers, routes)
2. **How do they relate?** (airlines operate routes, routes connect airports)
3. **What fields matter for downstream?** (if the agent needs `loyalty_tier`, it must be on the passenger entity now)
4. **What volume?** (4 airports is a demo, 400 is a load test)

### Worked Example: Ghost Kitchen

```
brands (24 rows): brand_id, brand_name, cuisine_type, launch_date, status
locations (4 rows): location_id, name, city, lat, lon, base_orders_day, growth_rate_daily
items (181 rows): item_id, brand_id→brands, category_id→categories, menu_id→menus, name, price, calories
categories (112 rows): category_id, category_name, description
menus (24 rows): menu_id, brand_id→brands, menu_name, description
brand_locations (~50 rows): brand_id→brands, location_id→locations, start_day, end_day, growth_rate_monthly
```

---

## 2. Event Stream

The event stream is the heartbeat of the demo. Events flow through time, reference seed data, and get consumed by everything downstream.

### Architecture: Canonical Dataset + Replay Engine

**Why not generate events live?** Live generation is fragile (job crashes = data stops), slow to backfill, and hard to reproduce. Instead:

1. **Offline generation**: Pre-generate a canonical dataset covering N days (e.g., 90 days)
2. **Replay engine**: A scheduled job replays events at configurable speed with checkpoint-based resumption

This gives you: instant historical backfill, reproducible data, configurable speed, and crash resilience.

### Event Schema

The event envelope is minimal. Only two fields are truly universal:

```
event_type: string    # what happened
ts: timestamp         # when it happened
body: string (JSON)   # everything else
```

Everything domain-specific lives in `body`. There is no prescribed `location_id`, `entity_id`, or `sequence` field at the envelope level — those are body fields if the business needs them. A ghost kitchen order has `order_id` and `location_id` in its body. A game event might have `player_id` and `grid_x, grid_y`. A banking transaction might have `account_id` and `branch_code`.

The Blueprint declares which body fields serve which roles (primary entity key, dimensional reference, spatial coordinate, etc.) so downstream layers know how to interpret them.

### Event Lifecycle: State Machine

Events often follow a lifecycle — some entity progresses through states. Model this as a directed graph:

```
# Ghost kitchen lifecycle (linear)
order_created → gk_started → gk_finished → gk_ready → driver_arrived → driver_picked_up → [driver_ping...] → delivered

# Airline lifecycle (branching)
booking_created → check_in → boarding → departed → [flight_position...] → landed → completed
                ↘ cancelled                    ↘ diverted → completed
                           ↘ no_show
```

Each state transition has:
- **Weight**: probability of taking this path (0.85 check_in, 0.15 cancelled)
- **Delay**: time distribution between states (lognormal with configurable median)
- **Body schema**: what data this event carries

Not all businesses have a lifecycle. Some just have independent events (page views, sensor readings, transactions). The lifecycle pattern is common but not mandatory.

### Canonical Dataset Generator

The offline generator walks the state machine to produce a parquet file of pre-generated events. See `assets/canonical_generator_pattern.md` for the full algorithm with worked examples. The key pieces:

1. **State machine definition** — states, transitions with weights, delay distributions
2. **Context factory** — picks which seed data applies to each entity lifecycle
3. **Body generators** — per-event-type functions producing compact columns from seed data
4. **Demand config** — volume, time-of-day curves, growth rates

The output is a compact parquet (one column per body field, nulls where not applicable) which is more efficient than JSON. The replay engine expands columns to JSON at replay time.

### Body Schemas

Each event type defines what's in its body. Bodies reference seed data:

```yaml
# Ghost kitchen: order_created body
order_id: string                # primary entity key
location_id: ref(locations)     # dimensional reference
customer_lat: float             # generated (random within location radius)
customer_lon: float             # generated
customer_addr: string           # generated
items: array                    # references: items, brands, categories
  - id: ref(items.item_id)
    name: ref(items.name)
    price: ref(items.price)
    brand_id: ref(brands.brand_id)
    qty: int (1-3, random)

# Ghost kitchen: driver_ping body (tracking event)
order_id: string                # links back to lifecycle entity
progress_pct: float             # derived (distance along route)
loc_lat: float                  # interpolated from route
loc_lon: float                  # interpolated from route
```

The key: every `ref()` must resolve to real seed data. This is what the coherence engine validates.

### Tracking Events

Some businesses have high-frequency repeated events during a specific state transition:
- **Delivery**: driver GPS pings every 60s during delivery
- **Airline**: flight position updates every 30s during flight
- **IoT**: sensor readings every 5s during operation
- **Gaming**: player position updates every tick

These are often the bulk of events (~63% in ghost kitchen). They don't have to be spatial — a heartbeat monitor emits vitals, a trading system emits price ticks.

### Demand Patterns

Realistic data needs realistic volume distribution:

```yaml
# Time-of-day patterns vary by business
restaurant:  peaks at 12:00 (3x) and 19:00 (3.5x)
airline:     peaks at 07:00 (2.5x) and 17:00 (2x)
banking:     plateau 09:00-17:00 (5x baseline)
hospital:    always-on with morning peak at 10:00

# Day-of-week and growth/decline over time
weekday_multipliers: [Mon 1.0, ... Sat 1.35, Sun 0.95]
growth_rates: per entity or per group, compounding daily
```

### Service Time Distributions

Time between lifecycle stages typically follows lognormal distributions (skewed right — most are fast, some take long):

```yaml
# Ghost kitchen example (minutes)
created_to_started: {median: 2, cv: 0.5}
started_to_finished: {median: 10, cv: 0.3}
ready_to_pickup: {median: 6, cv: 0.33}
```

---

## 3. Replay Engine

The replay engine is **business-agnostic** reusable code. It handles checkpointing, speed control, looping, and time-shifting. The only business-specific piece is a transform function that maps compact parquet columns to the output JSON format.

**Code:** `assets/replay_engine.py` — ready to use, takes a `transform_fn` parameter.

### How It Works

**Inputs:**
- Canonical dataset parquet with a `ts_seconds` column (the replay key)
- A `transform_fn`: `(DataFrame, time_shift: int) -> DataFrame`
- Config: catalog, schema, volume, start_day, speed_multiplier
- Optional: `entity_id_column` for loop-suffixing (e.g., `"order_id"`)

**State (checkpoint files in volume):**
- `_watermark`: last processed virtual timestamp (Unix seconds, monotonic)
- `_sim_start`: wall-clock time when simulation began (ISO timestamp)

**Logic:**
1. **First run**: backfill from day 0 → start_day + current time of day
2. **Subsequent runs**: advance by `elapsed_real_time × speed_multiplier`
3. **Looping**: wrap at dataset end, keep virtual timestamps monotonic, suffix entity IDs
4. **Time-shift**: project events so they appear relative to "today"
5. **Write**: JSONL to UC volume (consumed by SDP cloudFiles)
6. **Idempotent**: if job fails before checkpoint update, next run reprocesses same window

### The Business-Specific Transform

The only part that changes per business. Takes a Spark DataFrame of compact parquet columns and returns the final output DataFrame.

**Example:** `assets/example_ghost_kitchen_transform.py` shows the ghost kitchen transform — mapping `event_type_id` → strings, assembling `body` JSON from columns like `customer_lat`, `items_json`, `route_json`.

---

## 4. Unstructured Documents

Documents add a RAG / document intelligence dimension.

### Pattern

Each document type needs:
1. **The files themselves** (PDFs, images, etc.) — stored in a UC volume
2. **Structured metadata** (JSON) — what's in each file, for validation and ground truth
3. **A loading process** — copy files to volume, parse metadata into tables

### Generation Approach

For demos, documents can be:
1. **LLM-generated** — write markdown, convert to PDF (fast, good enough for demos)
2. **Template-based** — fill branded templates with seed data (more realistic)
3. **Curated** — hand-picked public domain documents (most realistic, least scalable)

The metadata JSON should always be generated alongside the documents so structured queries and RAG can be validated against ground truth.

### Worked Example: Ghost Kitchen

- 16 menu PDFs (one per brand) + `menu_metadata.json` with items, prices, nutrition, allergens
- 12 inspection report PDFs (4 locations × 3 dates) + `inspection_metadata.json` with scores, violations, corrective actions

---

## Coherence Checkpoints

After data generation, validate:

- [ ] Every entity FK resolves to a real parent row
- [ ] Every event body field that references an entity uses valid IDs from seed data
- [ ] Event timestamps are monotonically increasing within each entity lifecycle
- [ ] Tracking events fall within the time window of their parent transition
- [ ] Demand volume looks realistic (not too uniform, not too sparse)
- [ ] Seed data fields needed by downstream layers (pipeline, agent, app) are present
- [ ] If documents exist: metadata JSON matches actual file contents
