# Nullish Coalescing (`??`) vs Logical OR (`||`) — Empty String Gotcha

> `??` only falls back on `null` or `undefined`. An empty string `""` passes straight through. Use `||` (or normalize first) when empty string should be treated as "no value".

## The operators

```ts
const a = "" ?? "fallback"   // → ""        ← empty string passes through
const b = "" || "fallback"   // → "fallback" ← empty string is falsy, falls back
```

`??` (nullish coalescing) guards against exactly two things: `null` and `undefined`.  
`||` (logical OR) guards against all falsy values: `null`, `undefined`, `""`, `0`, `false`, `NaN`.

## When it bites you

Any time user input, an API response field, or a config value can be set to `""` (empty string) rather than `null`/`undefined` to mean "not set", and you use `??` to provide a fallback.

Common sources of empty strings:
- Form fields that were cleared (result is `""` not `null`)
- API payloads where the backend sends `""` instead of omitting the field or sending `null`
- Config objects where the UI allows the user to delete a URL but the field remains in the payload

## Real example: care_fe print logo

The facility print template API can return:
```json
{ "branding": { "logo": { "url": "", "alignment": "left" } } }
```

In `PrintPreview.tsx`, the code to resolve the logo source was:
```tsx
src={logo?.url ?? careConfig.mainLogo?.dark}
```

When `logo.url === ""`:
- `??` sees a non-null, non-undefined value (`""`) → passes it through
- `<img src="">` renders with a broken-image icon
- The Care fallback logo is never used

**Fix** — normalize empty string to `undefined` once, then use `??` freely:
```tsx
const logoUrl = logo?.url || undefined;  // "" → undefined
// ...
src={logoUrl ?? careConfig.mainLogo?.dark}  // now ?? works correctly
```

Normalizing at the top keeps all downstream checks (`logoUrl ?`, `src={logoUrl ?? ...}`, guards) consistent without having to remember to use `||` everywhere.

## When to use each

| Use `??` | Use `\|\|` |
|---|---|
| Value is typed `string \| null \| undefined` and `""` is a valid value | Value is typed `string \| null \| undefined` and `""` means "not set" |
| `0` or `false` are valid values you want to keep | `0` or `false` should also trigger the fallback |
| You want to be precise about "no value" semantics | Any falsy value should fall back |

## TypeScript note

TypeScript won't warn you when you use `??` with a `string` type that could be `""`. The type is correct — it is a string — but the semantic meaning (empty = not set) is a runtime concern, not a type-level one. The bug is invisible to the type checker.

## References

- [MDN — Nullish coalescing operator (??)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Nullish_coalescing)
- Bug context: care_fe `PrintPreview.tsx` default logo broken when `branding.logo.url === ""`, April 2026
