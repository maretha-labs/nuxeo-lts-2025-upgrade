---
name: nuxeo-2025-upgrade
description: Upgrade a Nuxeo project from LTS 2023 to LTS 2025 — Maven plugin/marketplace code and/or the Studio Web UI project. Use when the user wants to migrate a Nuxeo project to 2025, bump nuxeo-parent from 2023.x to 2025.x, move javax→jakarta, fix a Nuxeo build broken by the 2025 upgrade, or check whether a Studio/Web UI (Polymer) project needs changes for 2025. Asks for the project path, then applies version bumps, code migrations, test-infra fixes, and builds with JDK 21; for Web UI it verifies (rather than rewrites) the legacy elements.
---

# Nuxeo LTS 2023 → 2025 upgrade

You are upgrading a Nuxeo Maven project from LTS 2023 to LTS 2025. Work **in the user's
target project**, not in this skill's repo. Be autonomous: apply the known migrations, build,
read the compiler/test errors, and fix them. The companion file
[`reference/migrations.md`](reference/migrations.md) is your encyclopedia of exact before→after
changes — read it before editing code.

There are two kinds of project; ask which (or detect from the path):
- A **Maven plugin / marketplace** project (has a root `pom.xml`) → Phases 0–6 below.
- A **Studio Web UI** project (has `application.xml` + `studio/resources/nuxeo.war/ui/` Polymer
  `.html` elements) → **Phase UI** below, driven by
  [`reference/web-ui.md`](reference/web-ui.md). The Web UI upgrade is mostly transparent; that phase
  is verification, not a rewrite. A project may be both (a platform repo + a separate Studio repo) —
  handle each on its own path.

## Phase 0 — Preflight

1. Ask the user for the **absolute path to their Nuxeo project** (the directory with the root
   `pom.xml`). Confirm it's a Maven project currently on Nuxeo **2023** (root pom parent
   `org.nuxeo:nuxeo-parent:2023.x`). If it's already 2025 or not Nuxeo, stop and say so.
2. Verify **JDK 21** is available: `/usr/libexec/java_home -v21` (macOS) or check `java -version`.
   All Maven commands in this skill must run with `JAVA_HOME` pointed at JDK 21, e.g.
   `JAVA_HOME=$(/usr/libexec/java_home -v21) mvn ...`. Nuxeo 2025 enforces Java `[21,)`.
3. Decide the **target version** with the user (see "Versions" in migrations.md): the build
   parent is 2-part (e.g. `2025.16`) but the platform artifacts are **3-part** (e.g.
   `2025.16.13`). Get both, or ask which `2025.x` to target. If unsure, use the latest stable
   `nuxeo-parent` and read its `<parent>` version to find the matching platform version.
4. Create/switch to a working branch (e.g. `upgrade-2025`) if the user hasn't. Never commit or
   push unless asked.

## Phase 1 — Assess

Read every `pom.xml` (root + modules) and the marketplace `package.xml`. Record: parent version,
the `nuxeo.platform.version`/`nuxeo.distribution.version`/`nuxeo.target.version` properties,
`maven.compiler.release`, module list, the Studio artifact + version, the marketplace
`<platform>` line, and CI JDK/Maven versions. Note explicitly-pinned dependency versions (these
are the dangerous ones — see commons-io below).

Then grep the source tree for the migration triggers (the lists in migrations.md): `javax.`
imports, `org.joda.time`, `lombok`, `org.mockito.Matchers`, `nuxeo-elasticsearch-core`,
`RepositoryLightElasticSearchFeature`, `getForm()`/`FormData`, `mock-javamail`, removed
`HttpSession` methods, `defaultScroller="elastic"` in bulk-action contribs, **overridden
server JSPs** (`src/main/resources/web/nuxeo.war/*.jsp` — especially `login.jsp`, and
`ui/index.jsp` + a vendored `ui/redefine-bundled.js`, which need the 2025 CSP-nonce bootstrap —
see web-ui.md), and custom index config in env templates (`elasticsearch-doc-mapping.json` /
`elasticsearch-doc-settings.json*` under `nxserver/config/` — must be renamed for 2025). Build a
concrete work-list scoped to what this project actually uses.

## Phase 2 — Versions & POMs

Apply, per migrations.md "Versions & build":
- Root pom: `nuxeo-parent` → target 2-part version; `nuxeo.platform.version` /
  `nuxeo.distribution.version` / `nuxeo.target.version` → the **3-part** platform version;
  `maven.compiler.release` 17 → 21; bump per-module `<source>/<target>/<release>` 17 → 21.
- Marketplace `package.xml`: `<platform>lts-2023.*` → `lts-2025.*`.
- CI (`.github/workflows`, Jenkinsfile): JDK 17 → 21, Maven → 3.9.x.
- Studio dependency: set the project's Studio GAV to a 2025-built version (ask the user; Maven
  versions can't contain `/`, so a branch like `feature/2025` must be the sanitized GAV from
  Studio → Branch Management → "Maven GAV", often `2025-SNAPSHOT`).
- **Remove obsolete explicit version pins** that are older than 2025 needs — especially
  `commons-io` (must be ≥ 2.17; let the BOM provide 2.21.0).

