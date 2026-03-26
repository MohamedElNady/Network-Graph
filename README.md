# Network Graph Widget

An interactive entity relationship network graph widget for Mendix 10/11. Visualize nodes and edges from Mendix data sources with rich overlays, filtering, theming, and investigation-grade features.

---

## Feature Overview

| # | Feature | Config group |
|---|---|---|
| 1 | Core graph — nodes + directed edges from Mendix datasources | Node / Edge data source |
| 2 | 6 built-in node types with distinct shapes + colors | Node type styles |
| 3 | Custom node type styles — any shape, any color, per type | Node type styles |
| 4 | Image rendering in any shape via ctxRenderer | Node data source |
| 5 | Risk score visualization — configurable emphasis borders per tier | Node data source |
| 6 | Resizable side panel — hero image, attribute rows, connections | Side panel |
| 7 | Smart focus mode — dim unrelated nodes on click | Focus mode |
| 8 | Live search with node highlight and camera fit | Search |
| 9 | Lazy expand — double-click appends nodes without layout reset | Actions |
| 10 | Light / Dark / System theme + end-user toggle button | Theme |
| 11 | Alert badges — severity-colored circles on nodes from a separate datasource | Alerts |
| 12 | Timeline filtering — date range pickers or dual-thumb slider | Timeline |
| 13 | Advanced filter panel — filter by type, group, risk score | Filter panel |
| 14 | Source tags — colored pills below nodes showing the source system | Source tags |
| 15 | RTL layout — full mirror for Arabic / Hebrew deployments | Visualization → Theme |
| 16 | Audit trail hook — write timestamp + fire action on every node view | Side panel → Actions |
| 17 | Metrics bar — live node/edge/visible/score/badge counts below canvas | Advanced → Metrics bar |
| 18 | Investigation Mode — one-click preset: focus on + alerts highlighted + filter open | Advanced → Investigation Mode |

---

## Domain Model

### NetworkNode (required)

| Attribute | Type | Used by |
|---|---|---|
| `NodeID` | String | All features — unique node key |
| `Label` | String | Node label on graph |
| `NodeType` | Enum / String | Node styling, filter by type |
| `ImageUrl` | String (optional) | Node image + side panel hero |
| `RiskScore` | Integer (optional) | Risk border, filter by risk |
| `Group` | String (optional) | Focus coloring, filter by group |
| `EventDate` | DateTime (optional) | Timeline filtering |
| `SourceSystem` | String / Enum (optional) | Source tag pills |

### NetworkEdge (required)

| Attribute | Type | Used by |
|---|---|---|
| `FromNodeID` | String | Edge routing |
| `ToNodeID` | String | Edge routing |
| `RelationType` | String (optional) | Edge label, side panel connections |
| `Weight` | Integer 1–10 (optional) | Line thickness |
| `EdgeDate` | DateTime (optional) | Timeline edge filtering |

### NetworkAlert (optional — for alert badges)

| Attribute | Type | Notes |
|---|---|---|
| `AlertNodeID` | String | Must match a `NetworkNode.NodeID` |
| `Severity` | String / Enum | `critical`, `high`, `medium`, `low`, `info` |
| `AlertType` | String (optional) | Category label (e.g. "Watchlist", "Sanction") |

---

## Widget Properties Reference

### Node data source

| Property | Type | Required | Notes |
|---|---|---|---|
| Nodes | Datasource | Yes | Returns list of NetworkNode |
| Node ID | Attribute | Yes | String / AutoNumber / Long |
| Node label | Attribute | Yes | String |
| Node type | Attribute | Yes | String / Enum — drives styling |
| Node group | Attribute | No | String — optional cluster grouping |
| Node image URL | Attribute | No | String — rendered as node photo |
| Risk score | Attribute | No | Integer / Decimal — ≥ 70 = red border |
| Node timestamp | Attribute | No | DateTime — for timeline filtering |
| Source attribute | Attribute | No | String / Enum — for source tag pills |

### Edge data source

| Property | Type | Required | Notes |
|---|---|---|---|
| Edges | Datasource | Yes | Returns list of NetworkEdge |
| From node ID | Attribute | Yes | Must match a Node ID value |
| To node ID | Attribute | Yes | Must match a Node ID value |
| Edge label | Attribute | No | String — shown on connector |
| Edge weight | Attribute | No | Integer 1–10 |
| Edge timestamp | Attribute | No | DateTime — for timeline edge filtering |

