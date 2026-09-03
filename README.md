# TRATA

TRATA is a disaster-management and emergency-response prototype. It brings together citizen incident reporting, risk assessment, map-based situational awareness,rescue-unit coordination, public-alert workflows, and a graph-based priority-response module.

The application is intended for demonstration, academic, and prototype use. Its Sundarbans routing network is deterministic sample data, not a live emergency dispatch or road-navigation service.

## Overview

The host application provides:-

- citizen reporting, including SOS-style reports and location-based risk assessment;
- deterministic incident-priority calculation and optional Gemini-assisted triage;
- an authority dashboard for report review, rescue-unit assignment, and alert publishing;
- Leaflet-based visualization of reports, sensors, shelters, hazard zones, and rescue units;
- simulation, historical analysis, IoT, shelter, and multi-source operational-intelligence views; and
- Step 4 Priority Response, which batches incoming incidents, orders them by priority, allocates personnel, and routes responses through a deterministic graph.

## System Architecture

Step 4 is integrated into the existing application; it is not a separate service or application. The priority score calculated upstream remains the source of truth throughout dispatch processing.

```text
Citizen Incident
        ↓
Existing Priority Calculation
        ↓
Intake Buffer
        ↓
Persistent Priority Queue
        ↓
Step 4 Response Engine
        ↓
Resource Allocation + Graph Routing
        ↓
ResponseResult
        ↓
Authority / Dispatch Workflow
```

## Step 4 — Priority Response

### Intake Buffer

Incoming incidents first enter a temporary in-memory intake buffer, implemented as an array. A configurable `PRIORITY_BUFFER_WINDOW_MS` window (currently 5 seconds) batches incidents that arrive close together before dispatch ordering begins. The buffer performs no routing and does not sort incidents.

### Priority Queue

After the buffer window expires, buffered incidents are inserted into the persistent `PriorityQueue`. The queue orders entries by descending upstream `priorityScore`; entries with the same score use deterministic arrival-sequence tie-breaking. The highest-priority pending incident is selected first.

### Non-Preemptive Dispatch

An active dispatch is not interrupted by a later incident. Incidents received while dispatch is active are placed into the intake buffer. Once the current response completes, that buffer is flushed into the same persistent priority queue, and the next highest-priority pending incident is selected. If no work remains, the system returns to the idle state and waits for the next buffer cycle.

```text
Incoming incidents
        ↓
Intake Buffer
        ↓
Buffer Window
        ↓
Priority Queue
        ↓
Highest-Priority Incident
        ↓
Step 4 Processing
        ↓
Dispatch
        ↓
New incidents during dispatch → Intake Buffer
        ↓
Current dispatch completes
        ↓
Flush Buffer → Priority Queue
        ↓
Next Dispatch
```

## Graph-Based Routing

Step 4 uses a deterministic, manually constructed Sundarbans prototype network with:

- 50 location/road nodes (`s-01` through `s-50`);
- 5 government-center nodes (`g-01` through `g-05`); and
- 55 graph nodes in total.

Each node has a static latitude and longitude. Connectivity is manually defined as a weighted, undirected graph with symmetric edges. Edge weights are calculated using Haversine geographic distance, and each edge has an availability state. The dataset is a realistic-looking prototype network, not a live real-world road network.

## Coordinate Mapping

An incident’s latitude and longitude are mapped to the nearest graph node using Haversine geographic distance. The selected graph node becomes the routing origin for Step 4.

## A* Routing

The router uses A* search:

- `g(n)` is the accumulated route distance;
- Haversine distance to the target is the heuristic `h(n)`;
- `f(n) = g(n) + h(n)`;
- unavailable edges are ignored;
- successful paths are reconstructed and return their total route distance.

Every available government center is evaluated. Unreachable centers are discarded, and the reachable center with the shortest route distance is selected. If none can be reached, the response result has status `NO_FEASIBLE_ROUTE`.

## Resource Allocation

Personnel allocation is based on the existing priority-score thresholds:

| Priority score | Personnel |
| --- | ---: |
| 80–100 | 10 |
| 65–79 | 8 |
| 45–64 | 6 |
| Below 45 | 4 |

## Graph Blocking and Recovery

Graph edge availability can be changed using incident-area coordinates. Blocking marks the edges associated with the nearest graph node as unavailable; recovery marks them available again. This changes availability only—the underlying Haversine-derived distance is retained.

## Government Centers

The system contains five government-center graph nodes. When building a response, Step 4 evaluates each available center and selects the reachable center with the minimum route distance. The repository intentionally provides IDs rather than real-world center names.

## Project Structure

```text
src/
├── App.tsx                         # Host-application integration and state
├── components/                     # Citizen, authority, map, simulation, and analysis views
├── data/
│   └── disasterData.ts              # Prototype regions and simulation data
├── priorityResponse/
│   ├── types.ts                     # Step 4 domain types and response contracts
│   ├── graph.ts                     # Haversine, graph state, availability, and coordinate mapping
│   ├── graphData.ts                 # Static Sundarbans nodes and manually defined edges
│   ├── priorityQueue.ts             # Persistent priority ordering and tie-breaking
│   ├── astar.ts                     # A* pathfinding and center selection
│   ├── responseEngine.ts            # Buffer lifecycle, dispatch orchestration, and response generation
│   └── tests/
│       └── priorityResponse.test.ts # Step 4 tests
├── utils/
│   └── riskCalculators.ts           # Flood, landslide, and response-priority calculations
└── main.tsx                         # React entry point
server.ts                            # Express API and optional Gemini integrations
```

## Run Locally

**Prerequisite:** Node.js

1. Install dependencies:

   ```bash
   npm install
   ```

2. Optionally create a `.env` file and set a Gemini API key for the server’s AI-assisted endpoints:

   ```env
   GEMINI_API_KEY=your_api_key
   ```

   Without a key, the server uses its implemented deterministic fallback responses for those endpoints.

3. Start the development server:

   ```bash
   npm run dev
   ```

4. Open the application at `http://localhost:3000`.

## Validation Commands

```bash
npm test
npm run lint
npm run build
```

`npm test` runs the Step 4 test suite, `npm run lint` performs TypeScript checking, and `npm run build` creates the production client and server bundles.
