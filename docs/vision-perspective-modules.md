# Vision and Perspective Modules

This guide explains how to design and build **Vision** and **Perspective** component modules using `terminui` as a foundation.

## Concepts

### Vision

A **Vision** is a self-contained, reusable module that renders one focused area of your application. It owns its own data, state, and layout, and exposes a single render function that terminui can call inside a `terminalDraw` or `terminalDrawJsx` loop.

Think of a Vision as a bounded viewport into one domain of information:

- a CPU/memory stats panel
- a log tail viewer
- a list of items with selection state
- a progress dashboard

A well-designed Vision:
- accepts its configuration at creation time
- holds any mutable state internally (e.g., selected index, scroll offset)
- exposes a `render(frame, area)` function (or a JSX element) that the host can call
- has no side-effects outside its own area

### Perspective

A **Perspective** is a higher-order module that **orchestrates multiple Visions**. It controls:

- which Visions are currently active or visible
- how the available screen area is divided among them
- switching between different layout modes (tabs, split panes, full-screen)
- shared state that crosses Vision boundaries (e.g., the active tab index)

A Perspective is the "shell" of your application: it creates the outer layout and delegates rendering to the Visions that fill each region.

---

## Building a Vision Module

### 1. Stateless Vision (functional API)

The simplest Vision is a pure function that returns a `WidgetRenderer`. It receives configuration at creation time and closes over any static data.

```typescript
import {
  blockBordered,
  createTitle,
  createParagraph,
  renderParagraph,
  frameRenderWidget,
  createStyle,
  Color,
} from 'terminui';
import type { Frame, Rect } from 'terminui';

interface StatusVisionConfig {
  readonly title: string;
  readonly message: string;
  readonly color?: Color;
}

const createStatusVision = (config: StatusVisionConfig) => {
  const block = blockBordered({
    titles: [createTitle(config.title)],
  });

  const paragraph = createParagraph(config.message, {
    block,
    alignment: 'center',
  });

  const renderer = renderParagraph(paragraph);

  return {
    render(frame: Frame, area: Rect): void {
      frameRenderWidget(frame, renderer, area);
    },
  };
};

// Usage
const statusVision = createStatusVision({
  title: 'Status',
  message: 'All systems nominal.',
  color: Color.Green,
});
```

### 2. Stateful Vision (functional API)

When a Vision needs to track selection, scroll position, or any other user-driven state, keep that state in a plain object and pass it along with the renderer.

```typescript
import {
  blockBordered,
  createTitle,
  createList,
  createListState,
  renderStatefulList,
  styleFg,
  createStyle,
  Color,
  frameRenderStatefulWidget,
} from 'terminui';
import type { Frame, Rect, ListState } from 'terminui';

interface MenuVisionConfig {
  readonly title: string;
  readonly items: readonly string[];
}

interface MenuVision {
  readonly state: ListState;
  render(frame: Frame, area: Rect): void;
  selectNext(): void;
  selectPrev(): void;
}

const createMenuVision = (config: MenuVisionConfig): MenuVision => {
  const list = createList(config.items, {
    block: blockBordered({ titles: [createTitle(config.title)] }),
    highlightStyle: styleFg(createStyle(), Color.Yellow),
    highlightSymbol: '▶ ',
  });

  const state = createListState(0);
  const renderer = renderStatefulList(list);

  return {
    state,

    render(frame: Frame, area: Rect): void {
      frameRenderStatefulWidget(frame, renderer, area, state);
    },

    selectNext(): void {
      if (state.selected !== undefined) {
        state.selected = Math.min(state.selected + 1, config.items.length - 1);
      }
    },

    selectPrev(): void {
      if (state.selected !== undefined) {
        state.selected = Math.max(state.selected - 1, 0);
      }
    },
  };
};
```

### 3. Vision as a JSX component

JSX Visions are functions (or objects) that return a JSX element. They compose naturally inside Perspective layouts.

