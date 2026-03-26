# How to Configure the Network Graph Widget

This guide uses an **org chart / HR** domain as the running example. Substitute your own entity names, attributes, and microflows as needed. The widget works for any domain — supply chains, IT topology, knowledge graphs, process flows, etc.

---

## Step 1 — Build the Domain Model with Mendix Maia

Open **Maia** in Mendix Studio Pro and paste the following prompt into the **Domain Model Builder**:

---

### Maia Prompt — Domain Model Builder

```
Create a domain model for an interactive org chart and employee network with the following entities and associations:

---

Entity: NetworkNode
Purpose: Represents a single node in the graph (an employee, department, system, or document).
Attributes:
- NodeID (String, required) — Unique identifier. Used as the graph node key.
- Label (String, required) — Display name shown on the graph node.
- NodeType (String, required) — Controls shape and color. Example values: employee, department, system, document.
- ImageUrl (String, optional) — Photo URL rendered on person/employee nodes and in the side panel hero.
- Score (Integer, optional, default 0) — Generic score 0–100. Configured tiers render an emphasis border.
- Group (String, optional) — Optional grouping label used in filter panel and clustering.
- EventDate (DateTime, optional) — Timestamp used by the Timeline filter to show/hide this node.
- SourceSystem (String, optional) — Source system name rendered as a colored pill below the node (e.g. "HR", "AD", "ERP").

---

Entity: NetworkEdge
Purpose: Represents a directed relationship between two nodes.
Attributes:
- FromNodeID (String, required) — NodeID of the source node.
- ToNodeID (String, required) — NodeID of the target node.
- RelationType (String, optional) — Label shown on the connector (e.g. "reports to", "manages", "owns").
- Weight (Integer, optional, default 1) — Connector line thickness (1–10).
- EdgeDate (DateTime, optional) — Timestamp used by the Timeline filter to show/hide this edge.

---

Entity: NetworkBadge
Purpose: Badge records linked to nodes. Each badge renders as a colored circle on the node.
Attributes:
- BadgeNodeID (String, required) — Must match a NetworkNode.NodeID.
- Category (String, required) — Determines badge color via badgeTypeConfig. E.g. "approved", "flagged", "pending".
- Label (String, optional) — Additional label shown in the node badge tooltip.

---

Entity: GraphContext
Purpose: Root context entity holding the current graph session state.
Attributes:
- SelectedNodeID (String, optional) — Written by the widget when the user clicks a node.

---

Associations:
- GraphContext → NetworkNode: one-to-many
- GraphContext → NetworkEdge: one-to-many
- GraphContext → NetworkBadge: one-to-many

Note: NetworkNode and NetworkEdge are not directly associated with each other. The relationship
is expressed through string-based FromNodeID / ToNodeID, which the widget resolves at render time.
NetworkBadge links to NetworkNode by matching BadgeNodeID to NodeID at render time — no direct association needed.
```

---

## Step 2 — Microflows to Create

### ACT_Graph_LoadData
**Trigger:** Called when the user opens or refreshes the graph view.

```
Input parameter: $Context (GraphContext)

1. Delete all existing NetworkNode records associated with $Context
2. Delete all existing NetworkEdge records associated with $Context
3. Delete all existing NetworkBadge records associated with $Context
4. Call SUB_FetchNodes($Context)  → creates NetworkNode records
5. Call SUB_FetchEdges($Context)  → creates NetworkEdge records
6. Call SUB_FetchBadges($Context) → creates NetworkBadge records
7. Refresh the page data sources
```

### ACT_Graph_OpenNodeDetail
**Trigger:** On Node Click action in the widget. `SelectedNodeID` is already written before this fires.

```
Input parameter: $Context (GraphContext)

1. Retrieve first NetworkNode
   WHERE NetworkNode/GraphContext = $Context
   AND   NetworkNode/NodeID = $Context/SelectedNodeID
2. Show page PG_NodeDetail with the retrieved NetworkNode as context
```

### ACT_Graph_ExpandNode
**Trigger:** On Node Expand (double-click).

```
Input parameter: $Context (GraphContext)

1. Read $Context/SelectedNodeID
2. Call REST/integration API for second-degree connections of this node
3. Create new NetworkNode + NetworkEdge records associated with $Context
4. Do NOT delete existing nodes — widget is in Append mode
5. Commit all new objects
6. Refresh datasource → widget appends new nodes, preserves existing layout
```

---

## Step 3 — Page Setup

1. Create a page with a **Data View** over the `GraphContext` entity.
2. Inside the Data View, configure **three List datasources** (or microflow datasources):
   - One returning `NetworkNode` objects filtered by the current `GraphContext`
   - One returning `NetworkEdge` objects filtered by the current `GraphContext`
   - One returning `NetworkBadge` objects filtered by the current `GraphContext`
3. Place the **Network Graph** widget inside the Data View.
4. Map the properties as described in Step 4 below.

---

## Step 4 — Widget Property Mapping

