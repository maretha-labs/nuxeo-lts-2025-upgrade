# Nuxeo Web UI / Studio project — 2023 → 2025

**TL;DR: the Web UI upgrade is transparent. Do NOT rewrite the legacy Polymer elements.** A Studio
project's custom UI (the `Polymer({})` / `<dom-module>` `.html` elements, behaviors, layouts) keeps
working on 2025 unchanged. The real work is tiny: the Studio target platform, optionally dependency
versions, and — only if present — functional tests. Most of this phase is **verification**, not editing.

## Why it's transparent

- **The framework base is unchanged.** nuxeo-web-ui `lts-2025` is still **Polymer 3**
  (`@polymer/polymer ^3.5.1`) with all `@polymer/iron-*` and `@polymer/paper-*` at `^3.x`. `lit`/`lit-html`
  exist *alongside* Polymer (hybrid), but **there is no Polymer→Lit migration** of existing elements.
- **web-ui's own `AGENTS.md` (lts-2025) is explicit**: "Polymer 3 SPA, no migration to Lit underway…
  most elements use the legacy Polymer factory — do NOT convert to class-based… `.html` files with
  `<dom-module>` + inline `<script>` are intentional and should not be refactored."
- **Nuxeo docs**: upgrading Web UI 2023→2025 is "a transparent process… no specific migration required
  for Studio projects."

So **do not**: convert `Polymer({...})` to class/Lit, remove `<dom-module>`, strip `@apply` mixins,
or restructure `<link rel="import">` — these are all still supported and intentional.

## What actually changes

1. **Studio `application.xml`** — target platform and dependency versions:
   ```xml
   <targetPlatform><version>lts-2025.0</version></targetPlatform>
   <dependencies>
     <string>nuxeo-web-ui:2025.x.0</string>          <!-- was 2023.x -->
     <string>nuxeo-wopi:2025.x</string>
     ... align addon versions to current 2025.x ...
   </dependencies>
   ```
   (Often already set if the user created the branch in Studio. Align any addon left on a 2023 or
   mismatched version.)

2. **Functional tests (only if the project has them)** — Node 18 → 22 from 2025.5.0+:
   `require()` → `import`, with **explicit `.js` extensions** on relative paths. This is the *only*
   code change Nuxeo documents for Web UI. Most Studio projects have **no** ftest (`*.js` step
   definitions / page objects) — if `find` shows none, there's nothing to do here.

3. **Nothing else** for typical Studio custom elements.

## How to verify (do this instead of editing)

Confirm there's no removed/renamed element, behavior, or layout the project depends on:

1. **Review the web-ui delta** between the 2023 and 2025 branches (branch names are `LTS2023` and
   `lts-2025`):
   ```bash
   curl -s "https://api.github.com/repos/nuxeo/nuxeo-web-ui/compare/LTS2023...lts-2025?per_page=250" > cmp.json
   # commit themes:
   grep -oE '"message": "[^"]+"' cmp.json | grep -iE "breaking|remov|renam|deprecat|drop"
   # removed/renamed files (look for anything under elements/ or *behavior*/*layout*):
   python3 -c "import json;[print(f['status'],f['filename']) for f in json.load(open('cmp.json'))['files'] if f['status'] in ('removed','renamed')]"
   ```
   In the real 2025 delta this was all **additive/non-breaking** — RTL (Hebrew/Arabic), accessibility,
   Crowdin/i18n, CI/Chrome-for-testing bumps, bug fixes — and the only removed files were CI/ftest
   tooling (`.eslintrc.js`, addon step-definitions). **Zero elements/behaviors/layouts removed.**
   (Note: GitHub's compare caps `files` at ~300 and `commits` at 250; for a huge delta, also verify
   the project's actual dependency surface below.)

2. **Cross-reference the project's dependency surface.** In the Studio UI resources
   (`studio/resources/nuxeo.war/ui/`), extract and confirm each still exists in `lts-2025`:
   ```bash
   grep -rhoE "(Nuxeo|Polymer)\.[A-Za-z]+Behavior" --include=*.html . | sort | uniq -c   # behaviors
   grep -rhoE 'include="[^"]+"' --include=*.html . | sort | uniq -c                        # style includes
   grep -rhoE "<nuxeo-[a-z-]+" --include=*.html . | sort | uniq -c                         # elements used
   grep -rhoE '<link[^>]*rel="import"[^>]*href="[^"]+"' --include=*.html .                 # imports
   ```
   Core behaviors (`Nuxeo.LayoutBehavior`, `DocumentContentBehavior`, `RoutingBehavior`,
   `FiltersBehavior`, `FormatBehavior`, `Polymer.Iron*Behavior`) and style includes (`nuxeo-styles`,
   `iron-flex*`, `nuxeo-action-button-styles`) all still ship in 2025. **Internal** relative imports
   never break from web-ui changes. Removed slots/layouts degrade gracefully (the override just doesn't
   render) — they don't crash the app.

## EXCEPTION: an overridden `ui/index.jsp` (Web UI bootstrap) is NOT transparent

"Transparent" applies to Studio custom **elements**. If the project overrides the Web UI
**bootstrap page** — `web/nuxeo.war/ui/index.jsp` (usually in the *platform* repo's core bundle,
sometimes shipped from Studio resources) — a 2023-era copy breaks the whole UI on 2025, because
2025's Web UI is **CSP-nonce based** and changed its bundle-loading bootstrap. Console signature
(each maps to a specific gap in the old override):

```
config.jsp:6 Uncaught ReferenceError: NuxeoNonce is not defined      ← index.jsp must define const NuxeoNonce
main.bundle.js Uncaught TypeError: Nuxeo.UI.bundles is not iterable  ← old WebResourceManager <link rel="import"> loop + bare Nuxeo.UI={} shim must go
main.bundle.js method `_drawerAlign` not defined                     ← cascade: custom elements failed to register
TypeError: this.nxProvider.fetch is not a function                   ← cascade: page-provider elements never upgraded (lists/tasks won't load or sort)
```

Fix by rebasing the overridden `index.jsp` on the 2025 stock pattern, keeping the project's
customizations:
1. a scriptlet that generates a per-request `NX_NONCE_VALUE` (`UUID.randomUUID()`) and appends
   `'self' 'strict-dynamic' 'nonce-<value>'` to the response's `Content-Security-Policy`
   `script-src` (plus `object-src 'none'` if absent);
2. `<script nonce="..."> const NuxeoNonce = "<%= NX_NONCE_VALUE %>"; </script>` **before**
   `config.jsp` loads;
3. `nonce="<%= NX_NONCE_VALUE %>"` on **every** `<script>` tag — including the project's own custom
   scripts (a script without the nonce is blocked by the CSP);
4. the 2025 RTL/i18n bootstrap block (`window.nuxeo.I18n.direction` + `dir` attribute);
5. **remove** the 2023-era `WebResourceManager`/`ResourceContextImpl` `<link rel="import">`
   resources loop and the `var Nuxeo.UI = {}` script — 2025's `config.jsp`/`main.bundle.js` own
   bundle loading;
6. `jakarta.servlet.http.HttpServletResponse` import (not `javax`).

**If the override lives in the Studio project** (`studio/resources/nuxeo.war/ui/index.jsp`), do
NOT edit it from the platform repo — tell the user it must be updated in Studio Designer (or
removed there so the stock 2025 page applies).

Related: a vendored **`redefine-bundled.js`** (the custom-elements redefinition polyfill some
projects load from `nuxeo.war/ui/`) must stay in its original **minified single-line** form. A
prettified copy is corrupt — the re-formatting collides minified variable names
(`Uncaught SyntaxError: Identifier 'e' has already been declared`) and the polyfill never loads.
Restore the minified original and check it with `node --check`. Also harmless-but-noisy:
`dom-module <name> has style outside template` is a cosmetic Polymer warning from a custom
element, not an upgrade breakage.

After fixing, deploy and **hard-refresh** (the service worker/browser cache serves the old broken
scripts otherwise).

## Pre-existing gotchas to surface (not caused by the 2025 upgrade)

- **Dead CDNs**: old Studio UIs often `<link>` external assets like Font Awesome 4.x from
  `//maxcdn.bootstrapcdn.com` (sunset) — those icons may already be failing. Flag it; offer to swap
  the source. Unrelated to 2025, but a good time to fix.
- **Dependency drift** in `application.xml` (e.g. `nuxeo-web-ui` left on an earlier `2025.x` than the
  other addons) — align them all to a current `2025.x`.

## Recommendation to give the user

Don't change UI code preventively. Make the `application.xml` change (if not already done), migrate
functional tests if any exist, run the verification above, then **deploy to a 2025 sandbox and
smoke-test**. If something specific misbehaves, get the browser console error and fix *that* — don't
rewrite the legacy elements.