```tsx
/** @jsxRuntime automatic */
/** @jsxImportSource terminui */
import { Panel, Label, Gauge, List } from 'terminui/jsx';
import { Color, createListState } from 'terminui';

interface MetricsVisionProps {
  readonly title: string;
  readonly cpuPercent: number;
  readonly memPercent: number;
  readonly fg?: Color;
}

const MetricsVision = (props: MetricsVisionProps) => (
  <Panel title={props.title} p={1} fg={props.fg}>
    <Label text={`CPU  ${props.cpuPercent}%`} bold />
    <Gauge percent={props.cpuPercent} />
    <Label text={`Mem  ${props.memPercent}%`} bold />
    <Gauge percent={props.memPercent} />
  </Panel>
);

interface NavigationVisionProps {
  readonly items: readonly string[];
  readonly selected: number;
}

const NavigationVision = ({ items, selected }: NavigationVisionProps) => (
  <Panel title="Navigation" p={1} fg={Color.LightCyan}>
    <List items={items} state={createListState(selected)} highlightSymbol="▶ " />
  </Panel>
);

// Usage inside a Perspective:
// <MetricsVision title="System" cpuPercent={42} memPercent={68} fg={Color.Cyan} />
// <NavigationVision items={['Overview', 'Metrics']} selected={0} />
```

---

## Building a Perspective Module

A Perspective wires Visions together with a layout and manages switching between views.

### 1. Fixed-split Perspective (functional API)

The simplest Perspective splits the frame area into fixed regions and delegates to a Vision per region.

```typescript
import {
  createLayout,
  lengthConstraint,
  fillConstraint,
  splitLayout,
  frameRenderWidget,
  renderBlock,
  blockBordered,
  createTitle,
} from 'terminui';
import type { Frame } from 'terminui';

// Assume createMenuVision and createStatusVision are defined as above

const createDashboardPerspective = () => {
  const menuVision = createMenuVision({
    title: 'Navigation',
    items: ['Overview', 'Metrics', 'Logs', 'Settings'],
  });

  const statusVision = createStatusVision({
    title: 'Status',
    message: 'All systems nominal.',
  });

  // Vertical layout: 3-row header, fill body, 3-row footer
  const outerLayout = createLayout([
    lengthConstraint(3),
    fillConstraint(1),
    lengthConstraint(3),
  ]);

  // Horizontal body split: 24-col sidebar, fill content
  const bodyLayout = createLayout(
    [lengthConstraint(24), fillConstraint(1)],
    { direction: 'horizontal' },
  );

  return {
    menuVision,
    statusVision,

    render(frame: Frame): void {
      const [headerArea, bodyArea, footerArea] = splitLayout(outerLayout, frame.area);
      const [sidebarArea, contentArea] = splitLayout(bodyLayout, bodyArea);

      // Header
      frameRenderWidget(
        frame,
        renderBlock(blockBordered({ titles: [createTitle('My App', { alignment: 'center' })] })),
        headerArea,
      );

      // Sidebar vision
      menuVision.render(frame, sidebarArea);

      // Content area (placeholder — swap for any Vision)
      statusVision.render(frame, contentArea);

      // Footer
      frameRenderWidget(
        frame,
        renderBlock(blockBordered({ titles: [createTitle('q quit  ↑↓ navigate', { alignment: 'center' })] })),
        footerArea,
      );
    },
  };
};
```

### 2. Tabbed Perspective (functional API)

A tabbed Perspective keeps an array of Visions and an active index, rendering the selected Vision in the main body area.

`TabsConfig.selected` is immutable, so recreate the tabs widget on every render pass with the current `activeIndex` — widget construction is cheap and happens on the fast path anyway.

