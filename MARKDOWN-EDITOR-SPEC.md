# Markdown editor — everything needed to rebuild it as its own project

A Blazor live markdown editor: a textarea with line numbers and a context-aware
toolbar on the left, a rendered preview on the right, and a small block language
(`::: cards`, `::: panels`, `::: steps`, `::: bullets`) layered on top of ordinary
markdown.

Written from the current Who Goes copy at
`src/AkaPluss.Shared.Razor/Components/MarkdownEditor/`. The HellowSandbox copy is
the same code with different names (`Editor.razor` instead of
`MarkdownEditor.razor`, namespace `HellowSandbox.MarkdownEditor`) and no shared
design system.

---

## 1. Shape of the thing

~4,600 lines across 25 files. Three of them are most of it:

| File | Lines | What it is |
|---|---|---|
| `MarkdownEditor.razor` | 1,747 | The whole editor UI and all its state |
| `MarkdownEditor.razor.js` | 1,318 | Every text transform + the line-number gutter |
| `PageContentLoader.cs` | 373 | The `:::` block parser |
| `MarkdownEditor.razor.css` | 342 | Editor chrome (grid, gutter, toolbar, dialog) |
| `MarkdownView.razor(.css)` | 33 + 286 | Plain markdown → HTML, and prose styling |
| `ContentBlocks.razor(.css)` | 101 + 121 | Renders the parsed blocks |
| `IconBulletList.razor(.css)` | 36 + 53 | The `{icon} text` bullets |
| `CollapsibleTableExtension.cs` + `CollapsibleTableRenderer.cs` | 82 | Wraps tables in `<details>` |
| 12 × record files | 65 total | `ContentBlock` and its subtypes, one type per file |
| `_Imports.razor` | 8 | Kept in the folder so it compiles wherever it lands |

The folder is self-contained: drop it anywhere, fix the namespace, and it builds.
That was deliberate — it has already been moved twice.

---

## 2. Minimum project to host it

A Blazor **Server** (or any interactive-render) app. The editor is not static:
the toolbar, the options bar, the caret tracking and the line numbers are all JS
interop.

### NuGet

```
Markdig                                  # the only hard dependency
Microsoft.AspNetCore.Components.Web
```

`Markdig` version 1.3.2 is what Who Goes pins. `UseAdvancedExtensions()` is
required — the block language leans on its tables, generic attributes and
emphasis extras.

### Project file

A Razor Class Library (`Microsoft.NET.Sdk.Razor`) if you want it reusable, or
just drop the folder into the app. Colocated `.razor.js` is served at
`_content/<AssemblyName>/<path to the file>`, and **that path is hardcoded in
`OnAfterRenderAsync`** — change the assembly name and you must change the string:

```csharp
_module = await JS.InvokeAsync<IJSObjectReference>(
    "import",
    "./_content/AkaPluss.Shared.Razor/Components/MarkdownEditor/MarkdownEditor.razor.js");
```

If you build it as a plain app rather than a library, that becomes
`"./Components/MarkdownEditor/MarkdownEditor.razor.js"` or wherever it lands.

### The host page

```razor
@page "/editor"

<MarkdownEditor @bind-Value="_markdown"
                Images="_images"
                ImageFolder="illustrations" />

@code {
    private string _markdown = MarkdownEditor.SampleSource;
    private IReadOnlyList<string> _images = [];
}
```

Two things that cost time in Who Goes and are worth knowing up front:

- **Do not put `@rendermode` on the page** if the layout already renders the
  route subtree interactively. Declaring a render mode inside an
  already-interactive subtree throws, and the symptom is a blank page — no
  editor, no preview, no error in the UI.
- The component is a library and **cannot read the filesystem**. The host page
  supplies `Images` (file names) and `ImageFolder` (the path written into the
  markdown). In Who Goes that is
  `Env.WebRootFileProvider.GetDirectoryContents(folder)`.

---

## 3. Public API