### Actions

| Property | Notes |
|---|---|
| On node click | Microflow / nanoflow — fires after Selected node ID is written |
| Selected node ID | String attribute — receives the clicked node ID |
| On node expand | Fires on double-click — use to fetch second-degree connections |
| Append on expand | When true, new nodes are appended (preserves layout); when false, full replace |

### Side panel

| Property | Default | Notes |
|---|---|---|
| Show side panel | false | Opens on node click |
| Panel layout | Simple | Simple (scrollable list) or Tabs (Details / Connections) |
| Panel width (px) | 320 | Initial width; user can drag-resize 200–700 px |

### Focus mode

| Property | Default | Notes |
|---|---|---|
| Enable focus mode | false | Dims unrelated nodes/edges on click |
| Focus depth | Level 1 | Level 1 = direct neighbours; Level 2 = neighbours-of-neighbours |

### Investigation Mode

| Property | Default | Notes |
|---|---|---|
| Allow investigation mode | false | Shows a `◎ Investigate` toggle button in the topbar |
| Investigation mode (attribute) | — | Optional `Boolean` attribute — page can drive the mode on/off; widget writes back on toggle |
| On investigation toggle | — | Optional action fired on every toggle (on or off) |

When active:
- Focus mode turns on automatically (same depth as **Focus depth** setting)
- The filter panel opens automatically (if **Show filter panel** is enabled)
- All nodes that have badge alerts get their border highlighted in the badge's worst-severity color at full width 4 — immediately visible regardless of focus state
- Badge nodes outside the current focus neighborhood dim to 45% opacity instead of 12% — they stay discoverable
- An amber banner appears below the topbar: `◎ Investigation Mode · N alerts` with a `✕` exit button

### Search

| Property | Default | Notes |
|---|---|---|
| Show search box | false | Renders above the canvas |
| Search attribute | — | Node attribute to match against (String / Enum) |
| Auto-focus camera | true | Fits view to matching nodes after each keystroke |

### Node type styles

Object list — one row per type. Overrides built-in defaults per type name.

| Property | Notes |
|---|---|
| Type name | Must match enum key / string value (case-insensitive) |
| Shape | Circle, Circular image, Diamond, Star, Triangle up/down, Square, Box, Ellipse, Hexagon, Text |
| Background color | Hex — node fill |
| Border color | Hex — node border |
| Size (px) | Node radius |

### Theme

| Property | Default | Notes |
|---|---|---|
| Default theme | Light | Light, Dark, or System (follows OS preference) |
| Allow user theme toggle | true | Shows a Light/Dark pill button bottom-left of canvas |

### Alerts

| Property | Notes |
|---|---|
| Alert data source | Optional list of NetworkAlert objects |
| Alert node ID | Attribute — links alert to a node by ID |
| Alert severity | Attribute — `critical` / `high` / `medium` / `low` / `info` |
| Alert type | Attribute — optional category label |

Badges appear as colored circles at the top-right corner of each alerted node. Multiple alerts per node are merged — the badge shows the worst severity color and total count.

### Timeline

| Property | Default | Notes |
|---|---|---|
| Show timeline bar | false | Adds a date filter bar above the canvas |
| Timeline mode | Date range | Date range (two date pickers) or Slider (dual-thumb) |
| Node timestamp | — | DateTime attribute on node datasource |
| Edge timestamp | — | DateTime attribute on edge datasource |
| Hide undated edges | false | When true, edges without a timestamp are hidden while timeline is active |

### Filter panel

| Property | Default | Notes |
|---|---|---|
| Show filter panel | false | Adds a slide-in filter button top-left of canvas |
| Filter by type | true | Type checkbox section in panel |
| Filter by group | true | Group checkbox section in panel |
| Filter by risk score | false | Risk range (min/max) inputs in panel |

### Source tags

| Property | Default | Notes |
|---|---|---|
| Show source tags | false | Renders a colored pill below each node |
| Source attribute | — | String / Enum — value shown in the pill |

Each unique source system gets a consistent color derived from its name (8-color deterministic palette).

### Appearance

