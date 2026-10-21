# Nuxeo 2023 → 2025 migrations reference

Exhaustive, concrete before→after changes. Apply only the ones the project actually triggers
(grep first). "Verify" notes say how to confirm a class/artifact location against the real 2025
artifacts when in doubt (decompile a cached jar with `javap`, or grep the Nuxeo 2025 source).

---

## Versions & build

### Parent vs platform version (critical)
`nuxeo-parent` releases as a **2-part** version (e.g. `2025.16`). The platform artifacts
(`nuxeo-runtime`, `nuxeo-core-*`, …) release as a **3-part** version (e.g. `2025.16.13`). The
parent's own `<parent>` is `org.nuxeo:nuxeo-ecm:<3-part>` and the BOM versions every artifact via
`${nuxeo.platform.version}`. So:

```xml
<!-- root pom -->
<parent>
  <groupId>org.nuxeo</groupId>
  <artifactId>nuxeo-parent</artifactId>
  <version>2025.16</version>          <!-- was 2023.x : 2-part build parent -->
</parent>
...
<properties>
  <nuxeo.platform.version>2025.16.13</nuxeo.platform.version>      <!-- 3-part! not 2025.16 -->
  <nuxeo.distribution.version>2025.16.13</nuxeo.distribution.version>
  <nuxeo.target.version>2025.16.13</nuxeo.target.version>
  <maven.compiler.release>21</maven.compiler.release>             <!-- was 17 -->
</properties>
```
To find the 3-part version: read the cached `nuxeo-parent-<2-part>.pom`'s `<parent><version>`.
**Setting these to the 2-part `2025.16` makes every nuxeo artifact 404.**

### Java 21
`maven.compiler.release` 17 → 21 in the root pom and in **every module** that overrides it
(`<source>17</source><target>17</target>` or `<release>17</release>` inside `maven-compiler-plugin`).
Build with `JAVA_HOME=$(/usr/libexec/java_home -v21)`.

### Marketplace package (`package.xml`)
```xml
<platform>lts-2023.*</platform>   →   <platform>lts-2025.*</platform>
```

### CI (`.github/workflows/*.yml`, Jenkinsfile)
JDK `17` → `21`; Maven `3.8.x` → `3.9.9`.

### Studio dependency
Maven versions can't contain `/`. For a Studio branch (e.g. `feature/2025`), use the sanitized
GAV from Studio → Branch Management → "Maven GAV" (often `2025-SNAPSHOT`). If resolution 401s,
download the jar and `mvn install:install-file -DgroupId=nuxeo-studio -DartifactId=<Project>
-Dversion=<ver> -Dpackaging=jar -Dfile=<path>`, then build with `-nsu`.