The entire surface is four parameters and one constant.

```csharp
/// The markdown being edited. Two-way bindable: @bind-Value.
[Parameter] public string Value { get; set; } = "";
[Parameter] public EventCallback<string> ValueChanged { get; set; }

/// File names the Image picker offers, e.g. "savings.svg".
[Parameter] public IReadOnlyList<string> Images { get; set; } = [];

/// Folder the picker writes into the markdown path.
[Parameter] public string ImageFolder { get; set; } = "illustrations";

/// A short Norwegian document used as the default content.
public const string SampleSource = "...";
```

`PageContentLoader.Parse(string)` is also public, so a read-only consumer can
parse and render without the editor:

```razor
<ContentBlocks Blocks="PageContentLoader.Parse(markdown)" />
```

That is worth keeping separate — the editor is for authors, `ContentBlocks` is
for the pages that display the result.

---

## 4. The markdown dialect

**Everything is ordinary markdown unless it sits inside a `:::` fence.** That was
the design decision that made this workable: an earlier version meant "`##` is a
card", which stole a heading level from prose.

### Top level

| Syntax | Becomes |
|---|---|
| `# Title` | `TitleBlock` — an `<h1>`, and the only heading the parser intercepts |
| `## …` and deeper | left in the prose block, rendered as normal headings |
| anything else | `ProseBlock` → `MarkdownView` → Markdig |
| `::: kind [arg]` … `:::` | a structured block |

### `::: cards [collapsed]`

```markdown
::: cards
## Card headline {open}
**Label:** value
**Another:** value with **bold** and a [link](https://example.com)
:::
```

- Renders each card as a `<details>`.
- Fence arg `collapsed` → all cards start closed. Anything else → open.
- A trailing `{open}` or `{collapsed}` on the heading overrides the fence **for
  that card only** and is stripped from the text.
- **Card bodies hold nothing but `**Label:** value` lines.** Any other line is
  silently dropped. This is the single most surprising rule in the format and
  the reason the editor has a contextual options bar.

### `::: steps`

```markdown
::: steps
### Step title
Body markdown for this step.
:::
```

- `### ` opens a step; every following line is its body until the next `###`.
- Numbers come from a CSS counter, not the markup — reordering renumbers.
- Lines before the first `###` are discarded.

### `::: panels [1|2]`

```markdown
::: panels 1
## Panel title
Body markdown.
![alt text](illustrations/savings.svg)
{star} A bullet inside the panel
:::
```

- Arg is the column count: `1` = one wide panel, `2` = two across (default).
- `## ` opens a panel. Content before the first one is discarded.
- **An image on its own line is pulled out of the body** so it can sit *beside*
  the text rather than under it. An image inline in a sentence stays inline.
- Hidden below the `sm` breakpoint.

### `::: bullets`

```markdown
::: bullets
{dollar} Saves money
{star} Looks good
Plain line, gets the default icon
:::
```

- One bullet per non-blank line.
- `dollar` / `percentage` / `money` → coin glyph. **Everything else, including an
  unrecognised name, falls back to the star.** There is no error.

### Icon bullets inside other blocks

Fences do not nest — an inner `:::` would close the outer block. So `{icon} text`
lines are recognised **inside panel bodies and card field values** too, without a
fence. `SplitBullets` pulls them out of the body.

Consequence worth documenting for users: **bullets always render after the prose**,
regardless of where they appeared in the source. Interleaving prose between
bullets is not supported.

### From Markdig, available everywhere

| Syntax | Result |
|---|---|
| `**bold**` `*italic*` | standard |
| `++text++` | `<ins>` — underline. Markdown has no underline of its own |
| `~~text~~` | strikethrough |
| `==text==` | `<mark>` highlight |
| `^text^` / `~text~` | superscript / subscript |
| `[label](url){.cta}` | orange pill button (generic attributes) |
| a table | auto-wrapped in `<details>`; **more than 8 data rows starts collapsed** |

### Error handling