```typescript
import {
  createLayout,
  lengthConstraint,
  fillConstraint,
  splitLayout,
  frameRenderWidget,
  createTabs,
  renderTabs,
  styleFg,
  styleAddModifier,
  createStyle,
  Color,
  Modifier,
} from 'terminui';
import type { Frame, Rect } from 'terminui';

interface TabEntry {
  readonly label: string;
  render(frame: Frame, area: Rect): void;
}

interface TabbedPerspective {
  readonly activeIndex: number;
  nextTab(): void;
  prevTab(): void;
  render(frame: Frame): void;
}

const createTabbedPerspective = (tabs: readonly TabEntry[]): TabbedPerspective => {
  let activeIndex = 0;

  const tabsLayout = createLayout([lengthConstraint(3), fillConstraint(1)]);

  const highlightStyle = styleAddModifier(styleFg(createStyle(), Color.Yellow), Modifier.BOLD);

  const perspective: TabbedPerspective = {
    get activeIndex() {
      return activeIndex;
    },

    nextTab(): void {
      activeIndex = (activeIndex + 1) % tabs.length;
    },

    prevTab(): void {
      activeIndex = (activeIndex - 1 + tabs.length) % tabs.length;
    },

    render(frame: Frame): void {
      const [tabBarArea, bodyArea] = splitLayout(tabsLayout, frame.area);

      // Recreate the tabs widget each frame with the current selection.
      // TabsConfig is a plain object and construction is O(n) in the number
      // of tab labels — perfectly acceptable on the render hot path.
      const tabsWidget = createTabs(
        tabs.map((t) => t.label),
        { selected: activeIndex, highlightStyle },
      );
      frameRenderWidget(frame, renderTabs(tabsWidget), tabBarArea);

      // Delegate to the active Vision
      const activeTab = tabs[activeIndex];
      if (activeTab !== undefined) {
        activeTab.render(frame, bodyArea);
      }
    },
  };

  return perspective;
};

// Wire it up:
// const perspective = createTabbedPerspective([
//   { label: 'Overview', render: overviewVision.render.bind(overviewVision) },
//   { label: 'Metrics',  render: metricsVision.render.bind(metricsVision)  },
//   { label: 'Logs',     render: logsVision.render.bind(logsVision)        },
// ]);
```

### 3. Perspective with JSX (JSX API)

In the JSX API a Perspective is simply a component that owns layout state and delegates to Vision components. Use a `terminalLoopJsx` callback to re-render on every tick.

```tsx
/** @jsxRuntime automatic */
/** @jsxImportSource terminui */
import {
  createTestBackendState,
  createTestBackend,
  createTerminal,
  fillConstraint,
  lengthConstraint,
  Color,
  createListState,
} from 'terminui';
import { Column, Row, Panel, Label, Gauge, List, terminalDrawJsx } from 'terminui/jsx';

// ── Vision components ──────────────────────────────────────────────────────────

interface HeaderVisionProps {
  readonly appName: string;
  readonly activeTab: string;
}

const HeaderVision = ({ appName, activeTab }: HeaderVisionProps) => (
  <Panel p={1} fg={Color.Cyan}>
    <Row constraints={[fillConstraint(1), lengthConstraint(24)]}>
      <Label text={appName} bold />
      <Label text={`tab: ${activeTab}`} align="right" fg={Color.LightBlue} />
    </Row>
  </Panel>
);

interface MetricsVisionJsxProps {
  readonly cpu: number;
  readonly mem: number;
}

const MetricsVisionJsx = ({ cpu, mem }: MetricsVisionJsxProps) => (
  <Panel title="Metrics" p={1}>
    <Label text={`CPU ${cpu}%`} fg={Color.Green} bold />
    <Gauge percent={cpu} />
    <Label text={`Mem ${mem}%`} fg={Color.Yellow} bold />
    <Gauge percent={mem} />
  </Panel>
);

interface NavigationVisionProps {
  readonly items: readonly string[];
  readonly selected: number;
}

const NavigationVision = ({ items, selected }: NavigationVisionProps) => (
  <Panel title="Navigation" p={1} fg={Color.LightCyan}>
    <List items={items} state={createListState(selected)} highlightSymbol="▶ " />
  </Panel>
);

// ── Perspective ───────────────────────────────────────────────────────────────

interface AppState {
  readonly selectedNav: number;
  readonly cpu: number;
  readonly mem: number;
}

const DashboardPerspective = ({ state }: { readonly state: AppState }) => (
  <Column constraints={[lengthConstraint(3), fillConstraint(1)]}>
    <HeaderVision
      appName="My App"
      activeTab={['Overview', 'Metrics', 'Logs'][state.selectedNav] ?? 'Overview'}
    />
    <Row constraints={[lengthConstraint(20), fillConstraint(1)]} gap={1}>
      <NavigationVision
        items={['Overview', 'Metrics', 'Logs']}
        selected={state.selectedNav}
      />
      <MetricsVisionJsx cpu={state.cpu} mem={state.mem} />
    </Row>
  </Column>
);

// ── Render ────────────────────────────────────────────────────────────────────

const backendState = createTestBackendState(80, 20);
const terminal = createTerminal(createTestBackend(backendState));

const appState: AppState = { selectedNav: 1, cpu: 42, mem: 68 };

terminalDrawJsx(terminal, <DashboardPerspective state={appState} />);
```