### commons-io (silent runtime breaker)
Remove any explicit `commons-io` pin older than 2.17 — let the BOM provide it (2.21.0). An old
pin compiles fine but fails at **runtime** with
`NoClassDefFoundError: org/apache/commons/io/input/UnsynchronizedBufferedReader` (seen in
Nuxeo's `SQLDirectory`, and as test **errors** in CI: `SomeTest » NoClassDefFound ...`).

**The pin comes back with merges.** Removing it on the upgrade branch is not enough: merging
mainline (`master`/`release-candidate`) into the upgrade branch later re-resolves the root pom and
can silently reintroduce the old `<dependencyManagement>` block — yielding a half-merged pom
(2025 `nuxeo-parent` + 2023-era pins) that fails only in CI. After **every** merge from mainline:
```bash
mvn dependency:tree -pl <some-module> -Dincludes=commons-io   # must show >= 2.17 (e.g. 2.21.0)
```
and re-check the other root-pom migrations survived (compiler release 21, plugin pins).

Diagnosing "CI fails, local passes":
- the CI log literally shows the bad resolution: `Downloaded ...: commons-io-2.14.0.jar`;
- compare the CI reactor header version (`Building <project> X.Y.Z`) against the local pom — a
  mismatch means CI builds a different ref (often the merged remote branch while local is behind);
- old plugin versions in the CI log (e.g. `maven-enforcer-plugin:1.4.1`) are another tell that
  mainline's pom section won the merge.

---

## Source code

### Jakarta EE 10 (`javax` → `jakarta`)
Every Jakarta-EE namespace moves. These are the ones seen in practice:
```
javax.inject.*      → jakarta.inject.*
javax.ws.rs.*       → jakarta.ws.rs.*        (JAX-RS / WebEngine)
javax.servlet.*     → jakarta.servlet.*
javax.mail.*        → jakarta.mail.*
```
Mechanical sweep (safe — these four are all Jakarta EE):
```bash
for f in $(grep -rlE "javax\.(inject|ws\.rs|servlet|mail)" --include=*.java .); do
  perl -0pi -e 's/javax\.(inject|ws\.rs|servlet|mail)/jakarta.$1/g' "$f"; done
```
Do **not** move JDK `javax.*` (e.g. `javax.naming`, `javax.crypto`, `javax.xml`, `javax.security`).
Then the servlet-api dependency:
```xml
<groupId>javax.servlet</groupId><artifactId>javax.servlet-api</artifactId><version>3.0.1</version>
  → <groupId>jakarta.servlet</groupId><artifactId>jakarta.servlet-api</artifactId>  <!-- 6.0.0 via BOM -->
```

### Joda-Time → java.time
Joda is no longer in the distribution. Typical usage is just the current year/date:
```java
import org.joda.time.DateTime;     →  import java.time.Year;  (or java.time.LocalDate)
new DateTime().getYear()           →  Year.now().getValue()
DateTime d = new DateTime();        →  LocalDate d = LocalDate.now();   // d.getYear() unchanged
```

### Lombok (breaks on JDK 21)
Old Lombok throws `NoSuchFieldError: ... JCImport ... qualid` during compile. If usage is light,
remove it. Common case `@SneakyThrows` on a method that can't declare checked exceptions
(e.g. `AbstractWork.work()`): wrap the body and rethrow unchecked.
```java
@SneakyThrows
public void work() { ...body that throws checked... }
  →
public void work() {
    try { ...body... }
    catch (NuxeoException e) { throw e; }
    catch (Exception e) { throw new NuxeoException(e); }
}
```
Remove `import lombok.*;`. (Alternative: bump Lombok ≥ 1.18.30, but removing is cleaner if usage is small.)

### jetbrains @NotNull
Usually transitive and gone in 2025. Remove `import org.jetbrains.annotations.NotNull;` and the
`@NotNull` usages (they're hints only — behavior-neutral).

### Elasticsearch core removed → nuxeo-core-search
In poms (compile **and** `test-jar`):
```xml
<groupId>org.nuxeo.elasticsearch</groupId><artifactId>nuxeo-elasticsearch-core</artifactId>
  → <groupId>org.nuxeo.ecm.core</groupId><artifactId>nuxeo-core-search</artifactId>
```

**Deployment — search packages (IF on OpenSearch).** The 2025 search backend is modular: the
distribution no longer bundles a search client by default. **If** your deployment uses **OpenSearch**
(the typical case), you must explicitly deploy the OpenSearch client packages — otherwise search and
audit silently come up unconfigured. Check the current Nuxeo docs for the exact set, but at minimum:
- **`nuxeo-search-client-opensearch1`** — the repository search backend
- **`nuxeo-audit-opensearch1`** — the audit backend
Add them to the marketplace package `<require>`s / the distribution's installed packages (not a Maven
compile dependency). (If you run a different search engine — e.g. MongoDB Atlas Search — deploy the
corresponding client instead.)

Symptom of a missing audit backend, in the server startup banner:
`Component Loading Warnings: * In-Memory implementation for audit backend "default" is ONLY for
testing purpose.` — audit is silently running in memory; deploy the audit client package.

**Mixed profiles**: if only some environments run OpenSearch (e.g. cloud on OpenSearch, local dev
on the built-in `repository` search client), do NOT hard-require the opensearch packages in
`package.xml` — that installs the client on every profile, including ones with no cluster.
Provision them per environment instead (e.g. the Nuxeo Cloud env package list), and keep any
OpenSearch-specific config only in the templates of the profiles that use it.

### Custom index mapping/settings files must be renamed (`elasticsearch-doc-*` → `opensearch1-doc-*`)
If the project overrides the search index settings or mapping via files in an env template's
`nxserver/config/` (custom analyzers, extra field mappings, shards/replicas/max-result-window
tuning), the 2023-era file names are tied to the removed Elasticsearch stack and are **silently
ignored** by the 2025 opensearch1 client. Rename them after the client (`opensearch1`):
```
nxserver/config/elasticsearch-doc-settings.json(.nxftl)  →  nxserver/config/opensearch1-doc-settings.json(.nxftl)
nxserver/config/elasticsearch-doc-mapping.json           →  nxserver/config/opensearch1-doc-mapping.json
```
Symptoms of the stale names: the index is created with **default** settings/mapping (custom
analyzers and field mappings vanish — search behaves subtly wrong), or index creation fails with
`Unable to create index: nuxeo with client: search/default` when mapping references analyzers the
default settings don't define. If the project has no such override files, there is nothing to do —
the opensearch1 package ships the stock settings/mapping.

While renaming, also update the content:
- settings tunables use the new property names, e.g.
  `${(nuxeo.search.client.default.opensearch1.settings.numberOfShards)!1}`,
  `...numberOfReplicas`, `...indexTranslogDurability` (old `elasticsearch.*` settings tokens are dead);
- the mapping may need 2025 stock additions (e.g. `ecm:ancestorId`,
  `collection:documentIds`/`collectionMember:collectionIds` as `keyword`) — re-diff against the
  stock 2025 mapping rather than keeping a stale 2023 copy.

Things that are **not** renamed: the `elasticsearch.override.pageproviders` property (still honored
on 2025) and project-defined properties like a custom `elasticsearch.max.result.window` used only
by the project's own templates.

### WebEngine FormData / getForm() removed
`WebContext.getForm()` and `org.nuxeo.ecm.webengine.forms.FormData` are gone. A `@POST` handler
that read form values now receives a `MultivaluedMap`, and overrides return `Template`:
```java
import org.nuxeo.ecm.webengine.forms.FormData;   // remove
import jakarta.ws.rs.core.MultivaluedMap;         // add
import org.nuxeo.ecm.webengine.model.Template;    // add

public Object validateTrialForm() {                       // old
    FormData f = getContext().getForm();
    String x = f.getString("RequestId");
}
  →
public Template validateTrialForm(MultivaluedMap<String,String> parameters) {   // new
    String x = parameters.getFirst("RequestId");
}
```
Check the parent class signatures (e.g. `UserInvitationObject`): several methods changed return
type `Object` → `Template`. For request-scoped reads outside a form handler, use
`getContext().getRequest().getParameter(...)` (jakarta `HttpServletRequest`).

### TaskObject package move (routing REST)
```
org.nuxeo.ecm.restapi.server.jaxrs.routing.TaskObject
  → org.nuxeo.ecm.restapi.server.routing.TaskObject     (drops ".jaxrs")
```

### nuxeo-platform-comment-core merged
`nuxeo-platform-comment-core` no longer exists; its classes merged into `nuxeo-platform-comment`
(which has both jar and test-jar in the BOM). Remove `-core` deps; if `nuxeo-platform-comment` is
already a dependency, the merged classes come with it.

### commons-lang → commons-lang3
```
import org.apache.commons.lang.StringUtils;  →  import org.apache.commons.lang3.StringUtils;
<groupId>commons-lang</groupId><artifactId>commons-lang</artifactId>
  → <groupId>org.apache.commons</groupId><artifactId>commons-lang3</artifactId>
```

### PDFBox 2 → 3
```java
import org.apache.pdfbox.pdfparser.PDFParser;   →  import org.apache.pdfbox.Loader;
PDDocument.load(inputStream)                    →  Loader.loadPDF(file)   // try-with-resources
```
Add `org.apache.pdfbox:pdfbox-io` if needed.

### commons-logging (optional)
`commons-logging` is still managed in the 2025 BOM, so migration is **not required**. Only migrate
to log4j2 if you hit a runtime classpath issue:
```java
org.apache.commons.logging.Log / LogFactory.getLog(X.class)
  →  org.apache.logging.log4j.Logger / LogManager.getLogger(X.class)
```

### Overridden login.jsp (mobile banner removed) — runtime-only failure
If the project overrides server JSPs (`src/main/resources/web/nuxeo.war/login.jsp` is the common
one, for a branded login page), a 2023-based copy **fails JSP compilation on the 2025 server** —
Maven can't catch it. Symptom: every login 500s; the log shows
`ClassNotFoundException: org.apache.jsp.login_jsp` (the cascade) and, earlier, the real error:
`Unable to compile class for JSP` with `MobileBannerHelper resolves to a package` /
`getDisplayMobileBanner(HttpServletRequest) is undefined for the type LoginScreenHelper`.

2025 dropped the Nuxeo mobile app: `org.nuxeo.ecm.platform.web.common.MobileBannerHelper` and
`LoginScreenHelper.getDisplayMobileBanner()` no longer exist. **Do not rebase a heavily customized
JSP onto stock** — diff it against the stock 2025 `login.jsp` and surgically remove the dead code.
Stock reference: `nuxeo-platform-web-common-<ver>.jar` → `web/nuxeo.war/login.jsp` (same file in
the nuxeo-lts source at `modules/platform/nuxeo-platform-web-common/src/main/resources/web/nuxeo.war/login.jsp`):
```bash
unzip -p ~/.m2/repository/org/nuxeo/ecm/platform/nuxeo-platform-web-common/<ver>/nuxeo-platform-web-common-<ver>.jar \
  web/nuxeo.war/login.jsp > /tmp/stock-login.jsp
diff -w /tmp/stock-login.jsp <project>/src/main/resources/web/nuxeo.war/login.jsp
```
Remove, mirroring stock 2025:
- the `MobileBannerHelper` import and **every** `<% if (displayMobileBanner) { %>` block (the
  `mobile-banner.js` script include, the `#mobileBanner`/`mobileAppLink` CSS, the banner HTML+JS
  near the footer, and the iOS/Android user-agent script) plus the `displayMobileBanner` variable;
- the app-store links `<div class="mobile-apps">` and its CSS;
- imports that were only used by the banner block (`StringUtils`, `StringEscapeUtils`,
  `RepositoryManager`);
- Joda (also gone — see Joda-Time above): `new DateTime().toString("Y")` →
  `Integer.toString(Year.now().getValue())` with `<%@ page import="java.time.Year"%>`.
Verify: `grep -iE "MobileBanner|joda|displayMobileBanner" login.jsp` returns nothing, and the
remaining diff vs stock is only the project's intentional customizations. Check the other
overridden JSPs (`ui/index.jsp`, `login_welcome.jsp`, …) for the same patterns.

### Bulk action `defaultScroller="elastic"` — broken in production too
On 2025 the scroller registry is: `repository` (nuxeo-core-bulk), **`search`** (nuxeo-core-search;
the canonical document scroller, routes to the configured search backend), `repository_search`,
and `elastic` — which survives **only** as a deprecated alias contributed by the
`nuxeo-search-client-opensearch1` bundle ("deprecated since 2025.0, use opensearch scroller
instead"). If that bundle isn't deployed — common on Nuxeo Cloud 2025 — every submit of the action
fails:
```
NuxeoException: java.lang.IllegalArgumentException: Unknown Document Scroller for command:
  BulkCommand[... scroller=elastic ...]
```
(typically surfacing as a failing scheduled/cron listener, e.g. via `BulkService.submit`).

Fix the **production** contributions:
```xml
<action name="..." inputStream="..." httpEnabled="true" defaultScroller="elastic"/>
  →  <action name="..." inputStream="..." httpEnabled="true" defaultScroller="search"/>
```
`search` also exists under `CoreSearchFeature` in tests (its `searchIndex` option defaults to
`${nuxeo.search.client.default.scroller.index.name:=repository}`), so tests pass without any shim.

### Other removed APIs (grep; replace per Nuxeo upgrade notes)
`CoreSession#close()`, `Event#isLocal/isPublic`, `DownloadService#downloadBlobStatus`,
`AsyncBlob`, `FileManager#createDocumentFromBlob` → `createOrUpdateDocument(FileImporterContext)`,
`Batch#addChunk`, `CommentManager#getComments(doc,doc)` → `getComments(doc)`, `LifeCycleTrashService`
→ `PropertyTrashService`, `WorkManager#find/listWork/listWorkIds`, `org.mockito.Matchers` →
`org.mockito.ArgumentMatchers`. (Verify each against the 2025 jar before assuming.)

---

## Tests

### Search test feature
```
org.nuxeo.elasticsearch.test.RepositoryLightElasticSearchFeature
  → org.nuxeo.ecm.core.test.CoreSearchFeature        (in nuxeo-core-test)
```

### TransientStoreFeature package move
```
org.nuxeo.transientstore.test.TransientStoreFeature  →  org.nuxeo.transientstore.TransientStoreFeature
```

### Mockito
```
org.mockito.Matchers.anyX  →  org.mockito.ArgumentMatchers.anyX
```

### Servlet 6 HttpSession — removed methods
A mock implementing `jakarta.servlet.http.HttpSession` must drop the methods removed in Servlet 6
(they had `@Override` and no longer compile): `getSessionContext()` (+ `HttpSessionContext` import),
`getValue`, `getValueNames`, `putValue`, `removeValue`.

### CollectionFeature / SQLDirectoryFeature dependencies
Some test features that were transitive in 2023 need explicit (BOM-managed) test-jar deps in 2025:
```xml
<dependency><groupId>org.nuxeo.ecm.platform</groupId>
  <artifactId>nuxeo-platform-collections-core</artifactId><scope>test</scope></dependency>
<dependency><groupId>org.nuxeo.ecm.platform</groupId>
  <artifactId>nuxeo-platform-collections-core</artifactId><type>test-jar</type><scope>test</scope></dependency>
```
`CoreSearchFeature` (was `RepositoryLightElasticSearchFeature`) lives in `nuxeo-core-test`.

### mock-javamail → SmtpMailServerFeature
`org.jvnet.mock-javamail` is gone in 2025 (and its transitive `javax.mail:mail:1.4` is **banned**
by the enforcer). Remove the dependency and migrate the tests:
- `@Features(... org.nuxeo.mail.SmtpMailServerFeature.class)`; inject
  `SmtpMailServerFeature.MailsResult mailsResult;`
- `Mailbox.clearAll()` → `mailsResult.clearMails()`.
- If the code under test sends raw SMTP from `mail.transport.*` properties, point the port at the
  in-memory server and disable TLS/auth (set **all** props the sender reads — a missing one NPEs in
  `Properties.put`):
  ```java
  Framework.getProperties().setProperty("mail.transport.host", "localhost");
  Framework.getProperties().setProperty("mail.transport.port",
          Framework.getProperty("nuxeo.test.mail.smtp.port"));   // dumbster's random port
  Framework.getProperties().setProperty("mail.transport.auth", "false");
  Framework.getProperties().setProperty("mail.transport.usetls", "false");
  Framework.getProperties().setProperty("mail.transport.ssl.protocol", "TLSv1.2"); // keep if sender reads it
  ```
- Read results via `mailsResult.getMails()` / `getMailsByRecipient(addr)` / `getMailsBySubject(s)`.
  `MailMessage` has `getSubject/getContent/getSenders/getRecipients`.
- **Multi-recipient gotcha**: for one email sent to several recipients, the server returns the
  `To:` header as a single concatenated string, so `getMailsByRecipient` exact-matches only the
  first. Assert with `String.join(",", mail.getRecipients()).contains(addr)` instead.

### DublinCore contributor overwrite (test data)
If a test sets `dc:contributors` and then asserts on it, the DublinCore listener overwrites it with
the saving principal on save. Preserve the explicit value:
```java
doc.putContextData("disableDublinCoreListener", true);
doc.setPropertyValue("dc:contributors", contributors);
session.saveDocument(doc);
```

### Bulk scroller `elastic` in tests — SUPERSEDED
Earlier versions of this skill kept `defaultScroller="elastic"` in production ("production runs on
Elasticsearch") and deployed a test-only `elastic`→`RepositoryScroll` alias contrib. **That premise
is wrong on 2025**: `elastic` fails on real 2025 deployments too (see "Bulk action
`defaultScroller="elastic"` — broken in production too" above). Change the production contribs to
`defaultScroller="search"` instead — it resolves under `CoreSearchFeature` in tests as well, so the
alias contrib and its `@Deploy` lines are unnecessary (an existing alias shim is harmless but can
be deleted once no contrib references `elastic`).

### Quarantining unmigratable tests
For large tests on removed infra (e.g. the Jersey 1.x REST client: `com.sun.jersey.*`,
`org.nuxeo.ecm.restapi.test.BaseTest`, `CloseableClientResponse`, `org.nuxeo.jaxrs.test` — all
replaced by `RestServerFeature` + the new HTTP test client), it's acceptable to exclude them from
test compilation and track them, rather than block the upgrade:
```xml
<plugin><artifactId>maven-compiler-plugin</artifactId>
  <configuration>
    <!-- TODO(2025-upgrade): migrate to RestServerFeature + new HTTP test client -->
    <testExcludes><testExclude>**/SomeLegacyRestClientTest.java</testExclude></testExcludes>
  </configuration>
</plugin>
```

### SQL directories / vocabularies in tests
A test that opens a Studio **vocabulary** directory (e.g. a feature-flag vocab) fails on 2025 with
`DirectoryException: No directory registered with name: <voc>` unless the whole 2025 directory chain
is deployed. The 2023-era test contrib usually targets `SQLDirectoryFactory` and
`extends="template-directory"` — **both wrong for 2025**. The full recipe:

1. **Backend + `template-directory`** — add `SQLDirectoryFeature` to `@Features` and its test-jar dep:
   ```xml
   <dependency><groupId>org.nuxeo.ecm.platform</groupId>
     <artifactId>nuxeo-platform-directory-sql</artifactId><type>test-jar</type><scope>test</scope></dependency>
   ```
2. **`vocabulary` schema** — `@Deploy("org.nuxeo.ecm.directory.types.contrib")`.
3. **`template-vocabulary`** — defined in `nuxeo-platform-default-config` (NOT deployed in tests).
   Don't pull in all of default-config; just declare it in the test contrib (it's tiny).
4. **The vocab contrib** — target **`GenericDirectory`**, extend **`template-vocabulary`**, and require
   `org.nuxeo.runtime.started`:
   ```xml
   <component name="<your.pkg>.vocab.tests">
     <require>org.nuxeo.runtime.started</require>
     <extension target="org.nuxeo.ecm.directory.GenericDirectory" point="directories">
       <directory name="template-vocabulary" template="true" extends="template-directory">
         <schema>vocabulary</schema><idField>id</idField>
       </directory>
       <directory name="<VOC_NAME>" extends="template-vocabulary">
         <createTablePolicy>on_missing_columns</createTablePolicy>
         <dataLoadingPolicy>skip_duplicate</dataLoadingPolicy>
         <table>studio_vocabulary_<VOC_NAME></table>
         <dataFile>data/<VOC_NAME>.csv</dataFile>
       </directory>
     </extension>
   </component>
   ```
   Deploy it via the **test bundle** name (`<artifactId>-tests:<file>.xml`, per the test
   `META-INF/MANIFEST.MF` `Bundle-SymbolicName`), with a `data/<VOC_NAME>.csv` holding the header
   `id,label,obsolete,ordering`.

Alternatively, if the Studio bundle itself defines the vocab, deploy the Studio directory
contributions with a custom `TargetExtensions` subclass for the `directories` point in
`@PartialDeploy` (Nuxeo only ships `ContentModel`/`ContentTemplate`/`Automation`, so you write the
directory one). After changes, `mvn clean test` — stale `target/test-classes` can otherwise cause
spurious `MethodSorter`/`null file` errors.