The parser never throws and never loses content:

- Unknown fence kind → rendered as prose.
- Unterminated fence → still yields its block at EOF.
- Malformed card field → dropped (the one exception to "never loses content").

---

## 5. How it fits together

```
Value (string)
  └─ OnParametersSet: normalise CRLF → LF, then
     PageContentLoader.Parse()  ──►  IReadOnlyList<ContentBlock>
                                       └─ ContentBlocks.razor renders each case
                                          ├─ TitleBlock  → <h1>
                                          ├─ ProseBlock  → MarkdownView → Markdig
                                          ├─ CardsBlock  → <details>/<dl>
                                          ├─ StepsBlock  → <ol class="steps">
                                          ├─ PanelsBlock → grid of panels
                                          └─ BulletsBlock→ IconBulletList
```

Two separate Markdig pipelines, deliberately:

- `MarkdownView` — `UseAdvancedExtensions()` **plus** `CollapsibleTableExtension`,
  for full documents.
- `PageContentLoader.RenderInline` — `UseAdvancedExtensions()` only, and it
  **unwraps a lone `<p>` wrapper** so a one-line panel body or card value does not
  get a block-level paragraph that breaks the layout.

### Why `Value` is normalised to LF

The browser reports caret offsets against LF regardless of what is in the
textarea. All the line maths in `@code` counts LF. A CRLF document would put
every offset out by one per line.

### The `_source` guard

`OnParametersSet` re-parses only when the text actually changed. It also runs
after every `ValueChanged` round trip, and re-parsing there would throw away the
caret context that the options bar depends on.

---

## 6. JS interop contract

One module, imported once on first render. Two return shapes:

```csharp
private sealed record InsertResult(string Value, int Caret);
private sealed record WrapResult(string Value, int SelectionStart, int SelectionEnd);
```

### Exports

| Function | Purpose |
|---|---|
| `insertSnippet(textarea, snippet)` | insert text at the caret |
| `insertBlock(textarea, template, sample, selectText)` | insert a block; `{body}` in the template marks where a selection goes |
| `wrapSelection(textarea, marker, placeholder)` | `**`, `*`, `++`, `~~`, `==`, `^`, `~` — toggles, counts the existing marker run |
| `linkSelection(textarea, sampleUrl, placeholder)` | selection → `[text](url)`, leaves the URL selected |
| `toggleHeading(textarea, level, placeholder)` | converts the marked lines rather than replacing them |
| `toggleList(textarea, ordered, placeholder)` | marked lines → bullet or numbered list |
| `insertBullet(textarea, icon, placeholder)` | `{icon} text` |
| `insertStep(textarea, item, fence, selectText)` | appends into an enclosing `::: steps` if there is one |
| `setLine(textarea, index, text, caretInLine, selectLength)` | rewrite one line — how the options bar edits fence args and `{open}` markers |
| `moveLines(textarea, direction)` | Alt+↑ / Alt+↓ |
| `setCaret` / `setSelection` / `focusAndSelect` | selection restore after a re-render |
| `attachGutter` / `detachGutter` | line numbers |
| `watchCaret(textarea, dotNetRef)` / `stopWatchingCaret` | caret → `[JSInvokable] CaretMoved(int)` |
| `download(filename, text)` | blob + object URL |

### `[JSInvokable]` on the C# side

- `CaretMoved(int caret)` — fires as the caret settles. Recomputes which block,
  link and heading the caret is in; **only calls `StateHasChanged` when one of
  those actually changed**, because records compare by value. Without that guard
  every arrow-key press re-renders.
- `MoveLines(int direction)` — Alt+↑/↓, routed through the same value-and-selection
  round trip as the toolbar so the move is undoable.

### The undo rule — do not lose this

```js
document.execCommand("insertText", false, text)
```