### Core — Node data source

| Widget property | Mendix attribute | Notes |
|---|---|---|
| **Node data source** | NetworkNode list datasource | |
| Node ID | `NetworkNode/NodeID` | |
| Node label | `NetworkNode/Label` | Supports expressions — e.g. `{FirstName} + ' ' + {LastName}` |
| Node type | `NetworkNode/NodeType` | |
| Node group | `NetworkNode/Group` | |
| Node image URL | `NetworkNode/ImageUrl` | ⚠ See security note below |
| Score attribute | `NetworkNode/Score` | Optional — drives emphasis border tiers |
| Node size | `NetworkNode/Score` | Optional — per-node size override (Integer/Decimal) |
| Node timestamp | `NetworkNode/EventDate` | Optional — used by timeline filter |
| Source attribute | `NetworkNode/SourceSystem` | Optional — colored pill below node |

> **⚠ Image URL security:** Storing raw URL strings in `ImageUrl` means any misconfigured datasource could load arbitrary external URLs. For government or security-sensitive deployments, use a **Mendix FileDocument** entity instead:
> 1. Associate `NetworkNode` with a `System.FileDocument` specialization (e.g. `NodeImage`)
> 2. In your microflow, populate the `NodeImage` using `Core.storeFileDocumentContent()`
> 3. Use the `getFileUrl($NodeImage)` expression to return a Mendix-controlled URL
>
> This routes all images through Mendix's secure file storage (S3 / Azure Blob with pre-signed, time-limited URLs), enforces authentication, and prevents arbitrary URL injection.

### Core — Edge data source

| Widget property | Mendix attribute |
|---|---|
| **Edge data source** | NetworkEdge list datasource |
| From node ID | `NetworkEdge/FromNodeID` |
| To node ID | `NetworkEdge/ToNodeID` |
| Edge label | `NetworkEdge/RelationType` |
| Edge weight | `NetworkEdge/Weight` |
| Edge timestamp | `NetworkEdge/EdgeDate` |

### Actions

| Widget property | Value |
|---|---|
| On node click | `ACT_Graph_OpenNodeDetail` |
| Selected node ID | `GraphContext/SelectedNodeID` |
| On node expand | `ACT_Graph_ExpandNode` |
| Append on expand | `true` |
| On node view | `ACT_Graph_AuditNodeView` (optional — for audit logging) |
| Node viewed at | `GraphContext/NodeViewedAt` (optional — DateTime attribute) |

### Side panel

| Widget property | Recommended value |
|---|---|
| Show side panel | `true` |
| Panel layout | `Tabs (Details / Connections)` |
| Panel width (px) | `320` |
| Use custom panel content | `false` (set `true` to replace default layout with your own widgets) |
| Custom panel content | Drop Mendix widgets here when custom content is enabled |

The side panel opens as a flex sibling of the canvas — the canvas shrinks to make room rather than being overlaid.

**Header:** The panel header always shows the selected node's **label** as a bold title and its **type** as a colored pill badge below it. This gives immediate visual identity without needing to scan attribute rows.

**Hero image:** When `nodeImageUrl` is mapped, a full-width hero image renders at the top of the panel. The close button (✕) floats over the hero image as a translucent pill — it does not take up space in the header. Without a hero image the close button appears in the header row next to the title.

**Drag-resize:** A 5 px grab handle sits on the left edge of the panel. Drag it left to widen or right to narrow. Width is clamped between 200 px and 700 px. The hero image height scales proportionally with the panel width (clamped 120–220 px) so it remains well-proportioned at any size.

**Badge details:** When the selected node has badges, a "Badges (N)" section header appears in the details tab below the standard attributes. Each badge category is shown as a row with a colored dot and its count.

**Custom content:** When **Use custom panel content** is enabled, the widgets dropped into **Custom panel content** replace the default attribute list and connections tabs. The header and hero image still render above the custom content. The datasource item for the dropzone is the currently selected node — use it to bind attributes, call microflows, or embed any Mendix widget.

### Focus mode

| Widget property | Recommended value |
|---|---|
| Enable focus mode | `true` |
| Focus depth | `Level 1 — direct connections only` |

### Search

| Widget property | Recommended value |
|---|---|
| Show search box | `true` |
| Search attribute | `NetworkNode/Label` |
| Auto-focus camera on results | `true` |

### Theme

| Widget property | Recommended value |
|---|---|
| Default theme | `Light` (or `System` to follow OS) |
| Allow user theme toggle | `true` |
| Enable RTL layout | `false` — set `true` for Arabic / Hebrew deployments |

### Display

| Widget property | Recommended value |
|---|---|
| Use default type styles | `true` (uses generic 9-type starter palette when no custom types configured) |
| Show node labels | `true` |
| Show edge labels | `true` (if edge labels are mapped) |
| Physics enabled | `true` (auto-layout on load; can be paused) |
| Layout | `Force` (or `Hierarchical UD` / `Hierarchical LR`) |

