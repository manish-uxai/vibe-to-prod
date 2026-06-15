# Reachability Pass — Deterministic Orphan Detection

> **Run this once at the start of every audit and at the start of fix-it-all (Path 1 step 2).** It replaces grep-by-hand orphan detection. Hand-grepping is unreliable — across test runs the same codebase returned 8, 24, and 57 orphans depending on which files the agent thought to check. A real import-graph traversal returns the complete, identical list every time, on every model.

## Why proper resolution, not grep — and not a hand-rolled walk

Grep counts text matches; it cannot follow the import graph. A file can look orphaned (no one greps its name) while being reached via a barrel export, or look used (its name appears) while only living in a comment.

A hand-rolled forward-walk (parse imports with regex, traverse from the entry) is better than grep but has a dangerous failure: it only follows imports inside files it has _already reached_. Miss one live file — an unresolved alias, a dynamic import, a re-export the regex didn't parse — and every file imported _only_ by that missed file looks orphaned. These are **false orphans**, and deleting one breaks the build. A real run produced exactly this: a hand-rolled walk flagged `imports/Logo.tsx` as orphaned because it never reached `TopNavigation` (which imports Logo); deleting Logo broke the build and destroyed the designer's real logo.

The fix is to use a tool that does **real module resolution** via the project's tsconfig — the same resolution the bundler uses. That eliminates the regex-gap class of false orphans entirely.

## Primary method: madge

`madge` builds the dependency graph using real TS resolution (reads `tsconfig.json`, resolves `@/` aliases, follows dynamic imports). Figma Make exports ship with a tsconfig when the stack is Vite/React/TS, so aliases resolve correctly.

```bash
npx madge --orphans --extensions ts,tsx src
```

This was validated on a raw Figma export: it correctly resolved `@/imports/Logo` from a live component and did NOT flag Logo as an orphan — the exact case the hand-rolled walk got wrong. Warnings about skipped `figma:asset/...` virtual paths and external CSS (`tailwindcss`) are benign noise, not unresolved component imports.

**Capture full relative paths, not the basename summary.** madge's numbered summary collapses paths to basenames, which is ambiguous when two files share a name (e.g. a `MorningBriefing.tsx` in both `dashboard/` and `zone2/`). Use the full-path output so you act on the right file.

### Post-process madge output into three buckets

madge's raw list mixes three categories that MUST be treated differently. Sort every path:

