---
title: Declarative Layout & Styling
---

# Declarative Layout & Styling

ADAC lets you describe **what an architecture diagram should look like structurally**, while the layout engine works out the coordinates. Nothing here is an absolute position, and nothing is an engine-specific option: ADAC properties are translated into layout constraints by the diagram tooling.

:::info In this concept guide
**Time to read:** 10 minutes
**Goal:** Control how containers arrange their children, how connections leave and enter nodes, and how diagrams are styled, without ever specifying pixel positions.
:::

```yaml
layoutOptions:
  direction: LR # overall flow
  gap: 1
  containerPadding: 1

# on an element:
visual:
  layout:
    direction: TB # how *this* element arranges its direct children
    maxItemsPerLine: 2
    gap: 0.5
  style:
    background: true
```

## The Hierarchical Model

Layout is **hierarchical and composable**: each container controls the layout of its _direct children only_.

```text
VPC                    ← its layout places the subnets
├── Public Subnet      ← its layout places ALB, NAT
└── Private Subnet     ← its layout places ECS, DB
    ├── ECS
    └── DB
```

Properties are resolved in this order:

```text
global layoutOptions  →  nearest layout-declaring ancestor  →  element's own layout
```

## Units

Every length is a **logical unit** resolved through `layoutOptions.unitSize` (default `100`). With `unitSize: 100`, `gap: 2` is 200 px. The conversion applies to `gap`, `padding`, `edgeSpacing`, `size`, `minSize` and `maxSize`; units and pixels are never mixed.

## Global Options: `layoutOptions`