### Widget height

| Widget property | Recommended value |
|---|---|
| Widget height | `600` |
| Height unit | `px` (or `%` / `vh` for responsive layouts) |

### Investigation Mode

| Widget property | Recommended value |
|---|---|
| Allow investigation mode | `false` — set `true` to show the `◎ Investigate` button in the topbar |
| Investigation mode (attribute) | `GraphContext/InInvestigationMode` (optional Boolean attribute) |
| On investigation toggle | `ACT_Graph_OnInvestigationToggle` (optional action) |

> **Domain model:** Add an optional `InInvestigationMode` Boolean attribute to `GraphContext`. The widget writes back to it on every toggle, so your page can react (e.g. show/hide an alert list, switch a tab, or log the session start time).

### Export

| Widget property | Recommended value |
|---|---|
| Enable export | `false` — set `true` to show the Export button in the toolbar |

When enabled an **⬇ Export** button appears in the topbar. Clicking it opens a panel with:
- **Format**: PNG (downloads directly) or PDF (opens a print dialog)
- **Include background**: fills the export with the canvas background color (recommended for dark mode)
- **Include labels**: temporarily hides all node/edge labels during export if disabled

The browser's native right-click "Save image as" option on the canvas is always disabled regardless of this setting.

### Metrics bar

| Widget property | Recommended value |
|---|---|
| Show metrics bar | `false` — set `true` to show the slim status bar below the canvas |

When enabled a status bar appears below the graph with the following segments (segments are only shown when relevant data is present):

| Segment | When shown |
|---|---|
| `N nodes · E edges` | Always (when data is loaded) |
| `Visible: X of N` | Only when at least one filter, search, focus, or timeline is active |
| `{scoreLabel}: X` | Only when `nodeScoreAttribute` + `scoreLevels` are configured |
| `Badges: X` | Only when `badgeDataSource` is configured and at least one badge is loaded |

### Performance

| Widget property | Recommended value |
|---|---|
| Max nodes | `0` (unlimited) — set e.g. `200` to cap large datasets |
| Enable clustering | `false` — enable to auto-cluster high-connectivity hubs |
| Cluster hub size | `5` — nodes with ≥ this many connections are clustered |

When **Max nodes** is exceeded the widget shows an amber bar: `"Showing X of Y nodes"` with a **Load all nodes (may be slow)** button. Clicking it reloads all nodes for the current session without changing the Studio Pro setting.

When **Enable clustering** is on, high-degree nodes are automatically collapsed into cluster nodes after data loads. Clicking a cluster expands it. Search automatically opens any cluster that contains a match.

### Badges

| Widget property | Mendix attribute |
|---|---|
| Badge data source | NetworkBadge list datasource |
| Badge node ID | `NetworkBadge/BadgeNodeID` |
| Badge category | `NetworkBadge/Category` |
| Badge label | `NetworkBadge/Label` |
| Badge type config | See badge starter kit below |

Badge circles are drawn directly on the vis-network canvas as colored circles at the top-right corner of each node. Multiple badges per node are merged into one circle showing the **highest-priority category color** and the total count. All individual badges are listed in the node tooltip.

### Timeline

| Widget property | Mendix attribute |
|---|---|
| Show timeline bar | `true` |
| Timeline mode | `Date range` (or `Slider`) |
| Node timestamp | `NetworkNode/EventDate` |
| Edge timestamp | `NetworkEdge/EdgeDate` |
| Hide undated edges | `false` (or `true` for strict filtering) |

The timeline bar appears between the search box and the canvas. In **Date range** mode the user selects a from/to date with two date pickers. In **Slider** mode a dual-thumb slider is shown, calibrated automatically from the min/max timestamps in the loaded data.

Nodes without a timestamp are **always visible** regardless of the timeline range. Set **Hide undated edges** to `true` to also hide edges that have no timestamp while the timeline is active.

### Filter panel

| Widget property | Recommended value |
|---|---|
| Show filter panel | `true` |
| Filter by type | `true` |
| Filter by group | `true` |
| Filter by score | `true` |

A **⚙ Filter** toggle button appears inline in the search bar row. Clicking it slides in a panel from the left. The panel contains checkbox sections for type, group, and a numeric score range. Each section has **Select All / None** buttons.

### Source tags

| Widget property | Mendix attribute |
|---|---|
| Show source tags | `true` |
| Source attribute | `NetworkNode/SourceSystem` |

Source tags appear as small colored pills rendered directly on the canvas below each node. Each distinct source system name gets a consistent color (one of 8 palette colors, deterministically assigned from the name).

---

## Step 5 — Node Type Styles (Custom or Override)

To override or extend the starter palette, configure the **Node type styles** object list in Studio Pro.

**Starter palette** (used when `useDefaultStyles = true` and no custom entries exist):