| Property | Default | Notes |
|---|---|---|
| Height (px) | 600 | Fixed canvas height |
| Show node labels | true | Toggle node label visibility |
| Show edge labels | false | Toggle edge label visibility |
| Enable physics | true | Barnes-Hut force simulation |
| Layout algorithm | Force-directed | Force-directed, Hierarchical top-down, Hierarchical left-right |

---

## Node Type Defaults

| Type key | Shape | Color | Built-in use |
|---|---|---|---|
| `person` | Circular image (or dot) | Blue | Subjects, associates |
| `flight` | Triangle | Green | Flight records |
| `social` | Dot | Purple | Social media accounts |
| `financial` | Diamond | Amber | Accounts, transactions |
| `alert` | Triangle down | Red | Watchlist hits, flags |
| `document` | Square | Gray | Passports, visas, permits |

Any `nodeType` value not listed above and not configured in **Node type styles** renders as a gray dot.

---

## Overlay Interaction Model

When multiple filter features are active simultaneously, the widget uses an **intersection** model:

- Each active feature (focus mode, search, timeline, filter panel) computes a set of visible node IDs
- The visible set is the **intersection** of all active sets — a node must pass all active filters to be visible
- Dimmed nodes appear at 12% opacity; dimmed edges at 4% opacity
- Deactivating a feature (clearing search, clicking empty canvas) removes it from the intersection
- **Investigation mode** applies on top: badge nodes that would be dimmed are held at 45% opacity and highlighted with their alert color so they remain discoverable even outside the focus neighborhood

---

## Risk Score Reference

| Score | Border | Tooltip label |
|---|---|---|
| 0–39 | Normal 1.5 px | Low |
| 40–69 | Normal 1.5 px | Medium |
| 70–100 | Red 3 px | High |

---

## Alert Severity Reference

| Severity | Badge color |
|---|---|
| `critical` | Dark red `#A32D2D` |
| `high` | Dark orange `#C05621` |
| `medium` | Amber `#854F0B` |
| `low` | Green `#3B6D11` |
| `info` | Blue `#185FA5` |

---

## Microflow Patterns

### ACT_Graph_LoadInitial
```
Input: $Investigation

1. Delete existing NetworkNode + NetworkEdge for this investigation
2. Call integration SUBs to fetch nodes + edges from source systems
3. Commit all new objects
4. Refresh datasources → widget renders initial graph
```

### ACT_Graph_ExpandNode (double-click)
```
Input: $Investigation

1. Read $Investigation/SelectedNodeID
2. Call integration API for second-degree connections of that node
3. Create new NetworkNode + NetworkEdge objects (do NOT delete existing)
4. Commit + refresh → widget appends new nodes, preserves layout
   (requires "Append on expand" = true in widget config)
```

### ACT_Graph_OpenDetail (single click)
```
Input: $Investigation

1. Retrieve NetworkNode WHERE NodeID = $Investigation/SelectedNodeID
2. Show PG_NodeDetail with NetworkNode as context
```

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| Graph not rendering | Check browser console for errors. Ensure `.mpk` is in `widgets/` and App Directory was synchronized. |
| Nodes all overlap | Enable physics or switch to a hierarchical layout. |
| Node click not firing | Confirm `selectedNodeId` attribute is set AND an action is configured. |
| Images not showing | Verify the URL is publicly accessible from the browser. `circularImage` shape requires a URL. |
| Alert badges not visible | Confirm `badgeDataSource` is set, `badgeNodeId` matches a node ID, and the datasource is populated. |
| Timeline not filtering | Confirm `nodeTimeAttribute` is set to a DateTime attribute on the node datasource. |
| Filter panel empty | Panel populates from loaded data. Confirm nodes are loaded before opening the panel. |
| Source tags not rendering | Confirm `showSourceTag` is enabled and `nodeSource` attribute is mapped. |
| Dark mode colors not restoring | Theme changes update DataSet in-place without resetting layout. If colors look wrong, check that no other widget or CSS is overriding `.network-graph-wrapper`. |
| Investigation mode button not showing | Set **Allow investigation mode** to `true` in **Advanced → Investigation Mode**. |
| Investigation mode not highlighting alerts | Confirm `badgeDataSource` is configured and at least one badge is loaded. Alert highlighting only activates when badges exist. |
| Investigation mode not opening filter panel | Enable **Show filter panel** in Features. The filter panel must be enabled for investigation mode to open it automatically. |

---


