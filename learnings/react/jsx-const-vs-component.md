# React: `const element = (...)` vs `function Component()` vs `renderX()`

> For local, static JSX reused once or twice in the same render, prefer `const element = (...)`. For anything that takes props, has variations, or might be reused/tested separately, extract to a `function Component(props)`.

## Three common patterns

```tsx
// 1. Element constant — a React element value, created once per render
const facilityInfo = (
  <div>
    <h1>{facility.name}</h1>
  </div>
);
{facilityInfo}

// 2. Component — a function returning JSX, called by React with its own lifecycle
function FacilityInfo({ facility }: { facility: Facility }) {
  return (
    <div>
      <h1>{facility.name}</h1>
    </div>
  );
}
<FacilityInfo facility={facility} />

// 3. Render function — a plain helper that returns JSX, called manually
const renderFacilityInfo = (facility: Facility) => (
  <div>
    <h1>{facility.name}</h1>
  </div>
);
{renderFacilityInfo(facility)}
```

## Differences at a glance

| Aspect | `const element` | `function Component` | `renderX()` helper |
|---|---|---|---|
| Takes props | No | Yes (typed, idiomatic) | Yes (parameters) |
| Has its own lifecycle / hooks | No | Yes | No (runs in caller's scope) |
| Shows up in React DevTools as a node | No | Yes | No |
| Memoizable independently | No | Yes (`React.memo`) | No |
| Best for | Local static JSX reused in same render | Reusable, parameterized, testable UI | Quick local variations needing args |
| Cost when unused conditionally | Created every render even if not rendered | Only called when used in JSX | Only called when invoked |

## When to choose which

**Use `const element = (...)`** when:
- The markup is built from values already in scope.
- You reuse it 1–3 times in the same component (e.g. logo placed in multiple alignment branches).
- No parameters needed.

**Use `function Component()`** when:
- You need props.
- It has its own state, effects, or context.
- It's reused across files, or you want it to appear as a discrete node in React DevTools.
- You may want to memoize it.

**Use a `renderX()` helper** when:
- You need parameters but don't want to pay the cost of a full component (no hooks, no DevTools node, simple inline variation).
- It's a one-off inside a single component — extracting a full component would be over-engineering.

## Real-world example: care_fe `PrintPreview`

In `FacilityPrintLayout`, the logo + facility-info block is rendered three different ways depending on `alignment`:

```tsx
const facilityInfo = (
  <div className="text-left">
    <h1>{facility.name}</h1>
    {/* ... */}
  </div>
);

const logoImg = <img src={...} alt={...} />;

return alignment === "center" ? (
  <div className="flex flex-col items-center">
    {logoImg}
    {facilityInfo}
  </div>
) : (
  <div className="flex justify-between">
    {alignment === "left" ? <>{logoImg}{facilityInfo}</> : <>{facilityInfo}{logoImg}</>}
  </div>
);
```

`const` was the right pick because:
- `facilityInfo` and `logoImg` are reused in 3 layout branches in the same render.
- They take no params; they read from closure (`facility`, `logo`).
- No internal state, no DevTools node value-add.
- Extracting to components would mean threading 5+ props for no benefit.

If `facilityInfo` later needed to display optional fields conditionally per template, or had to live in another file, **promote it to a real component** (`<FacilityHeaderInfo facility={facility} />`).

## Rule of thumb

> Start with `const element = (...)`. Promote to a component the moment you need props, hooks, or reuse outside the file.

Don't over-engineer with components for one-off local markup — and don't pretend a `const` is a "component" (it isn't, it's just a value).

## References

- [React docs — Your first component](https://react.dev/learn/your-first-component)
- Bug context: care_fe `PrintPreview.tsx` print-template logo alignment refactor, April 2026