| Type name | Shape | Background color | Border color | Size (px) |
|---|---|---|---|---|
| `person` | Circular image | `#E6F1FB` | `#185FA5` | 36 |
| `employee` | Circular image | `#E6F1FB` | `#185FA5` | 32 |
| `organization` | Hexagon | `#EEEDFE` | `#534AB7` | 30 |
| `system` | Square | `#EAF3DE` | `#3B6D11` | 26 |
| `product` | Diamond | `#FAEEDA` | `#854F0B` | 26 |
| `document` | Square | `#F1EFE8` | `#5F5E5A` | 22 |
| `location` | Triangle up | `#EBF7F4` | `#0F7A6B` | 28 |
| `event` | Triangle down | `#FEF3C7` | `#D97706` | 26 |
| `service` | Dot | `#F3F4F6` | `#6B7280` | 22 |

Any node type not listed in either the starter palette or your custom list falls back to a gray dot (size 22). You can add entirely new types (e.g. `vendor`, `device`, `contract`) by adding rows to this list without modifying the widget code.

Set **Use default type styles** to `false` to disable the starter palette — all types then fall back to gray until you add custom entries.

---

## Step 6 — Sample Test Data

Use your preferred method (microflow, import mapping, REST mock, or direct database insert) to seed the following records into your Mendix module. The data below is the org-chart example used throughout this guide.

---

### NetworkNode sample records

| NodeID | Label | NodeType | Score | Group | EventDate | SourceSystem |
|---|---|---|---|---|---|---|
| EMP-001 | Alice Johnson | employee | 85 | Engineering | 2023-03-01 | HR |
| EMP-002 | Bob Smith | employee | 72 | Engineering | 2023-06-15 | HR |
| EMP-003 | Carol White | employee | 60 | Marketing | 2022-11-01 | HR |
| EMP-004 | David Lee | employee | 45 | Finance | 2024-01-10 | HR |
| EMP-005 | Eve Martinez | employee | 90 | Engineering | 2021-08-20 | AD |
| EMP-006 | Frank Chen | employee | 30 | Sales | 2024-03-05 | HR |
| DEP-001 | Engineering | organization | 0 | Division A | 2020-01-01 | ERP |
| DEP-002 | Marketing | organization | 0 | Division B | 2020-01-01 | ERP |
| DEP-003 | Finance | organization | 0 | Division B | 2020-01-01 | ERP |
| DEP-004 | Sales | organization | 0 | Division B | 2020-01-01 | ERP |
| SYS-001 | HR Portal | system | 0 | IT | _(null)_ | AD |
| SYS-002 | ERP Platform | system | 0 | IT | _(null)_ | AD |
| SYS-003 | Analytics Engine | system | 0 | IT | _(null)_ | AD |
| DOC-001 | Q1 Budget Plan | document | 0 | Finance | 2026-01-15 | ERP |
| DOC-002 | Product Roadmap | document | 0 | Engineering | 2026-02-01 | ERP |

---

### NetworkEdge sample records

| FromNodeID | ToNodeID | RelationType | Weight | EdgeDate |
|---|---|---|---|---|
| EMP-002 | EMP-001 | reports to | 3 | 2023-06-15 |
| EMP-004 | EMP-001 | reports to | 3 | 2024-01-10 |
| EMP-006 | EMP-003 | reports to | 2 | 2024-03-05 |
| EMP-001 | DEP-001 | member of | 2 | 2023-03-01 |
| EMP-002 | DEP-001 | member of | 2 | 2023-06-15 |
| EMP-003 | DEP-002 | member of | 2 | 2022-11-01 |
| EMP-004 | DEP-003 | member of | 2 | 2024-01-10 |
| EMP-006 | DEP-004 | member of | 2 | 2024-03-05 |
| EMP-001 | SYS-001 | accesses | 1 | _(null)_ |
| EMP-005 | SYS-002 | administers | 4 | _(null)_ |
| EMP-005 | SYS-003 | administers | 4 | _(null)_ |
| EMP-004 | DOC-001 | owns | 2 | 2026-01-15 |
| EMP-002 | DOC-002 | owns | 2 | 2026-02-01 |

---

### NetworkBadge sample records

| BadgeNodeID | Category | Label |
|---|---|---|
| EMP-001 | approved | Performance review approved |
| EMP-002 | pending | Awaiting 360 review |
| EMP-005 | flagged | Access review required |
| EMP-005 | approved | Security clearance OK |
| SYS-002 | flagged | Patch pending |

> EMP-005 has two badges (flagged + approved). The widget merges them — the node badge shows the highest-priority color (`flagged`) and count `2`. Both entries appear in the side panel badge list.

---

## Step 7 — Feature Configuration Reference

### Feature: Expression Labels

The **Node label** and **Edge label** properties accept Mendix template expressions, not just single attributes. This means you can combine multiple fields into one label:

```
{FirstName} + ' ' + {LastName}
'Flight ' + {FlightNumber} + ' (' + {Origin} + ' → ' + {Destination} + ')'
{ProductCode} + ': ' + {ProductName}
```