Assigning `textarea.value` directly **wipes the browser's native undo stack**, so
Ctrl+Z did nothing after a toolbar click. `execCommand` records the change as if
it had been typed. It is formally deprecated but is still the only way to write
into a textarea without destroying its history, and every current browser
supports it. There is a fallback that assigns `.value` and dispatches `input`,
but edits made that way are not undoable.

Each JS function computes a **whole new value**; `applyValue` then diffs it
against the current text and replaces only the smallest changed span. That keeps
the undo entry tight and the selection predictable.

### The gutter

Line numbers are not a `<pre>` next to the textarea — they have to survive soft
wrapping. `attachGutter`:

1. Builds a hidden mirror `<div>` copying 14 computed styles from the textarea
   (`fontFamily`, `lineHeight`, `whiteSpace`, `overflowWrap`, `tabSize`, …).
2. Measures each logical line's rendered height to know how many rows it occupies.
3. Writes the numbers into a strip that is **translated**, not scrolled —
   `transform: translateY(-scrollTop)` — because two real scroll positions cannot
   be kept in agreement to the pixel every frame.
4. A `ResizeObserver` catches both the window resizing and the textarea's own
   resize handle.

Two bugs that were fixed here and will come back if it is rewritten:

- Measure with `getBoundingClientRect().height`, **not `offsetHeight`**. Integer
  rounding against a 22.4px line-height drifted the gutter 47px over 120 lines.
  Snap the measured height to whole rows.
- The gutter must be `position: absolute; top: 0; bottom: 0`. As a flex item it
  sized itself from its own content and grew the textarea to 22,423px.

---

## 7. CSS architecture

### Scoped CSS and `::deep` — the thing to internalise

Blazor scoped CSS adds an attribute (`b-abc123`) to elements **the owning
component's `.razor` file writes**, and rewrites the last element of each
selector to require it. That means:

- `@((MarkupString)html)` output carries **no** scope attribute. Any rule
  targeting it needs `::deep`, hung off a container the component does render:
  `.panel-body ::deep a { … }` → `.panel-body[b-abc123] a { … }`.
- A child component's markup carries **that component's** attribute, not the
  parent's. `MarkdownView` styles its own anchors and those rules do not reach
  `ContentBlocks`' panels — which is why panel, step, card-field and bullet
  bodies each repeat the link colour.
- **A `<Button>` renders its own element.** No scoped rule from the calling
  component can style it. All styling has to go through its class parameter.

### Tailwind (v4) interactions

- Utilities live in a `@layer`. **Unlayered scoped CSS beats them regardless of
  specificity.** A `display: grid` in the scoped file silently defeated
  Tailwind's `.hidden`; the fix was `.editor-grid > section:not(.hidden)`.
- `@theme inline` **tree-shakes**: a `--color-*` variable is only emitted if some
  *utility* uses it. Eight of twelve tokens the editor's CSS referenced did not
  exist at runtime, and the export dialog rendered transparent. Every
  `var(--color-…)` in the editor CSS therefore carries a brand fallback:
  `var(--color-background, var(--white))`.
- Class names are found by **scanning source files as text**. A class assembled at
  runtime is never generated. Where the editor needs class strings in C#, they are
  written as literal constants so the scanner finds them.
- A utility that is not in the built CSS fails **silently**. `items-end` did
  nothing for a while for exactly this reason. Grep the built stylesheet before
  believing a utility works.

### The two-column alignment

The columns use **CSS subgrid** so the toolbar, the field and the preview line up
across the divide:

```css
.editor-grid {
    grid-template-columns: minmax(0, 1fr) minmax(0, 1fr);
    grid-template-rows: auto auto auto;   /* header, toolbar, body */
}
.editor-grid > section:not(.hidden) {
    display: grid;
    grid-row: 1 / -1;
    grid-template-rows: subgrid;
}
```

The toolbar only exists in the source column and its height changes with how many
rows of buttons wrap, so the two columns have to *agree* on that row rather than
each stacking independently. The preview column renders an empty
`<div aria-hidden="true">` to hold its place in the toolbar row.