---

## Patterns and Best Practices

### Keep Visions self-contained

Each Vision should only write to the `Rect` it is given. Never reach outside the assigned area—terminui's double-buffer diff renderer handles the rest.

```typescript
// ✅ correct — writes only inside `area`
const render = (area: Rect, buf: Buffer): void => {
  renderParagraph(paragraph)(area, buf);
};

// ❌ avoid — touching cells outside `area` corrupts the frame
const render = (area: Rect, buf: Buffer): void => {
  bufferSetString(buf, 0, 0, 'absolute position', createStyle());
};
```

### Use `frameRenderWidget` / `frameRenderStatefulWidget`

Always go through the Frame API rather than calling buffer functions directly. The Frame API is the stable, public contract.

```typescript
// ✅
frameRenderWidget(frame, renderParagraph(p), area);
frameRenderStatefulWidget(frame, renderStatefulList(list), area, listState);

// works but bypasses the frame abstraction
renderParagraph(p)(area, frame.buffer);
```

### Avoid stale widget configs

`createParagraph`, `createList`, and similar functions are cheap and can be called on every frame when the underlying data changes. Only cache them if creation is demonstrably expensive.

```typescript
// Fine to recreate on each tick — configs are plain objects
const render = (frame: Frame, area: Rect, liveData: string[]): void => {
  const list = createList(liveData, { block: blockBordered() });
  frameRenderWidget(frame, renderList(list), area);
};
```

### Share state via plain objects

Vision modules can share a plain object as their state. Perspectives pass this state down; they never need to know the internals.

```typescript
interface SharedState {
  selectedNav: number;
  logs: string[];
  cpuHistory: number[];
}

const state: SharedState = {
  selectedNav: 0,
  logs: [],
  cpuHistory: [],
};

// Both visions read from the same state object
navVision.render(frame, sidebarArea, state);
contentVision.render(frame, bodyArea, state);
```

### Delegate keyboard / input handling to the Perspective

A Perspective is the right place to handle key events and mutate shared state. Visions only read state; they never write to it outside their own internal concerns (e.g., scroll offset).

```typescript
// Perspective-level keypress handler
const onKey = (key: string): void => {
  if (key === 'ArrowDown') perspective.menuVision.selectNext();
  if (key === 'ArrowUp')   perspective.menuVision.selectPrev();
  if (key === 'Tab')       perspective.nextTab();
  requestRender();
};
```

---

## Complete Example: Two-pane Perspective with Tab Switching

The following runnable example ties everything together: a two-tab Perspective that switches between a metrics Vision and a log Vision.