Single-attribute bindings like `{Label}` continue to work exactly as before — no migration needed.

---

### Feature: Node Size from Attribute

Map any Integer or Decimal attribute from the node datasource to **Node size**. The per-node value overrides the type config size for that node:

- Map `NetworkNode/Score` → nodes with score 90 appear visually larger than score 10
- Falls back to the type config size when the attribute value is 0 or null

---

### Feature: Score Emphasis (Multi-Tier)

The **Score attribute** + **Score levels** object list drives configurable emphasis borders:

1. Map `NetworkNode/Score` (or equivalent) to **Score attribute**
2. Set **Score label** to a domain-appropriate name (e.g. "Priority", "Rating", "Risk")
3. Add entries to **Score levels**:

| Score label | Min value | Border color |
|---|---|---|
| High | 80 | `#A32D2D` |
| Medium | 40 | `#854F0B` |
| Low | 0 | `#3B6D11` |

**Runtime logic:** Levels are sorted descending by min value. The first level where `score >= min` is applied. If no level matches, no emphasis border is shown (the score is still displayed in the tooltip).

Example configurations:

- **3-tier risk**: `[{min:80,color:"#A32D2D",label:"High"},{min:40,color:"#854F0B",label:"Medium"},{min:0,color:"#3B6D11",label:"Low"}]`
- **Star rating**: `[{min:4,color:"#D97706",label:"★★★★"},{min:2,color:"#888780",label:"★★"}]`
- **Simple flag**: `[{min:1,color:"#A32D2D",label:"Flagged"}]`

---

### Feature: Badge Type Config

The **Badge type config** object list maps each badge category value to a color and priority. The badge with the highest priority determines the circle color when a node has multiple badges. All badges are listed in the tooltip.

**Starter kit — approval workflow:**

| Category value | Badge color | Priority |
|---|---|---|
| `approved` | `#3B6D11` | 1 |
| `pending` | `#854F0B` | 2 |
| `flagged` | `#A32D2D` | 3 |

**Starter kit — deployment status:**

| Category value | Badge color | Priority |
|---|---|---|
| `deployed` | `#3B6D11` | 1 |
| `staging` | `#185FA5` | 2 |
| `failed` | `#A32D2D` | 3 |

**Starter kit — security categories (replicating v1 severity colors):**

| Category value | Badge color | Priority |
|---|---|---|
| `info` | `#185FA5` | 1 |
| `low` | `#3B6D11` | 2 |
| `medium` | `#854F0B` | 3 |
| `high` | `#C05621` | 4 |
| `critical` | `#A32D2D` | 5 |

If no `badgeTypeConfig` entries are configured, all badges render as gray (`#888780`). No crash occurs.

---

### Feature: Edge Type Styles

The **Edge type styles** object list maps each relationship type to a visual style. The style is matched using the **Edge type expression** (case-insensitive).

**Setup:**
1. Set **Edge type expression** to `{RelationType}` (or a computed expression)
2. Add entries to **Edge type styles**

**Example — org chart:**

| Type name | Line color | Width | Dashed | Arrow |
|---|---|---|---|---|
| `reports to` | `#185FA5` | 2 | false | true |
| `manages` | `#185FA5` | 3 | false | true |
| `member of` | `#888780` | 1 | true | false |
| `accesses` | `#854F0B` | 1 | true | true |

**Example — IT topology:**

| Type name | Line color | Width | Dashed | Arrow |
|---|---|---|---|---|
| `depends on` | `#A32D2D` | 2 | false | true |
| `calls` | `#185FA5` | 1 | false | true |
| `hosted by` | `#3B6D11` | 1 | true | false |

When `edgeTypeAttribute` is not set, the widget falls back to using the edge label value for type lookup — so for simple cases you do not need to configure both.

Unmatched edge types use the default edge color and style.

---

### Feature: Custom Side Panel Content

1. Set **Show side panel** → `true`
2. Set **Use custom panel content** → `true`
3. The **Custom panel content** dropzone appears — drop any Mendix widgets (Data view, Text, Image, Microflow button, etc.)
4. The dropzone datasource is the **selected node** (same object as the node datasource). Bind widget properties directly to node attributes.

The header and hero image still render above the custom content. When **Use custom panel content** is `false` the default attribute list / connections tabs are shown — the dropzone is hidden in Studio Pro.

---

### Feature: Hub Clustering

Enable **Enable clustering** in the Performance group. Set **Cluster hub size** to the minimum connection count that should trigger clustering (default 5).

**Behaviour:**
- After data loads, any node with ≥ `clusterHubSize` connections is replaced by a cluster bubble labelled "Cluster"
- Clicking a cluster bubble expands it, revealing the original nodes
- Badge circles and source tags are suppressed on nodes that are collapsed inside a cluster
- Search automatically opens any cluster that contains a matching node

Re-clustering fires whenever `enableClustering`, `clusterHubSize`, or the underlying data changes.

---