`minmax(0, 1fr)` and `min-width: 0` are load-bearing: grid and flex items default
to `min-width: min-content`, so one wide preview table stretched the entire grid.

Hiding a column uses `.hidden`, not removal from the DOM — the textarea keeps its
scroll position, caret and `ElementReference`, and the preview keeps whichever
cards the reader had expanded.

### Tunables

```css
--editor-font: ui-monospace, SFMono-Regular, Consolas, 'Liberation Mono', Menlo, monospace;
--editor-font-size: 0.875rem;
--editor-line-height: 1.6;
--editor-padding: 1rem;
--editor-gutter-width: calc(var(--digits, 2) * 1ch + 1.5rem + 1px);
```

`--digits` is set from JS as the line count grows.

---

## 8. What a standalone project has to replace

The editor is currently wired into the AkaPluss design system. Porting means
providing these, or swapping them out.

### Design tokens the CSS reads

Twelve semantic tokens, each already carrying a brand fallback:

```
--color-background  --color-foreground     --color-muted-foreground
--color-border      --color-card           --color-section-primary
--color-primary     --color-primary-foreground  --color-primary-hover
--color-ring        --color-destructive    --color-destructive-foreground
```

The fallbacks they currently name — `--white`, `--deep-blue`, `--grayblue-1/2`,
`--sunglow-primary`, `--orange-strong`, `--warm-vanilla`, `--cream-white`, `--red`
— are Who Goes brand variables. Define either set.

### Shared components used

| Used | Where | Replacing it |
|---|---|---|
| `<Button>` | ~33 buttons: toolbar, segmented toggles, `+`, export, dialog | A plain `<button>` plus the CSS that was deleted when it was adopted. Budget a day. |
| `<DropdownMenu>` + `Trigger`/`Content`/`Item` | the Icon and Image pickers | Was a native `<select>` before. That worked but needed an `@key` hack — a select keeps showing the last thing picked, so picking the same image twice raised no event. |
| `TailwindComponent.Cn()` / tailwind-merge | how `Class` overrides win over a component's own classes | Only needed if you keep `Button`. |

`DropdownMenu` also needs `services.AddScoped<DropdownMenuJsInterop>()` and its
own JS bundle — check the DI registration when porting.

### Tailwind utilities used in the markup

If the new project has no Tailwind, these all need hand-written CSS:

```
bg-card bg-primary bg-section-primary border border-border border-card-info-border
flex flex-1 grid grid-cols-1 sm:grid-cols-[minmax(0,18rem)_1fr] sm:block
gap-3 gap-4 gap-5 gap-6 gap-x-8 gap-y-5 items-center items-start justify-end
self-center shrink-0 min-w-0 m-0 p-0 mb-1 mb-2 mb-4 mb-6 mb-8 mt-2 mt-3 mt-8
px-4 px-5 px-8 py-3 py-5 py-8 h-40 w-auto w-48 w-64 size-3 size-3.5 size-4
rounded-sm overflow-hidden overflow-auto hidden list-none font-bold font-semibold
text-sm text-lg text-4xl text-foreground text-muted-foreground text-primary-foreground
left-0 right-auto origin-top-left cursor-pointer
```

Plus the arbitrary values the toolbar Buttons carry: `h-6 h-7 h-9 w-6 w-8 p-0
px-3 text-xs text-base leading-none rounded-none first:rounded-l-[5px]
last:rounded-r-[5px] border-l first:border-l-0 disabled:pointer-events-auto
disabled:cursor-not-allowed rounded-[2px] px-[0.15rem] text-[0.7em]`.

---

## 9. Features, so nothing is lost in the rewrite

- Live preview, side by side, on a shared grid.
- Line numbers that follow soft wrapping.
- **Contextual block options bar** above the textarea. This is the feature that
  makes the block language usable — it shows the parameters of whatever block the
  caret is in, instead of making the author remember them:
  - cards → all open/collapsed, this card open/collapsed/default, `+` card, `+` field
  - panels → wide / two across, `+` panel
  - steps → `+` step
  - a link → normal / button (`{.cta}`)
  - otherwise → the block's name, or "put the cursor inside a `:::` block"