```typescript
import {
  createTestBackendState,
  createTestBackend,
  createTerminal,
  terminalDraw,
  testBackendToString,
  createLayout,
  lengthConstraint,
  fillConstraint,
  splitLayout,
  frameRenderWidget,
  frameRenderStatefulWidget,
  createTabs,
  renderTabs,
  createList,
  createListState,
  renderStatefulList,
  gaugePercent,
  renderGauge,
  blockBordered,
  createTitle,
  styleFg,
  styleAddModifier,
  createStyle,
  Color,
  Modifier,
} from 'terminui';
import type { Frame, Rect } from 'terminui';

// ── Visions ──────────────────────────────────────────────────────────────────

const createMetricsVision = (cpu: number, mem: number) => ({
  render(frame: Frame, area: Rect): void {
    const layout = createLayout([fillConstraint(1), fillConstraint(1)]);
    const [cpuArea, memArea] = splitLayout(layout, area);

    frameRenderWidget(
      frame,
      renderGauge(gaugePercent(cpu, {
        block: blockBordered({ titles: [createTitle(`CPU ${cpu}%`)] }),
        gaugeStyle: styleFg(createStyle(), Color.Green),
      })),
      cpuArea,
    );

    frameRenderWidget(
      frame,
      renderGauge(gaugePercent(mem, {
        block: blockBordered({ titles: [createTitle(`Mem ${mem}%`)] }),
        gaugeStyle: styleFg(createStyle(), Color.Yellow),
      })),
      memArea,
    );
  },
});

const createLogVision = (entries: readonly string[]) => {
  const list = createList(entries, {
    block: blockBordered({ titles: [createTitle('Logs')] }),
    highlightStyle: styleFg(createStyle(), Color.LightCyan),
    highlightSymbol: '› ',
  });
  const state = createListState(entries.length - 1);
  const renderer = renderStatefulList(list);

  return {
    state,
    render(frame: Frame, area: Rect): void {
      frameRenderStatefulWidget(frame, renderer, area, state);
    },
  };
};

// ── Perspective ───────────────────────────────────────────────────────────────

const createAppPerspective = () => {
  let activeTab = 0;
  const tabLabels = ['Metrics', 'Logs'] as const;

  const metricsVision = createMetricsVision(42, 68);
  const logVision = createLogVision([
    '13:00:01 INFO  server started on :3000',
    '13:00:05 INFO  connected 3 clients',
    '13:00:12 WARN  high memory pressure detected',
    '13:00:30 ERROR connection reset by peer',
  ]);

  const highlightStyle = styleAddModifier(styleFg(createStyle(), Color.Yellow), Modifier.BOLD);
  const outerLayout = createLayout([lengthConstraint(3), fillConstraint(1)]);

  return {
    nextTab(): void {
      activeTab = (activeTab + 1) % tabLabels.length;
    },

    render(frame: Frame): void {
      const [tabArea, bodyArea] = splitLayout(outerLayout, frame.area);

      // TabsConfig is immutable — recreate with current selection each frame
      frameRenderWidget(
        frame,
        renderTabs(createTabs(tabLabels, { selected: activeTab, highlightStyle })),
        tabArea,
      );

      if (activeTab === 0) {
        metricsVision.render(frame, bodyArea);
      } else {
        logVision.render(frame, bodyArea);
      }
    },
  };
};

// ── Run ───────────────────────────────────────────────────────────────────────

const backendState = createTestBackendState(60, 12);
const terminal = createTerminal(createTestBackend(backendState));
const perspective = createAppPerspective();

// Frame 1 — Metrics tab
terminalDraw(terminal, (frame) => perspective.render(frame));
console.log('=== Metrics tab ===');
console.log(testBackendToString(backendState));

// Frame 2 — Logs tab
perspective.nextTab();
terminalDraw(terminal, (frame) => perspective.render(frame));
console.log('=== Logs tab ===');
console.log(testBackendToString(backendState));
```

---

## Summary

| Concept | Role | Owns |
|---|---|---|
| **Vision** | Renders one focused domain | Its own widget configs and state |
| **Perspective** | Orchestrates multiple Visions | Layout, tab index, shared state |

Use the `terminalDraw` / `terminalDrawJsx` loop as the heartbeat; let Perspectives own the layout split and the active-view decision; let Visions own their data and render logic. Because terminui's double-buffered diff renderer only flushes changed cells, switching between Visions and partial re-renders are fast by default.