### Feature: Max Nodes Limit

Set **Max nodes** to a positive integer (e.g. `200`) to cap the number of rendered nodes. Nodes are sliced in datasource order — refine your datasource XPath/sort to ensure the most relevant nodes are first.

When the limit is exceeded an amber warning bar appears above the canvas:

> **Showing 200 of 847 nodes.** `[Load all 847 nodes (may be slow)]`

Clicking **Load all** overrides the limit for the current browser session without changing the Studio Pro setting. The bar disappears once all nodes are loaded.

Set **Max nodes** to `0` (default) for no limit.

---

### Feature: Side Panel

The side panel is a resizable drawer that opens when a node is clicked. It is a flex sibling of the canvas — the canvas shrinks to make room. Width is draggable between 200 px and 700 px via the 5 px grab handle on the panel's left edge.

**Header — node identity:** The panel always displays the selected node's label as a **bold title** and its type as a **colored pill badge** below it. This gives immediate visual identity without scrolling attribute rows.

**Hero image:** When **Node image URL** is mapped and the node has an image, a full-width image renders at the top of the panel before the header. The close button (✕) floats over the hero image as a translucent circle. When no image is present, the close button appears inline in the header next to the title. The hero height scales with panel width (clamped 120–220 px).

**Badge details in side panel:** When a selected node has badges, a `Badges (N)` section header appears in the details tab below the standard attributes. Each badge row shows:
- A **colored dot** matching the badge category color from `badgeTypeConfig`
- The **category name** as the row label
- The **badge label** (if mapped) and **count** as the value

**Connections tab:** Lists all edges from/to the selected node. Each row shows a direction arrow (blue → outgoing, green ← incoming), the connected node's label, and the relation type. Connection rows highlight on hover.

**Custom content:** When **Use custom panel content** is enabled, the custom dropzone replaces the default attribute list and connections tabs. The header and hero image still render above the custom content.

---

### Feature: Focus Mode

When enabled, clicking a node dims all nodes and edges not connected to it.
- **Level 1** — direct neighbours only
- **Level 2** — neighbours-of-neighbours (2-hop BFS)

Click an empty area of the canvas to reset focus.

---

### Feature: Search

The search bar matches against the configured **Search attribute** (e.g. `Label`). Matching nodes get a gold border highlight; non-matching nodes are dimmed to 12% opacity. The **Auto-focus camera** option fits the view to matching nodes after each keystroke.

---

### Feature: Lazy Expand

With **Append on expand = true** and an **On node expand** action:
1. Double-click a node → widget writes its ID to `SelectedNodeID` and fires the action
2. Your microflow fetches new NetworkNode + NetworkEdge records and commits them
3. The widget detects the datasource refresh and appends only the new items without resetting the layout

---

### Feature: Theme System
- **Light** — default warm off-white canvas (`#FAFAF8`)
- **Dark** — dark canvas (`#1C1C1E`) with adjusted node fonts, edge colors, panel, and search colors
- **System** — follows the OS `prefers-color-scheme` media query
- **Allow user theme toggle** — shows a high-contrast pill button at bottom-left of the canvas

Theme changes update the graph in-place using `DataSet.update()` — no layout reset occurs.

---

### Feature: Timeline Filtering

The timeline bar sits between the topbar (search + filter toggle) and the canvas.

**Date range mode** — two `<input type="date">` controls with 150 ms debounce on change.

**Slider mode** — dual-thumb range slider. `minDate` and `maxDate` are auto-derived from the loaded data timestamps. The selected range is shown as a label next to the slider.

**Edge filtering** — if `edgeTimeAttribute` is mapped, edges are also filtered by their timestamp. Edges without a timestamp:
- `hideEdgesWithoutTimestamp = false` (default) → always visible
- `hideEdgesWithoutTimestamp = true` → hidden while any timeline range is active

Click the **✕** button to clear the timeline filter.

---

### Feature: Filter Panel

The filter panel slides in from the left side of the canvas. The **⚙ Filter** toggle button sits inline in the search bar row; it shows a red badge with the count of active filter dimensions.

**Type filter** — checkboxes for each unique `_type` value in loaded nodes.

**Group filter** — same structure for `_group` values.

**Score filter** — two number inputs (Min / Max, 0–100 range). The section label uses the **Score label** property value (e.g. "Priority", "Rating", "Risk").

**Clear all** — button in the panel footer resets all three dimensions simultaneously.

---

### Feature: RTL Layout

Enable **Enable RTL layout** (`enableRTL = true`) in **Visualization → Theme** for Arabic or Hebrew deployments.

**What flips:**

| Element | LTR | RTL |
|---|---|---|
| Side panel | Right side of canvas | Left side of canvas |
| Filter panel | Slides in from left | Slides in from right |
| Theme toggle | Bottom-left | Bottom-right |
| Panel header | Title left, close right | Title right, close left |
| Resize handle | Left edge of panel | Right edge of panel |
| Connection arrows | `→` / `←` kept LTR | BiDi wrapping prevented via `dir="ltr"` |

