# Platform primitives — what already ships before you reach for a library

This is the lookup for the platform-primitive rung of the ladder — a native element over a component library, CSS over JavaScript, a database constraint over application code — and a library earns its place only when the primitive is genuinely insufficient: old-browser support it lacks, an edge case it doesn't handle, or ergonomics that matter at scale.

## HTML elements

The browser ships these as native form controls and widgets before you install a component for them.

| You reach for | The platform has |
|---|---|
| Date picker library | `<input type="date">` |
| Time picker library | `<input type="time">` |
| Color picker library | `<input type="color">` |
| Range slider library | `<input type="range">` |
| Progress bar component | `<progress value="70" max="100">` |
| Meter/gauge component | `<meter value="0.7">` |
| Modal/dialog library | `<dialog>` + `dialog.showModal()` |
| Accordion/FAQ component | `<details><summary>Title</summary>…</details>` |
| Searchable dropdown | `<input list="id"> <datalist id="id">` |

## CSS

Reach for these before writing JavaScript to do what a stylesheet already does.

| You reach for | The platform has |
|---|---|
| Responsive font size | `font-size: clamp(1rem, 2.5vw, 2rem)` |
| Fluid spacing | `padding: clamp(1rem, 5vw, 3rem)` |
| Dark mode | `@media (prefers-color-scheme: dark)` |
| Reduced motion | `@media (prefers-reduced-motion: reduce)` |
| Responsive layout without breakpoints | `grid-template-columns: repeat(auto-fill, minmax(250px, 1fr))` |
| Component-level responsive design | `@container` queries |
| Global design tokens / theming | CSS custom properties (`--color-primary: #7c3aed`) |
| Smooth scroll | `scroll-behavior: smooth` |
| Sticky header | `position: sticky; top: 0` |
| Scroll-snap carousel | `scroll-snap-type: x mandatory` + `scroll-snap-align: start` |
| Aspect ratio enforcement | `aspect-ratio: 16 / 9` |
| Truncate text with ellipsis | `overflow: hidden; text-overflow: ellipsis; white-space: nowrap` |
| Multi-line text clamp | `-webkit-line-clamp: 3` |
| CSS cascade layers (style isolation) | `@layer base, components, utilities` |
| Nested CSS selectors | Native CSS nesting (no preprocessor needed) |
| `has()` parent selector | `:has(input:checked)` |

## JavaScript and browser APIs

Runtime and browser APIs that make an installed package redundant.

| You reach for | The platform has |
|---|---|
| `query-string` / `qs` | `new URLSearchParams(location.search)` |
| `lodash.clonedeep` | `structuredClone(obj)` |
| `lodash.groupby` | `Object.groupBy(arr, fn)` |
| `numeral` / `accounting` | `new Intl.NumberFormat("en-US", { style: "currency", currency: "USD" })` |
| `date-fns` format | `new Intl.DateTimeFormat("en-US", { dateStyle: "long" }).format(date)` |
| `date-fns` relative time | `new Intl.RelativeTimeFormat("en", { numeric: "auto" }).format(-3, "day")` |
| `plural` / `i18n` plurals | `new Intl.PluralRules("en-US").select(count)` |
| `clipboard.js` | `navigator.clipboard.writeText(text)` |
| `uuid` (v4) | `crypto.randomUUID()` |
| Infinite scroll library | `new IntersectionObserver(cb).observe(sentinel)` |
| Resize listener library | `new ResizeObserver(cb).observe(element)` |
| DOM mutation watcher | `new MutationObserver(cb).observe(el, options)` |
| `is-online` / `connectivity check` | `navigator.onLine` + `online`/`offline` events |
| `store.js` / `localForage` (simple case) | `localStorage.setItem(key, JSON.stringify(val))` |
| Abort fetch on timeout | `AbortSignal.timeout(5000)` passed to `fetch` |
| Custom event bus | `new EventTarget()` / `dispatchEvent(new CustomEvent("x", { detail }))` |