## Phase 3 — Source code migrations

Work through migrations.md "Source code". The big ones: `javax.{inject,ws.rs,servlet,mail}` →
`jakarta.*` (and `javax.servlet-api` → `jakarta.servlet-api`); Joda-Time → `java.time`;
remove Lombok usages (old Lombok breaks on JDK 21); drop jetbrains `@NotNull`; Elasticsearch
core dep → `nuxeo-core-search`; WebEngine `FormData`/`getForm()` → `MultivaluedMap`/`getRequest()`;
`TaskObject` package move; `commons-lang` → `lang3`; PDFBox 2 → 3; Mockito `Matchers` →
`ArgumentMatchers`. Prefer mechanical, project-wide `perl -0pi` substitutions for the namespace
moves, then let the compiler catch the rest.

Two migrations the compiler will **not** catch (they fail only at runtime — see migrations.md):
- **Overridden `login.jsp`** (or any `nuxeo.war` JSP): removed helpers (`MobileBannerHelper`,
  `LoginScreenHelper.getDisplayMobileBanner`) make the JSP fail to compile on the server — every
  login 500s. Diff the override against the stock 2025 JSP and surgically remove the dead API usage.
- **Bulk actions with `defaultScroller="elastic"`**: the `elastic` scroller is a deprecated alias
  that only exists if the `nuxeo-search-client-opensearch1` bundle is deployed. Change the
  production contributions to `defaultScroller="search"` (the canonical 2025 scroller).

## Phase 4 — Test-infrastructure migrations

Tests fail differently from main code. Apply migrations.md "Tests": `RepositoryLightElasticSearchFeature`
→ `CoreSearchFeature`; `mock-javamail` → `SmtpMailServerFeature` (point `mail.transport.port` at
`nuxeo.test.mail.smtp.port`, disable auth/tls, read via `MailsResult`); `TransientStoreFeature`
package move; add `CollectionFeature`/`SQLDirectoryFeature` deps where transitive ones vanished;
remove Servlet-6 `HttpSession` method overrides; `comment-core` → `comment`. For bulk actions,
change `defaultScroller="elastic"` → `"search"` in the **production** contribs (Phase 3) — the
`search` scroller exists under `CoreSearchFeature` too, so no test-only alias is needed. (Earlier
versions of this skill added an `elastic`→repository test alias and kept `elastic` in production;
that is superseded — `elastic` also breaks on real 2025 cloud deployments.)

## Phase 5 — Build & fix loop

Build incrementally with JDK 21, reading errors and fixing against migrations.md:
```
JAVA_HOME=$(/usr/libexec/java_home -v21) mvn -nsu clean test-compile -B -fae
```
- `-fae` (fail at end) surfaces all module errors per run; `-nsu` avoids slow snapshot re-resolution.
- Distinguish **code errors** (fix per migrations.md) from **resolution errors** (need the private
  2025 repo / VPN — tell the user, don't fight them).
- When `test-compile` is green, run the suite: `mvn -nsu test -pl <module>` (Nuxeo tests run on
  H2/in-mem by default — no MongoDB needed for most). Fix failures; many are the test-infra items
  in Phase 4. Re-read migrations.md "Tests" for the subtle ones (DublinCore contributor overwrite,
  multi-recipient SMTP parsing, directory schemas).
- If a test relies on infra that's genuinely hard to reconstruct in 2025 (large REST-client
  tests on the removed Jersey 1.x stack), it's acceptable to **quarantine** it via
  `maven-compiler-plugin` `<testExcludes>` with a `TODO(2025-upgrade)` comment and tell the user —
  rather than block the whole upgrade. Track it explicitly.

## Phase 6 — Report

Summarize: versions changed, code migrations applied, test fixes, build/test status, anything
quarantined, and anything needing the user (Studio GAV, private-repo access, an unresolved test).
Be honest about what was verified (compiled? tests run?) vs. assumed.

Remind the user about **deployment** search packages: if the target runs on OpenSearch, they must add
`nuxeo-search-client-opensearch1` and `nuxeo-audit-opensearch1` to the distribution/marketplace
packages or search and audit come up unconfigured (see migrations.md "Elasticsearch core removed").

## Phase UI — Studio Web UI project (Polymer)

For a Studio Web UI project, read [`reference/web-ui.md`](reference/web-ui.md) first. The headline:
**the Web UI 2023→2025 upgrade is transparent — do NOT rewrite the legacy `Polymer({})` /
`<dom-module>` `.html` elements, behaviors, or layouts.** This phase is mostly verification.

1. **`application.xml`**: ensure `<targetPlatform><version>lts-2025.0</version>` and align the
   `<dependencies>` addon versions (`nuxeo-web-ui`, `nuxeo-wopi`, …) to current `2025.x`. Often the
   user already did this when creating the Studio branch.
2. **Functional tests** (only if present — `find` for `*.js` step-definitions/page-objects under an
   `ftest`/test dir): Node 18→22 means `require()` → `import` with explicit `.js` extensions. Most
   Studio projects have none → skip.