**Canvas labels:** Vis.js renders node and edge labels center-aligned — RTL mode has no effect on canvas text and no extra config is required.

**Drag handle in RTL:** Dragging the resize handle rightward widens the panel; leftward narrows it (reversed from LTR).

---

### Feature: Audit Trail Hook

Add two optional properties to your **GraphContext** entity:
- `SelectedNodeID` (String) — already required for on-click actions
- `NodeViewedAt` (DateTime) — new, optional

**Widget configuration:**

| Property | Value |
|---|---|
| On node view | `ACT_Graph_AuditNodeView` |
| Node viewed at | `GraphContext/NodeViewedAt` |

**Timing guarantee:** The widget writes `NodeViewedAt = new Date()` BEFORE firing `onNodeView`. Your microflow can safely read `$Context/NodeViewedAt` and `$Context/SelectedNodeID` to create the audit record.

**When it fires:** Every single click AND every double-click (expand). It is separate from `onNodeClick` (which is for navigation) — you can wire both independently.

**Example microflow — ACT_Graph_AuditNodeView:**
```
Input: $Context (GraphContext)
1. Create AuditLog
   - NodeID    = $Context/SelectedNodeID
   - ViewedAt  = $Context/NodeViewedAt
   - UserName  = $currentUser/Name
2. Commit AuditLog (without refresh)
```

---

### Feature: Badge Datasource Failure Warning

When the badge datasource is unavailable (REST timeout, bad XPath, permissions error), the graph renders normally without badges. A blue dismissible banner appears above the canvas:

> **Badge data source unavailable — graph is shown without badges.** `[✕]`

- Clicking ✕ dismisses the banner for the current session
- The banner reappears automatically if the datasource recovers and then fails again
- This is automatic — no configuration required. It activates whenever `badgeDataSource` is configured and its status becomes "unavailable"

---

### Feature: Source Tags

Source tags are drawn on the vis-network canvas as small rounded pill shapes directly below each node.

- Each unique source system name is assigned one of 8 palette colors via a deterministic hash
- Colors are consistent across sessions and do not depend on load order
- Tag text is truncated to 12 characters with an ellipsis if the source name is longer
- Tags are culled to the current viewport — off-screen nodes are skipped for performance

**Palette colors:** `#185FA5` · `#3B6D11` · `#854F0B` · `#534AB7` · `#A32D2D` · `#0F7A6B` · `#7A4B0F` · `#2E6B8A`

---

### Feature: Export

Enable **Enable export** (`enableExport = true`) in **Advanced → Export**. An **⬇ Export** button appears in the topbar alongside Search and Filter.

> **⚠ CORS requirement for image-bearing nodes:** Images displayed on nodes via `nodeImageUrl` must be served with `Access-Control-Allow-Origin: *` (or the origin of your Mendix app). The widget fetches images with `crossOrigin = "anonymous"`. If the image server does not send the correct CORS header, the canvas becomes "tainted" and `toDataURL()` will throw a SecurityError — export will fail with an alert.
>
> **Government/internal servers:** If your asset server is on a locked-down internal network and cannot be configured for CORS, implement server-side export as a Mendix Java action:
> 1. Read the same `NetworkNode` datasource in a microflow
> 2. Fetch node images server-side using `com.mendix.core.Core.retrieveFileDocumentContent()` (or an HTTP call)
> 3. Use **Apache Commons Imaging** (PNG) or **iText** (PDF) to compose the graph layout programmatically
> 4. Return the file to the user via `Core.storeFileDocumentContent()` + a download link
>
> This approach bypasses the browser canvas entirely and is CORS-independent.

Clicking the button opens an inline panel with three options:

| Option | Default | Notes |
|---|---|---|
| Format | PNG | PNG downloads directly via `<a download>`. PDF opens a print dialog. |
| Include background | On | Fills the export canvas with the widget background color. Useful for dark mode. |
| Include labels | On | When off, node and edge labels are temporarily hidden during export then restored. |

**Security:** The browser's native right-click → "Save image as" is always suppressed on the canvas, regardless of whether `enableExport` is on. This prevents uncontrolled exports and is always active.

**CORS requirement:** Images shown on nodes via `nodeImageUrl` must be served with `Access-Control-Allow-Origin: *` (or the origin of your Mendix app). The widget fetches images with `crossOrigin = "anonymous"`. If the image server does not send the correct CORS header the canvas is tainted and `toDataURL()` will throw a SecurityError — export will fail and show an alert.

---

### Feature: Investigation Mode

Enable **Allow investigation mode** (`allowInvestigationMode = true`) in **Advanced → Investigation Mode**. A `◎ Investigate` toggle button appears in the topbar alongside Search, Filter, and Export.

**What activates when the user clicks Investigate:**