## Node.js standard library

Built-ins that make a wrapper package redundant.

| You reach for | The platform has |
|---|---|
| `mkdirp` | `fs.mkdirSync(path, { recursive: true })` |
| `rimraf` | `fs.rmSync(path, { recursive: true, force: true })` |
| `make-dir` | `fs.mkdirSync(path, { recursive: true })` |
| `slash` (win paths) | `path.posix` or `path.normalize()` |
| `uuid` (v4) | `crypto.randomUUID()` |
| `is-stream` | `val instanceof stream.Readable` |
| `object-assign` | `Object.assign()` / spread |
| `array-uniq` | `[...new Set(arr)]` |
| `array-flatten` | `arr.flat(Infinity)` |
| `flat` | `arr.flat(depth)` |
| `path-exists` | `fs.existsSync(path)` |
| `load-json-file` | `JSON.parse(fs.readFileSync(path, "utf8"))` |
| `write-json-file` | `fs.writeFileSync(path, JSON.stringify(obj, null, 2))` |
| `pkg-dir` | `path.resolve(__dirname, "..")` / `import.meta.dirname` |

## Python standard library

Standard-library equivalents for packages that wrap what Python already ships.

| You reach for | The platform has |
|---|---|
| `python-dateutil` (basic parsing) | `datetime.fromisoformat()` (Python 3.7+) |
| `pytz` | `zoneinfo.ZoneInfo("America/New_York")` (Python 3.9+) |
| `attrs` (simple data classes) | `@dataclass` |
| `six` | drop it, Python 2 is gone |
| `pathlib2` | `pathlib.Path` (built-in since Python 3.4) |
| `enum34` | `enum.Enum` (built-in since Python 3.4) |
| `typing_extensions` (common types) | `from __future__ import annotations` + built-in generics |
| `simplejson` (basic use) | `json` (stdlib) |
| `click` (single command) | `argparse` (stdlib) |
| `mergedeep` | `dict \| other_dict` (Python 3.9+) |
| `more-itertools` (basic) | `itertools` (stdlib): `chain`, `islice`, `groupby`, `product` |
| `toolz` (basic) | `functools`: `lru_cache`, `partial`, `reduce` |

## Database

Database features that replace application-layer code for the same job. Constraints are the integrity backstop behind boundary validation, not a replacement for it: the boundary still rejects bad input with a handled error (§9 of the skill), and the constraint catches whatever slips past any code path.

| You reach for | The platform has |
|---|---|
| Pagination offset/limit | `LIMIT 20 OFFSET 40` |
| Running totals | `SUM(...) OVER (ORDER BY date)` (window function) |
| Rank within group | `RANK() OVER (PARTITION BY category ORDER BY score DESC)` |
| Pivot / cross-tab | `FILTER (WHERE ...)` + conditional aggregation |
| Deduplication | `SELECT DISTINCT` / `ON CONFLICT DO NOTHING` |
| Soft-delete filtering | Generated column + partial index |
| Tree traversal | Recursive CTE (`WITH RECURSIVE`) |
| Full-text search (basic) | `tsvector` / `MATCH AGAINST` / `FTS5` |
| JSON storage + query | `jsonb` (Postgres) / `JSON_EXTRACT` (SQLite/MySQL) |
| UUID generation | `gen_random_uuid()` (Postgres) / `UUID()` (MySQL) |
| Timestamps on insert/update | `DEFAULT now()` + trigger or `ON UPDATE CURRENT_TIMESTAMP` |
| Enforce uniqueness | `UNIQUE` constraint |
| Enforce referential integrity | `FOREIGN KEY` |
| Enforce value ranges | `CHECK (price > 0)` |

## Provenance and maintenance

Last verified 2026-09. Every row is a platform feature whose support drifts with browser and runtime releases, so before relying on a row in production, check the installed runtime's or target browsers' support for that exact feature. When a row stops holding, delete it rather than caveating it — a table of caveats is what makes a reader stop trusting the table.