1. **Entry points** (`main.tsx`, `index.tsx`) — madge flags these because nothing _imports_ the entry (it's loaded by the HTML). NOT orphans. **Drop them from the list.**
2. **`ui/` library primitives** (any path containing `/ui/` — `accordion.tsx`, `calendar.tsx`, `drawer.tsx`, etc.) — unused shadcn/Radix components that are a _library_, not dead exploration. **Note separately, never delete or refactor.** The designer may use them next week.
3. **App / variant orphans** (everything else — `StyleGuide.tsx`, dead `MorningBriefing.tsx` variants, `imports/` Figma artifacts) — genuine dead exploration. These go in the audit's "Orphaned files" section and are **deletion candidates** — but only after the inverse-grep confirm below.

## Fallback method: hand-rolled script (only if madge won't install)

If `npx madge` fails to install (locked-down environment, no registry access), fall back to the script below. It is less reliable than madge — it can produce false orphans on unresolved aliases — so the inverse-grep confirm below is MANDATORY when using it.

```js
const fs = require("fs");
const path = require("path");
const root = process.cwd();
const srcRoot = path.join(root, "src");
const entry = path.join(srcRoot, "main.tsx"); // adjust if entry differs (index.tsx, etc.)
const exts = [".ts", ".tsx", ".js", ".jsx"];

function read(file) {
  try {
    return fs.readFileSync(file, "utf8");
  } catch {
    return null;
  }
}

function resolveImport(fromFile, spec) {
  let base;
  if (spec.startsWith("@/")) base = path.join(srcRoot, spec.slice(2));
  else if (spec.startsWith("."))
    base = path.resolve(path.dirname(fromFile), spec);
  else return null; // bare package import — not a local file
  const candidates = [];
  for (const ext of exts) candidates.push(base + ext);
  for (const ext of exts) candidates.push(path.join(base, "index" + ext));
  candidates.push(base);
  return candidates.find((f) => fs.existsSync(f)) || null;
}

const reachable = new Set();
const queue = [entry];
while (queue.length) {
  const file = queue.pop();
  if (!file || reachable.has(file)) continue;
  reachable.add(file);
  const text = read(file);
  if (!text) continue;
  // static imports, re-exports, AND dynamic/lazy imports — all three matter
  const patterns = [
    /import\s+(?:[^'";]+\s+from\s+)?['"]([^'"]+)['"]/g,
    /export\s+[^'";]+\s+from\s+['"]([^'"]+)['"]/g,
    /import\(\s*['"]([^'"]+)['"]\s*\)/g,
  ];
  for (const regex of patterns) {
    for (const match of text.matchAll(regex)) {
      const resolved = resolveImport(file, match[1]);
      if (resolved && resolved.startsWith(srcRoot) && !reachable.has(resolved))
        queue.push(resolved);
    }
  }
}

// Inventory every component/source file, then subtract the reachable set
const all = [];
function walk(dir) {
  for (const entry of fs.readdirSync(dir, { withFileTypes: true })) {
    const full = path.join(dir, entry.name);
    if (entry.isDirectory()) walk(full);
    else if (entry.isFile() && /\.(tsx|ts)$/.test(entry.name)) all.push(full);
  }
}
walk(srcRoot);

const orphans = all
  .filter((f) => !reachable.has(f))
  .map((f) => path.relative(root, f))
  .sort();

// Two-bucket split: ui/ library primitives vs app/variant orphans
const uiOrphans = orphans.filter((f) => /\/ui\//.test(f));
const appOrphans = orphans.filter((f) => !/\/ui\//.test(f));

console.log(
  `reachable=${all.length - orphans.length}  orphans=${orphans.length}`,
);
console.log(
  "\n--- APP / VARIANT ORPHANS (skip for refactoring; deletion candidates — confirm with inverse grep first) ---",
);
appOrphans.forEach((f) => console.log(f));
console.log(
  "\n--- UI LIBRARY PRIMITIVES (unused, but DO NOT delete — they are a component library) ---",
);
uiOrphans.forEach((f) => console.log(f));
```

## Using the output

The three buckets are defined above (entry points → drop; ui/ primitives → leave alone; app/variant orphans → deletion candidates). Apply them the same way regardless of which method produced the list:

- **Audit:** put the app/variant orphans in the dedicated "Orphaned files (skip for refactoring)" report section. Note ui/ primitives separately and neutrally ("unused library primitives — leave in place"). Never put a ui/ primitive on the deletion list — deleting `card.tsx` because it's currently unused would be wrong; it's a library component.
- **Fix-it-all:** mark every app/variant orphan `SKIP — orphaned` in the progress ledger before the first edit. Never split, type-fix, icon-swap, or touch an orphaned file — not in round one, not in round two, regardless of size. A 2,000-line orphaned god-component is skipped entirely. Deletion of app/variant orphans is allowed, but ONLY through the safety protocol below.

## Deleting orphans safely — MANDATORY two-step, never delete on the detector's word alone

Both detection methods can misclassify. The hand-rolled fallback walks **forward** from the entry and produces false orphans when it misses a live importer. madge is far more reliable but can still skip files behind virtual import protocols. Either way, the orphan list **flags candidates**; it does not authorize deletion.

This is not hypothetical. A real run deleted an `imports/` folder flagged as orphaned; two files in it (`Logo.tsx`, an SVG paths file) were imported by live components (`TopNavigation`, `SampleStockView`) the forward-walk hadn't reached. The deletion broke the build, and the recovery — stubbing the logo back as an empty component — destroyed the designer's real logo. Both mistakes are forbidden by the protocol below. (Using madge would have resolved Logo's importer correctly and never flagged it — but the safety protocol stays regardless, as the net under any detector.)

**Step 1 — Confirm before deleting (the inverse check).** The detector asks "can I reach this file from the entry?" Before deleting ANY flagged orphan, ask the opposite question directly: **"does any file in the whole `src` tree import this one?"**

```bash
# For each deletion candidate, grep the entire tree for imports of it.
# Use the filename without extension; check both @/ alias and relative forms.
grep -rn "imports/Logo\|/Logo'\|/Logo\"" src/ --include='*.tsx' --include='*.ts'
```

If grep returns **even one importer**, the file is NOT an orphan — the script's reachable-set was incomplete. Drop it from the deletion list and treat it as live. Only files with **zero importers across the entire tree** are safe to delete. This inverse grep is authoritative because it scans every file directly instead of relying on the walk reaching them. Reachability _flags candidates_; this grep _authorizes deletion_.

**Step 2 — If a deletion breaks the build, RESTORE, never stub.** Even with step 1, if `npm run build` fails after deleting orphans, that failure is **proof a file was misclassified** — something live imported what you removed. The only correct response:

```bash
git checkout -- <deleted-file-path>   # restore the real file
```

Then re-mark it not-orphaned and move on. **NEVER** create a stub, an empty component, or a placeholder that "renders invisibly" to make the build pass — that silently destroys the designer's real asset (logo, icons, SVGs) while hiding the damage behind a green build. A broken build after deletion means the orphan detection was wrong; fix the classification by restoring, not by faking the deleted file. If the project isn't under git (no restore possible), do not delete orphans at all — only flag them in the report.
