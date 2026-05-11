# BART Graph Database & Food Delivery Route Optimization

> **UC Berkeley MIDS — W205: Fundamentals of Data Engineering**
> Team: Kevin Yi, Abhay Naik, Rohan Krishnamurthi, Sudiksha Sarvepalli

---

## Table of Contents
1. [Problem Statement](#problem-statement)
2. [Dataset](#dataset)
3. [Approach & Architecture](#approach--architecture)
4. [Results](#results)
5. [Impact](#impact)
6. [Limitations & Challenges](#limitations--challenges)
7. [Learnings & Findings](#learnings--findings)
8. [Why This Project Matters](#why-this-project-matters)
9. [Skills Demonstrated (Data Scientist Portfolio)](#skills-demonstrated-data-scientist-portfolio)
10. [Tech Stack](#tech-stack)
11. [Project Structure](#project-structure)

---

## Problem Statement

**Business context:** We acted as data engineers advising the fictional company **ACME Gourmet Meals (AGM)** — a food delivery service looking to modernize its supply chain using the Bay Area Rapid Transit (BART) network as its distribution backbone.

**Core questions we needed to answer:**
1. Which BART stations are most centrally located relative to existing customers and should be prioritized as delivery pickup hubs?
2. How should AGM group/cluster BART stations to expand its delivery footprint across the entire Bay Area?
3. What is the optimal delivery route from AGM's main warehouse to any given BART station pickup location?
4. How can real-time data from delivery robots and drones be managed efficiently at scale?
5. How should persistent customer and order data be stored for business intelligence?

The overarching goal: **build an end-to-end data engineering pipeline** — from raw relational data → graph database → graph algorithm analytics → NoSQL real-time and document storage — that enables intelligent, automated food delivery decisions.

---

## Dataset

| File | Description | Size |
|---|---|---|
| `stations.csv` | 50 BART stations with latitude, longitude, and transfer time (seconds) | 50 rows |
| `lines.csv` | Station-to-line membership with sequence ordering across 6 BART lines | 114 rows |
| `travel_times.csv` | Inter-station travel times (seconds) between adjacent station pairs | 51 rows |
| `zip_codes` (DB table) | ZIP code geographic centroids + population, used for geodesic fencing | External DB |
| `customers` (DB table) | AGM customer records with street, city, state, ZIP, and closest store info | ~8,000 rows (CA only) |
| `zip_codes` (DB table) | ZIP code geographic centroids + population, used for geodesic fencing and customer geocoding | External DB |
| `sales` (DB table) | Transaction records joined to customers to compute city-level sales volume | External DB |

**BART lines covered:** Blue, Green, Red, Yellow, Orange, and additional lines

**Key data characteristics:**
- Travel times are symmetric (same in both directions) and stored once per pair
- Transfer times represent the seconds a rider needs to switch between lines at a station
- Station coordinates enable geospatial queries
- Customer exact addresses were unavailable — only ZIP codes — so precise customer locations were **randomly simulated** within a 3-mile geodesic bounding box around each ZIP centroid using WGS84 calculations (see Graph Construction details below)

---

## Approach & Architecture

The pipeline was implemented in **5 sequential notebooks**, each building on the previous:

### Part 1 — Relational Database Foundation (PostgreSQL)
- Designed a **3NF-normalized** PostgreSQL schema with three tables: `stations`, `lines`, `travel_times`
- Defined appropriate primary keys (composite where needed) and data types (`varchar`, `numeric`)
- Loaded all CSV data via `COPY` commands using Python's `psycopg2`

### Part 2 — SQL Queries for Graph Preparation
- Wrote complex multi-table JOIN queries to extract the relationships needed to build the graph:
  - Station-to-line membership
  - All valid line transfer pairs (cross joins filtered by station)
  - Directional segments between adjacent stations with travel weights
- Used **self-joins** on the `lines` table with sequence arithmetic (`b.sequence = a.sequence + 1`) to derive adjacency

### Part 3 — Graph Database Construction (Neo4j)
Built a weighted, directed graph in Neo4j via Cypher queries called from Python:

**Node types:**
- `depart <Station>` — departure node per station (100 nodes total)
- `arrive <Station>` — arrival node per station
- `<line> <Station>` — one node per station-line combination (114 nodes)

**Relationship types:**
- `depart → line_node`: weight = 0 (boarding)
- `line_node → arrive`: weight = 0 (alighting)
- `line_node → line_node (same station, different line)`: weight = transfer_time (line transfer)
- `line_node → line_node (adjacent stations, same line)`: weight = travel_time (segment traversal)

**Final graph:** 214 nodes, 436 relationships

### Part 4 — Shortest Path Verification (Dijkstra's Algorithm)
- Used **Neo4j GDS (Graph Data Science) library** with `gds.shortestPath.dijkstra.stream()` to compute time-optimal routes
- Verified 5 representative routes (Dublin→Antioch, SFO→OAK airport, Downtown Berkeley→Castro Valley, San Bruno→San Leandro, Embarcadero→Civic Center)
- Results expressed in both seconds and minutes with full step-by-step path

### Part 5 — Geospatial Analysis (Geodesic Fencing)
- Used the `geographiclib` library to compute a bounding box around each station's GPS coordinates at a given radius (1–5 miles)
- Queried a `zip_codes` PostGIS-style table to find all ZIP codes within that bounding box
- Summed population figures to estimate the **catchment population** for each station

### Part 6 — Harmonic Centrality: Customer-Station Bipartite Graph (Graph_Centrality_Algorithm)

This notebook is the most analytically sophisticated component and implements the full Harmonic Centrality pipeline end-to-end:

**Step 1 — Customer geocoding from ZIP codes:**
- Queried the `customers` table (California records only) and joined to `zip_codes` to retrieve the geographic centroid for each customer's ZIP
- Since exact street addresses weren't available as coordinates, generated a **random precise GPS location** for each customer within a 3-mile geodesic bounding box around their ZIP centroid using `geographiclib` (WGS84 ellipsoid)
- This simulates realistic customer scatter within a ZIP without fabricating data

**Step 2 — Distance computation (all customers × all stations):**
- Computed geodesic distances (miles) from every customer's simulated location to every BART station using `geod.Inverse()`
- Identified each customer's closest station and their distance to it
- Stored the full distance dictionary per customer for edge-weight assignment

**Step 3 — Bipartite graph construction in Neo4j:**
- Built a **two-node-type graph**: `Station` nodes (50) + `Customer` nodes (~8,000 CA customers)
- Created undirected `DISTANCE_LINK` edges between a customer and a station only if distance ≤ threshold (5 miles max) — this distance filter is what makes the centrality scores meaningful
- Edge weight = geodesic distance in miles

**Step 4 — Harmonic Centrality at three distance thresholds (GDS):**
- Re-projected the graph in Neo4j GDS (`gds.graph.project`) at 1-mile, 2-mile, and 5-mile customer-to-station cutoffs
- Ran `gds.closeness.harmonic.stream()` on each projection — scores rank stations by their aggregate closeness to the customer nodes they're connected to
- Results ranked all 50+ nodes (stations + customers) by centrality score; top stations surfaced as the best delivery hubs

**Step 5 — Degree Centrality (complementary metric):**
- Also ran `gds.degree.stream()` on the 5-mile graph to count raw connection volume per node
- Degree centrality answers "which station has the most customers within 5 miles?" vs. Harmonic Centrality's "which station is most efficiently close to its customers?"

**Step 6 — Additional SQL business analytics:**
- Sales volume by Bay Area city (CTE joining `sales` → `customers`)
- Average geodesic distance to the closest BART station, grouped by city
- Most frequent "closest station" ranking — which stations are the nearest hub for the most customers
- Customer city distribution across California

### Business Analytics Layer (Neo4j Graph Algorithms — AGM Proposal)

| Algorithm | Business Use Case | Key Finding |
|---|---|---|
| **Harmonic Centrality** (1-mile graph) | Identify stations for ultra-local drone delivery | Oakland/Berkeley corridor stations top-ranked |
| **Harmonic Centrality** (2-mile graph) | Prioritize stations for short-range robot delivery | **Ashby** ranked highest |
| **Harmonic Centrality** (5-mile graph) | Prioritize stations for car/truck delivery radius | **Rockridge** ranked highest |
| **Degree Centrality** (5-mile graph) | Raw count of customers within 5 miles per station | Complements harmonic score by showing volume vs. closeness |
| **Louvain Modularity** | Community detection to cluster stations for Bay Area market expansion | Identified natural geographic clusters for delivery zone planning |
| **Geodesic Fencing + Louvain** | Select the single best station per community to serve the most new customers | Stations serving 226K–624K people within 3 miles: Ashby (226K), Rockridge (317K), Glen Park (624K), Montgomery (425K), and others |
| **Dijkstra's Shortest Path** | Optimize warehouse-to-station delivery routing to minimize cost, time, and drone recharges | Fastest multi-hop BART paths computed for all origin-destination pairs |

### NoSQL Complementary Layer
- **Redis**: Real-time pub/sub messaging for delivery robot/drone location tracking and order status updates; native geospatial data support aligns with autonomous vehicle operations
- **MongoDB**: Document-oriented storage for customer profiles and purchase history; horizontal scaling and flexible schema support business growth without schema migrations

---

## Results

- **Graph database built successfully**: 214 nodes, 436 directed weighted relationships representing the entire BART network
- **Shortest paths validated** for 5 route pairs with correct total cost (seconds) and minute conversions:
  - Dublin → Antioch: **96.9 minutes** (5,813 seconds)
  - SFO → OAK Airport: **64.7 minutes** (3,882 seconds)
  - Downtown Berkeley → Castro Valley: **36.9 minutes** (2,214 seconds)
  - Embarcadero → Civic Center: **4.0 minutes** (240 seconds) — same-line, no transfers
- **Harmonic Centrality run at three thresholds** on a bipartite Station-Customer graph:
  - 1-mile graph: Oakland/Berkeley corridor stations dominate (dense existing customer base)
  - 2-mile graph: **Ashby** — top station for drone/robot deployment
  - 5-mile graph: **Rockridge** — top station for car/truck delivery radius
  - Degree Centrality (5-mile): confirmed Rockridge as the station with the highest raw customer volume within 5 miles
- **City-level SQL analytics** revealed which Bay Area cities drive the most AGM sales, which cities have customers farthest from any BART station (expansion opportunity), and which stations are the de-facto nearest hub for the largest customer segments

---

## Impact

**Operational efficiency:**
- Route optimization via Dijkstra's reduces unnecessary BART transfers, directly cutting drone recharge cycles and fuel costs
- Delivery resource allocation (drones vs. vehicles) guided by data — not guesswork

**Customer growth:**
- Community detection identifies underserved, high-population zones (e.g., South Bay via Berryessa/North San Jose: 293K residents)
- Data-driven station selection can expand AGM's serviceable market without proportional cost increases

**Scalability:**
- Graph databases traverse complex multi-hop relationships orders of magnitude faster than equivalent SQL joins
- Redis pub/sub and MongoDB's horizontal sharding enable the system to scale to thousands of concurrent delivery robots

**Strategic positioning:**
- Real-time autonomous delivery infrastructure (robots + drones) reduces labor costs and positions AGM as a logistics technology company, not just a meal delivery service

---

## Limitations & Challenges

### Technical Challenges
- **Cypher query learning curve**: Neo4j's Cypher language was entirely new; correctly constructing the graph creation logic (especially the node-relationship schema for transfers vs. segments) required iteration and peer collaboration
- **Graph design complexity**: Determining when to apply Harmonic Centrality vs. Closeness Centrality was non-trivial; connecting 50 station nodes to ~8,000 customer nodes without filtering produced a graph so dense that centrality scores became meaningless — solved by adding distance-based relationship filters during graph construction
- **Customer location approximation**: Exact customer GPS coordinates were unavailable — only ZIP codes. Random location generation within a 3-mile ZIP centroid bounding box is a reasonable approximation but introduces noise; customers near ZIP boundaries may be misassigned to the wrong station catchment area
- **Degree vs. Harmonic Centrality trade-off**: Degree Centrality counts raw connections (volume) while Harmonic Centrality weights by inverse distance (closeness efficiency) — both metrics are needed but can recommend different stations, requiring judgment to interpret which matters more for a given delivery vehicle type

### Data Limitations
- BART travel times are static averages — real-world delays, peak/off-peak schedules, and weekend service are not modeled
- Customer location data (~8,000 points) is synthetic; real deployment would require live customer data integration
- Geodesic bounding box (square approximation) introduces ~5–10% error near the corners vs. a true circular radius — geodesic circle calculation was out of scope
- ZIP code population centroids approximate real customer density but don't account for actual demographics or spending patterns

### Scope Limitations
- Redis and MongoDB implementations are proof-of-concept; production deployment would require authentication hardening, replication, and failover configuration
- Demand forecasting and customer segmentation algorithms were scoped but not fully implemented in this iteration

---

## Learnings & Findings

**Bipartite graph design unlocks richer analytics:**
The Harmonic Centrality notebook introduced a fundamentally different graph structure — a bipartite Station-Customer graph rather than a station-only transit graph. This enabled customer-centric scoring of stations rather than structural network centrality. Choosing the right graph topology for the question is as important as choosing the right algorithm.

**Harmonic vs. Degree Centrality answer different questions:**
Degree Centrality tells you which station has the most customers within range (volume). Harmonic Centrality tells you which station is most efficiently close to its customers (proximity quality). A station can rank high on volume but low on closeness if its customers are spread at the edge of the radius — both metrics are needed for a complete resource allocation picture.

**Simulating missing data with geodesic randomization:**
When only ZIP code data is available, randomly placing customers within a geodesic bounding box around the ZIP centroid is a principled approximation. It avoids point-stacking all customers at the same coordinate while preserving the general geographic distribution — a practical data engineering pattern for incomplete address data.

**Graph algorithm selection matters more than implementation:**
The most important insight was that blindly applying an algorithm to a naive graph produces useless results. Adding meaningful filters when building relationships (e.g., only connect customer nodes within X miles of a station) is what makes centrality scores interpretable and actionable.

**Relational → Graph is a deliberate design process:**
Moving data from a normalized SQL schema to a Neo4j graph is not automatic. Each SQL entity (stations, lines, transfers, segments) maps to a specific node or relationship type with carefully chosen weights. The SQL queries in Part 2 were precisely engineered to feed the graph construction in Part 3.

**NoSQL databases are complements, not replacements:**
Redis excels at ephemeral, high-frequency operations (robot location pings, order status). MongoDB handles flexible, evolving document structures (customer profiles). PostgreSQL retains relational integrity for financial records. Each tool has a regime where it dominates — the insight is knowing which regime you're in.

**Graph traversal outperforms SQL joins at scale:**
For a 50-node network, SQL joins are manageable. At 5,000 stations or millions of customer nodes, graph traversal's O(nodes + edges) complexity vs. SQL's exponential join costs makes the graph database the only viable option.

**Community detection enables business segmentation:**
Louvain Modularity revealed natural geographic groupings in the BART network that align with actual Bay Area neighborhoods — this is not just a technical curiosity but directly maps to delivery zone planning.

---

## Why This Project Matters

This project demonstrates a **complete data engineering stack** applied to a real-world logistics problem:

1. **End-to-end pipeline design**: From raw CSV ingestion → relational modeling → graph construction → algorithm application → business recommendation
2. **Multi-paradigm database fluency**: Relational (PostgreSQL), graph (Neo4j), in-memory key-value (Redis), and document (MongoDB) — each chosen for the right reason
3. **Translating algorithms to business value**: Graph algorithms aren't academic exercises here — each one answers a specific business question with a quantifiable outcome
4. **Geospatial reasoning**: Geodesic distance calculations and population-weighted station scoring demonstrate production-level spatial data engineering skills
5. **Scalability thinking**: Architecture decisions (Redis pub/sub, MongoDB horizontal scaling, graph traversal) are motivated by the limits of the alternatives at scale

For a data scientist, this project bridges the gap between analytical modeling and the data infrastructure that makes models possible in production.

---

## Skills Demonstrated (Data Scientist Portfolio)

| Category | Specific Skills |
|---|---|
| **Data Engineering** | ETL pipeline design, relational schema design (3NF), CSV ingestion, data validation |
| **SQL** | Multi-table JOINs, self-joins, CTEs, composite primary keys, aggregations, sequence-based adjacency queries, city-level sales analytics |
| **Graph Databases** | Neo4j, Cypher query language, GDS library, node/relationship modeling, graph construction from relational data, bipartite graph design |
| **Graph Algorithms** | Dijkstra's shortest path, Harmonic Centrality, Degree Centrality, Louvain Modularity community detection, geodesic fencing |
| **NoSQL Databases** | Redis (pub/sub, geospatial), MongoDB (document modeling, horizontal scaling) |
| **Geospatial Analysis** | Geodesic distance calculation (WGS84), bounding box construction, random point simulation within geofences, population-weighted catchment analysis, customer-to-station distance matrices |
| **Python** | `psycopg2`, `neo4j` driver, `pandas`, `numpy`, `geographiclib`, `random`, Jupyter notebooks |
| **Systems Architecture** | Multi-database system design, real-time vs. persistent data separation, scalability trade-off analysis |
| **Business Translation** | Converting graph algorithm outputs into delivery resource allocation, market expansion, and route optimization decisions |
| **Team Collaboration** | 4-person team project with modular notebook structure and shared database environments |

---

## Tech Stack

```
┌─────────────────────────────────────────────────────┐
│                   Data Sources                       │
│         CSV files (stations, lines, travel)          │
└──────────────────────┬──────────────────────────────┘
                       │
              ┌────────▼────────┐
              │   PostgreSQL     │  ← Relational foundation
              │  (3NF schema)    │     SQL JOIN queries
              └────────┬────────┘
                       │  Python (psycopg2)
              ┌────────▼────────┐
              │     Neo4j        │  ← Graph database
              │  (Cypher + GDS)  │     Dijkstra, Harmonic,
              └────────┬────────┘     Louvain algorithms
                       │
          ┌────────────┼────────────┐
          │                         │
 ┌────────▼────────┐     ┌─────────▼────────┐
 │      Redis       │     │     MongoDB       │
 │  (Real-time ops) │     │  (Persistent docs)│
 │  Pub/Sub + Geo   │     │  Customer profiles│
 └──────────────────┘     └──────────────────┘
```

**Languages:** Python 3, Cypher (Neo4j), SQL (PostgreSQL)

**Libraries:** `psycopg2`, `neo4j`, `pandas`, `numpy`, `geographiclib`, `math`

**Infrastructure:** Docker-based multi-container environment (PostgreSQL + Neo4j + Redis + MongoDB)

---

## Project Structure

```
project-3/
├── README.md
├── bart_map.png                            # Visual BART network map
├── stations.csv                            # 50 BART stations (lat, lon, transfer_time)
├── lines.csv                               # 114 station-line membership records
├── travel_times.csv                        # 51 inter-station travel time pairs
├── project_3_1_solution.ipynb              # Part 1: PostgreSQL schema + data loading
├── project_3_2_solution.ipynb              # Part 2: SQL queries for graph prep
├── project_3_3_solution.ipynb              # Part 3: Neo4j graph construction
├── project_3_4_solution.ipynb              # Part 4: Shortest path verification
├── project_3_5_solution.ipynb              # Part 5: Geodesic fencing + population
├── Graph_Centrality_Algorithm.ipynb        # Part 6: Harmonic & Degree Centrality (bipartite graph)
├── W205_final_project_documentation        # Project documentation
└── DATASCI205_Final_Presentation           # AGM business proposal deck
```

---

*UC Berkeley MIDS — W205 Fundamentals of Data Engineering | Project 3*