3. **Verify, don't edit**: run the two checks in web-ui.md — (a) the `LTS2023...lts-2025` web-ui
   compare for removed/renamed elements/behaviors/layouts (the real 2025 delta removed none — it's
   RTL + a11y + i18n + tooling), and (b) the project's own behavior/style/element/import surface
   against what 2025 still ships (all core behaviors and `iron-*`/`nuxeo-styles` remain).
4. **Surface, don't silently fix**: flag pre-existing non-2025 issues (dead CDNs like Font Awesome
   from `maxcdn.bootstrapcdn.com`, dependency drift) and recommend a **2025 sandbox smoke-test** —
   fix specific console errors if they appear, rather than refactoring elements preventively.

Do **not** convert Polymer elements to Lit/class-based, remove `<dom-module>`, or strip `@apply` —
nuxeo-web-ui 2025 is still Polymer 3 and its maintainers mark those patterns as intentional.

## Known sharp edges

- **Version mismatch 404s**: setting the platform property to the 2-part parent version (e.g.
  `2025.16`) makes every nuxeo artifact 404 — it must be the 3-part platform version (`2025.16.13`).
- **commons-io downgrade**: an explicit `<version>2.14.0</version>` (or any < 2.17) breaks
  Nuxeo's `SQLDirectory` at runtime with `NoSuchFieldError/NoClassDefFoundError` on
  `UnsynchronizedBufferedReader`. Remove the pin — and **re-check after every merge from
  mainline**: merging `master`/`release-candidate` into the upgrade branch can reintroduce the
  old `<dependencyManagement>` block, failing only in CI ("CI fails, local passes" — the CI log
  shows `Downloaded ...: commons-io-2.14.0.jar` and a different reactor version than your local
  pom). Verify with `mvn dependency:tree -Dincludes=commons-io`.
- **Lombok on JDK 21**: a stale Lombok throws `NoSuchFieldError: JCImport.qualid` during compile
  even if only one annotation is used. Remove the usage (or bump Lombok ≥ 1.18.30).
- **Studio `/` in version**: a Studio branch name with a slash is not a valid Maven version; use
  the GAV from Studio's "Maven GAV" button, or install the jar locally with `mvn install:install-file`.
- **Test SQL vocabularies/directories**: directory templates and test datasources moved into
  `*-tests` jars in 2025 (e.g. `template-directory` is now in `nuxeo-platform-directory-sql` test-jar
  via `SQLDirectoryFeature`; the `vocabulary` schema is in `org.nuxeo.ecm.directory.types.contrib`).
  Studio-vocabulary test directories can be stubborn — if a feature-flag/vocabulary directory won't
  register, treat it as a known sharp edge and surface it to the user.
- **Login 500s after deploy — `ClassNotFoundException: org.apache.jsp.login_jsp`**: an overridden
  `login.jsp` (in `src/main/resources/web/nuxeo.war/`) fails server-side JSP compilation because it
  was copied from 2023 stock and uses removed APIs (`MobileBannerHelper`,
  `LoginScreenHelper.getDisplayMobileBanner`, Joda `DateTime`). The `ClassNotFoundException` is
  only the cascade — scroll up in the logs to the `Unable to compile class for JSP` error listing
  the real undefined symbols. See migrations.md "Overridden login.jsp".
- **`Unknown Document Scroller ... scroller=elastic` at runtime**: a bulk action contrib still
  declares `defaultScroller="elastic"`. On 2025 that name is only a deprecated alias contributed by
  `nuxeo-search-client-opensearch1`; when that bundle isn't deployed (common on Nuxeo Cloud), every
  submit of the action fails — typically surfacing as a failing cron/scheduled listener. Use
  `defaultScroller="search"`.
- **Version drift between pom and environment**: stack traces name the deployed jars (e.g.
  `nuxeo-platform-web-common-2025.19.14.jar`) — compare against the pom's
  `nuxeo.platform.version`; cloud envs are often ahead of the version you built against.
- **Broken Web UI after upgrade (blank app, lists/tasks not loading or sorting)**: an overridden
  `ui/index.jsp` still on the 2023 bootstrap. Signature errors: `NuxeoNonce is not defined`,
  `Nuxeo.UI.bundles is not iterable`, cascading `this.nxProvider.fetch is not a function` /
  `method _X not defined`. Rebase the override on the 2025 CSP-nonce bootstrap (recipe in
  web-ui.md). If the override lives in the **Studio** project's resources, don't edit it from the
  platform repo — tell the user to update (or remove) it in Studio Designer. A prettified
  `redefine-bundled.js` (`SyntaxError: Identifier 'e' has already been declared`) must be restored
  to its minified original.
- **Custom index mapping/settings silently ignored**: env-template overrides named
  `elasticsearch-doc-mapping.json` / `elasticsearch-doc-settings.json(.nxftl)` are dead names on
  2025 — the opensearch1 client reads `opensearch1-doc-mapping.json` /
  `opensearch1-doc-settings.json(.nxftl)`. Custom analyzers/fields quietly disappear (or index
  creation fails with `Unable to create index: nuxeo with client: search/default`). See
  migrations.md "Custom index mapping/settings files must be renamed".