`layoutOptions` is an optional root-level object that sets diagram-wide values. Options you leave out may be filled from the [default profile](#default-profile-elk).

| Property           | Values                                                | Description                                                                                                                                |
| ------------------ | ----------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| `engine`           | `elk`, `custom`, `orthogonal`, `tsm`                  | Layout engine. A CLI/API override still wins; ELK is used when neither this nor the top-level `layout` is set.                             |
| `defaults`         | `v1` (default), `none`                                | Default profile for options you leave out, on `elk` only. See [Default Profile](#default-profile-elk).                                     |
| `unitSize`         | number > 0                                            | Pixels per logical unit.                                                                                                                   |
| `direction`        | `LR` `RL` `TB` `BT`                                   | Flow of the whole diagram; default for nested containers.                                                                                  |
| `gap`              | number \| `{horizontal, vertical}`                    | Space between siblings.                                                                                                                    |
| `containerPadding` | number \| `{top,right,bottom,left}`                   | Default container padding.                                                                                                                 |
| `align`            | `start` `center` `end`                                | Default cross-axis alignment for arranged containers.                                                                                      |
| `justify`          | `start` `center` `end` `space-between` `space-around` | Default main-axis distribution for arranged containers.                                                                                    |
| `edgeSpacing`      | number ≥ 0                                            | Edge-to-node clearance (edge-to-edge is half).                                                                                             |
| `edgeRouting`      | `orthogonal` `polyline` `splines` `straight`          | Default connection routing.                                                                                                                |
| `background`       | boolean (default `false`)                             | Default node/card background.                                                                                                              |
| `labelMaxChars`    | integer 4–80                                          | Most characters on one line of a node label; longer labels wrap, then end in `…`. Can be set per element via `visual.style.labelMaxChars`. |
| `edgeLabels`       | boolean (default `true`)                              | Draw each connection `type` on its edge.                                                                                                   |

## Default Profile (elk)

On `elk`, `layoutOptions` you leave out are filled from a small, trialled profile, so files written before declarative layout existed get a consistent baseline without being edited. Your own values always win, at the global and the element level. Defaults are never written into the document: validation and the [engine warning](#engine-support) see only what you declared.

Set `layoutOptions.defaults: none` to keep the engine's own behaviour.

Profile `v1`:

| Option             | Default                                          | Effect                                                                               |
| ------------------ | ------------------------------------------------ | ------------------------------------------------------------------------------------ |
| `direction`        | `LR`                                             | The flow the builder already used, now stated explicitly.                            |
| `containerPadding` | `{top: 0.6, right: 0.3, bottom: 0.3, left: 0.3}` | At `unitSize: 100`: keeps the 60px title band and trims the sides from 40px to 30px. |

Padding is in logical units, so it follows `unitSize`. A defaulted top padding is never below 60px, so container titles keep their band: at `unitSize: 50` the top stays 60px and the sides become 15px; at `unitSize: 200` the top is 120px and the sides 60px, so containers grow instead of becoming more compact. The 60px floor applies only to the default; a `containerPadding` you declare is used as written. Declare it, or set `defaults: none`, if you use another `unitSize`.

The generation log records one line (`Layout defaults applied: …`) naming the defaults that were used.

## Per-Element Layout: `visual.layout`

Any service (or other element) can carry a `visual.layout` block describing how it arranges its **direct children** and where it sits among its siblings.

| Property          | Values                                                | Inheritance         | Applies to                |
| ----------------- | ----------------------------------------------------- | ------------------- | ------------------------- |
| `direction`       | `LR` `RL` `TB` `BT`                                   | **inherited**       | children                  |
| `gap`             | number \| `{horizontal, vertical}`                    | **inherited**       | children (per-axis merge) |
| `align`           | `start` `center` `end`                                | **inherited**       | children, cross axis      |
| `justify`         | `start` `center` `end` `space-between` `space-around` | **inherited**       | children, main axis       |
| `padding`         | number \| `{top,right,bottom,left}`                   | local               | this container            |
| `wrap`            | boolean                                               | local               | children                  |
| `itemsPerLine`    | integer ≥ 1 (exact)                                   | **never inherited** | children                  |
| `maxItemsPerLine` | integer ≥ 1 (maximum, lines balanced)                 | **never inherited** | children                  |
| `size`            | `{width, height}` number \| `auto`                    | local               | this element              |
| `minSize`         | `{width, height}` number \| `auto`                    | local               | this element              |
| `maxSize`         | `{width, height}` number \| `auto`                    | local               | this element              |
| `order`           | integer                                               | placement           | position among siblings   |
| `position`        | `top-left` … `bottom-right`, `center`                 | **never inherited** | region in the parent      |

`order` and `position` describe the element itself within its parent, so they are never inherited either.

### Arrangement

A container is **arranged** once it (or its nearest layout-declaring ancestor) declares `direction`, `wrap`, `itemsPerLine`, `maxItemsPerLine`, `align` or `justify`. Declaring `order` on a child also arranges the parent. Arranged containers lay their children out in a line along `direction`, wrapping when a line limit is set. Containers that declare nothing keep the engine's own flow. `gap`, `padding` and sizes never arrange anything on their own.

- `itemsPerLine: 2` puts exactly two items on each line.
- `maxItemsPerLine: 4` uses the fewest lines and balances them: five items give `3 + 2`, not `4 + 1`.
- `wrap: false` together with a line limit is invalid.
- `RL` / `BT` reverse the sequence along the axis.
- `position` in an arranged container moves the element to the start or end of its line; elsewhere it pins the element to the first/last layer of the flow.

### Sizing

`size`, `minSize` and `maxSize` are constraints, not coordinates. A container can never be smaller than its content, so `size` and `minSize` act as a floor. `maxSize` is applied to leaf elements; for containers the engine cannot enforce a ceiling, so a warning is logged when the content does not fit.

## Connection Layout: `connections[].layout`

Connections can control which side of a node they leave and enter, and how they are routed.

```yaml
connections:
  - from: app
    to: database
    bidirectional: true
    layout:
      fromSide: right # left | right | top | bottom | auto
      toSide: left
      routing: orthogonal # orthogonal | polyline | splines | straight
```

- Sides become fixed ports; `auto` leaves the choice to the engine.
- `bidirectional: true` draws arrowheads at both ends.
- Two connections between the same nodes in opposite directions are nudged apart so they never overlap.
- A per-connection `routing` of `straight` or `splines` is drawn by the renderer; the ELK engine itself routes the whole diagram with `edgeRouting`.

## Styling: `visual.style`

`visual.style.background` overrides `layoutOptions.background` for a single element. With the default (`false`) no card or icon pill is drawn behind an icon; `true` restores the classic white card. `visual.style.labelMaxChars` overrides the global label width for one element.

## Complete Example

A VPC that stacks a public and a private subnet; the private subnet wraps its services into rows of three.

```yaml title="declarative-layout.adac.yaml"
version: "0.2"

metadata:
  name: "Declarative layout example"
  created: "2026-01-01"

layoutOptions:
  engine: elk
  unitSize: 100
  direction: LR # overall flow: left to right
  gap: 0.6
  containerPadding: 0.5
  edgeSpacing: 0.4
  edgeRouting: orthogonal
  background: false # no card behind icons unless an element asks for one

infrastructure:
  clouds:
    - id: "aws-main"
      provider: "aws"
      region: "us-east-1"
      services:
        - id: "vpc"
          service: "vpc"
          name: "Production VPC"
          visual:
            layout:
              direction: TB # subnets are stacked
              gap: 0.8
              minSize: { width: 6 }

        - id: "public-subnet"
          service: "subnet"
          name: "Public Subnet"
          configuration: { vpc: "vpc" }
          visual:
            layout:
              direction: LR
              align: center
              order: 1 # sits above the private subnet

        - id: "private-subnet"
          service: "subnet"
          name: "Private Subnet"
          configuration: { vpc: "vpc" }
          visual:
            layout:
              direction: LR
              itemsPerLine: 3 # wrap after three services
              justify: center
              order: 2

        - id: "api"
          service: "lambda"
          name: "API"
          configuration: { subnets: ["private-subnet"] }
          visual:
            layout: { order: 1 }

        - id: "aurora"
          service: "rds-aurora-postgres"
          name: "Aurora"
          configuration: { subnets: ["private-subnet"] }
          visual:
            layout: { order: 4 }
            style:
              background: true # this one keeps the classic card

connections:
  - id: "api-to-db"
    from: "api"
    to: "aurora"
    type: "database-query"
    bidirectional: true
    layout:
      fromSide: right
      toSide: left
```

## Deprecated Aliases

<div className="fill-table">

| Deprecated | Used instead  |
| ---------- | ------------- |
| `unit`     | `unitSize`    |
| `routing`  | `edgeRouting` |

</div>

Both still work. Validators report them as warnings, and the diagram log repeats them. When both names are set, the new one wins.

## How It Maps to the Engine

ELK ignores per-container `direction` when connections cross container boundaries, so the ELK adapter realises arrangements with structure it _does_ honour:

- a row along the flow is a chain of invisible, layout-only edges;
- wrapped lines are invisible groups (type `layout-line`), stacked or chained;
- `align`, `justify`, declared order within a stacked layer and `maxSize` checks run as a post-layout pass that moves nodes and re-attaches their edges;
- connections that run _between lines_ of one container are withheld from layering (otherwise ELK staggers the lines) and routed afterwards.

Layout-only edges (ids starting `__adac_layout__`) and line groups are never drawn. The `custom`, `orthogonal` and `tsm` engines receive the plain hierarchy and ignore arrangement properties and the connection-level `bidirectional` and `routing` properties; `background`, `labelMaxChars` and `edgeLabels` still apply. See [Engine Support](#engine-support) for the full matrix and the warning emitted when these properties are declared with a non-ELK engine.

## Engine Support

Only the `elk` engine applies the full set of declarative properties. The other engines lay out the plain hierarchy.

| Property                                                                                                                             | `elk` | `custom` / `orthogonal` / `tsm` |
| ------------------------------------------------------------------------------------------------------------------------------------ | :---: | :-----------------------------: |
| `layoutOptions`: `direction`, `gap`, `containerPadding`, `align`, `justify`, `edgeSpacing`, `edgeRouting` (and deprecated `routing`) |  yes  |               no                |
| `visual.layout`: `direction`, `gap`, `padding`, `align`, `justify`, `wrap`, `itemsPerLine`, `maxItemsPerLine`, `order`, `position`   |  yes  |               no                |
| `connections[].layout`: `fromSide`, `toSide`, `routing`; `bidirectional`                                                             |  yes  |               no                |
| `visual.layout`: `size`, `minSize`, `maxSize` on **containers**                                                                      |  yes  |               no                |
| `visual.layout`: `size`, `minSize`, `maxSize` on leaf elements                                                                       |  yes  |               yes               |
| `layoutOptions`: `unitSize`, `background`, `edgeLabels`, `labelMaxChars`                                                             |  yes  |               yes               |

A **container** here is a service that holds others: another service names it as its `vpc`, `vnet`, `subnet` or `subnets` (in `configuration` or on the service), or it `runs` applications. Only for these are `size`, `minSize` and `maxSize` counted as ignored; on any other element every engine applies them.

### Unsupported-engine warning

When a file **declares** any property marked "no" and the engine in effect is not `elk`, a warning is emitted once per file. The engine in effect is, in order: the `--layout` flag, `layoutOptions.engine`, the top-level `layout`, then `elk`. The warning names the engine and lists the ignored property groups:

```text
Warning: Layout engine "custom" ignores declarative layout properties:
layoutOptions (direction, gap); visual.layout (align, order). The diagram will
not follow the declared arrangement. Use --layout elk or set
layoutOptions.engine: elk.
```

- It is a **warning, not an error**. The file is still valid and a diagram is still produced, just without the declared arrangement.
- Only properties the author wrote count. Files that declare none of them, or only the shared properties, never warn. Defaults never trigger it.
- `adac validate` and `adac diagram` print the same warning, and it appears in the generation log (`--validate`, API consumers). With `--validate`, it is reported once.
- The web viewer shows it as a dismissible banner.
- `adac diagram --help` prints the engine support matrix.

## Known Limitations

- `order` among the root's own children is best-effort; use `position` or connections to steer the top level.
- Real connections between items of an arranged container follow the arranged structure rather than ELK's layering, so such edges are routed with a single orthogonal bend and may cross other items.
- `labelPosition`, `labelSpacing`, `avoidOverlap`, `laneSpacing` and other advanced options are not implemented yet.

## Next Steps

- [Connections](./connections) for the full connection object.
- [Infrastructure](./infrastructure) for services and clouds.
- [Full Schema Specification](../reference/schema) for the root object.