- Formatting buttons that **convert** rather than replace: marking text and
  clicking Heading turns that text into a heading.
- Context-aware disabling, each with a tooltip explaining why:
  - formatting is disabled on a heading line — headings render as plain text
  - Title and Heading are disabled inside a `:::` block — `#` would render a
    heading mid-panel and `##` would silently start a new card
- Alt+↑ / Alt+↓ moves the marked lines, undoably.
- Icon picker and image picker.
- Export dialog: editable filename, fixed `.md` suffix, Enter to confirm, Escape
  and backdrop to cancel.
- Column toggles: text only, preview only, or both. The last open column stays open.
- Block and character count under the field.

### Deliberately not supported

Worth writing down so they are not re-litigated:

- Nested fences (an inner `:::` closes the outer one — hence inline `{icon}` bullets).
- Prose interleaved between bullets.
- Anything other than `**Label:** value` inside a card.
- Headings inside a `:::` block.
- Text formatting inside a title.

---

## 10. Traps, in the order they will bite

1. **A Razor comment inside a tag's attribute list compiles**, then throws
   `InvalidCharacterError: … is not a valid attribute name` at render. The page
   comes up with just the heading and no error pointing at the cause. Put the
   comment above the tag.
2. **`aria-pressed="@someBool"` renders empty.** Blazor treats a bool attribute as
   present/absent; ARIA needs the words. Write `@(x ? "true" : "false")`.
3. **`![alt](url)` contains a valid `[alt](url)`** one character in. Link
   detection needs the `(?<!!)` lookbehind, or the options bar offers to turn an
   image into a CTA button.
4. **A `<Component>` that does not resolve compiles as a literal HTML tag** with
   only an RZ10012 *warning*, and silently renders nothing. This broke prose for a
   while after a folder move. Treat RZ10012 as an error.
5. **Scoped CSS does not reach another component's markup or a `MarkupString`.**
   See §7.
6. **Tailwind tree-shakes both utilities and theme variables.** See §7.
7. A collapsed browser pane gives zero-width measurements. Every layout number
   read from it is meaningless.

If the target repo has `TreatWarningsAsErrors` with StyleCop/Meziantou (Who Goes
does), budget real time for the port: one type per file (SA1402/SA1649), `using`
ordering, `string.Empty`, `CultureInfo.InvariantCulture`, and regex timeouts —
note `[GeneratedRegex]` has no `(pattern, timeout)` overload, so it needs the
explicit `RegexOptions` argument too.

---

## 11. Suggested layout for a fresh project

```
MarkdownEditor/                 # the RCL, or just a folder in the app
  MarkdownEditor.razor          # UI + state
  MarkdownEditor.razor.css
  MarkdownEditor.razor.js
  Parsing/
    PageContentLoader.cs
    Blocks/                     # the 12 records
    CollapsibleTableExtension.cs
    CollapsibleTableRenderer.cs
  Rendering/
    ContentBlocks.razor(.css)
    MarkdownView.razor(.css)
    IconBulletList.razor(.css)
  _Imports.razor
```

Worth doing differently this time:

- **Split `MarkdownEditor.razor`.** 1,747 lines in one file is the main thing
  making it hard to work on. The parsing/caret-context helpers (`DetectBlock`,
  `DetectLink`, `DetectPlainHeading`, `FindFenceClose`) are pure functions over a
  string and belong in their own class with unit tests.
- **Put the block language behind an interface.** `PageContentLoader.Parse` and
  `ContentBlocks` are already decoupled from the editor; formalising it would let
  a consumer register its own block kinds instead of editing the `switch`.
- **Test the parser directly.** It is pure `string → IReadOnlyList<ContentBlock>`
  with no I/O and no interop, and every rule in §4 is one test.