| Behavior | Detail |
|---|---|
| Focus mode on | Same depth as the **Focus depth** setting. Clicking a node dims all unrelated nodes. |
| Filter panel opens | Only if **Show filter panel** is enabled. Opens automatically on activation. |
| Alert highlighting | Every node with at least one badge gets its border set to the badge's worst-severity color at width 4. |
| Softer dim for alert nodes | Badge nodes outside the focus neighborhood dim to **45% opacity** instead of 12% — they stay discoverable while clearly outside the current focus. |
| Amber banner | `◎ Investigation Mode · N alerts` appears below the topbar with a `✕` exit button. |

**Deactivating:**
- Click `◎ Investigate` again, or click `✕` in the banner
- The focused node is cleared automatically on exit
- All visual overrides (alert borders, softer dim) are removed immediately

**Driving from the page (Mendix attribute):**

| Property | Value |
|---|---|
| Investigation mode (attribute) | `GraphContext/InInvestigationMode` |
| On investigation toggle | `ACT_Graph_OnInvestigationToggle` |

The widget reads `InInvestigationMode` on every render and enters investigation mode if the attribute is `true`. When the user toggles via the button, the widget writes back to `InInvestigationMode` and fires `onInvestigationToggle`. This lets a Mendix page:
- Enter investigation mode programmatically (e.g. when an alert count exceeds a threshold)
- Exit it via a page button outside the widget
- Log the session start/end time in `onInvestigationToggle`

**Example microflow — ACT_Graph_OnInvestigationToggle:**
```
Input: $Context (GraphContext)

1. If $Context/InInvestigationMode = true:
   - Create InvestigationSession
     - StartedAt = [%CurrentDateTime%]
     - UserName  = $currentUser/Name
   - Commit InvestigationSession
2. Else:
   - Retrieve open InvestigationSession for current user
   - Set EndedAt = [%CurrentDateTime%]
   - Commit
```

---

### Feature: Metrics Bar

Enable **Show metrics bar** (`enableMetricsBar = true`) in **Advanced → Metrics bar**. A slim status bar appears below the graph canvas.

```
15 nodes · 13 edges  │  Visible: 8 of 15  │  Priority: 6  │  Badges: 4
```

Each segment is independent and only shown when it carries information:

| Segment | Condition |
|---|---|
| `N nodes · E edges` | Always shown when data is loaded |
| `Visible: X of N` | Shown (and highlighted in blue) only when focus / search / timeline / filter is active |
| `{scoreLabel}: X` | Shown only when `nodeScoreAttribute` + at least one `scoreLevels` entry are configured. `scoreLabel` is used as the prefix (e.g. "Priority", "Risk") |
| `Badges: X` | Shown only when `badgeDataSource` is configured and at least one node has a badge |

The visible count uses the same intersection logic as the graph overlay — if focus and filter are both active, the count reflects nodes passing **both**.

---

## Step 8 — Deploy the Widget

1. Copy `dist/2.0.0/mendix.NetworkGraph.mpk` to your Mendix project's `widgets/` folder.
2. In Studio Pro: **App → Synchronize App Directory**.
3. The **Network Graph** widget appears in the toolbox.
4. Run locally and open your graph page.

---

## Overlay Architecture Notes (for developers)

The widget uses a **unified intersection model** for all overlay features:

```
applyOverlay() runs whenever any of the following change:
  - focusedNodeId          (focus mode)
  - searchMatchIds         (search)
  - timelineVisibleIds     (timeline nodes)
  - timelineVisibleEdgeIds (timeline edges)
  - filterVisibleIds       (filter panel)

Algorithm:
  visibleSets = [] of Set<nodeId>
  For each active feature → push its Set into visibleSets
  visibleNodes = INTERSECTION of all sets
  dimmedNodes  = ALL node IDs − visibleNodes

Node update:
  - opacity 0.12 if dimmed (0.45 if dimmed AND has badge alert in investigation mode)
  - gold border (#F59E0B) if search-highlighted and not dimmed
  - badge worstColor border + lineWidth 4 if has alert and investigation mode is on
Edge update: dim if either endpoint is dimmed OR edge is outside timeline range
```

`baseNodesRef` / `baseEdgesRef` store clean node/edge objects (with all `_*` metadata fields intact). `DataSet.update()` is used for all overlay changes — `setData()` is only called on actual data changes, which resets the physics layout. This separation is what keeps the layout stable across theme changes, filter changes, and search queries.

---

## Upgrade Guide — v1 to v2

| v1 property | v2 property |
|---|---|
| `nodeRiskScore` | `nodeScoreAttribute` |
| `alertDataSource` | `badgeDataSource` |
| `alertNodeId` | `badgeNodeId` |
| `alertSeverity` | `badgeCategory` |
| `alertType` | `badgeLabel` |
| `filterByRisk` | `filterByScore` |
| (hardcoded severity colors) | `badgeTypeConfig` — add one row per category |
| (hardcoded investigation node types) | `nodeTypeConfig` — add rows for your domain; or enable `useDefaultStyles` |
