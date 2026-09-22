# nmg-corpus-batch

## Purpose

Reproduces the entire fictional Northstar Mobility Group RAG-evaluation
corpus — the `Northstar Enterprise Knowledge` CMS Space, its 9 folders, 8
picklists, 11 depot-scoped Content Structures, 260 Content Entries, and the
4 Search Blueprints the `AI Hub - Quickstart 2 - RAG` guide's Search Worker
agents reference — on any Liferay instance via the Headless Batch Engine, so
the whole corpus can be recreated from source in one deploy instead of only
existing as live API state on one instance. Each Content Structure's Object
Definition `className` is **pinned** to a fixed value in the batch files
themselves (see "Pinned Content Structure class names" below), which is what
lets the 4 Blueprints' `searchableAssetTypes` stay fully static across every
fresh deploy — no live resolution, no resync step, no separate creation
step, ever.

## Type

`batch` CET — pure `batch-engine-data.json` files, no build step, no
companion script. One `blade gw deploy` creates the entire CMS corpus *and*
the 4 Search Blueprints, in file order.

## Files

| File | Role |
| --- | --- |
| `client-extension.yaml` | CET + OAuth companion (`Liferay.Headless.Batch.Engine.everything`, `Liferay.Object.Admin.REST.everything`, `Liferay.Headless.Object.everything`, `Liferay.Headless.Admin.List.Type.everything`, `Liferay.Search.Experiences.REST.everything`) |
| `batch/00-00-space.batch-engine-data.json` | The `NMG-KNOWLEDGE` Asset Library (Space) |
| `batch/00-01..09-picklist-*.batch-engine-data.json` | The 9 List Type Definitions (BusinessDomain, Country, ProductFamily, Audience, LifecycleStatus, Confidentiality, Severity, IssueStatus, ArticleType) |
| `batch/01-00-folders.batch-engine-data.json` | The 9 Object Entry Folders (stable, human-authored ERCs — `NMG-FOLDER-*`) |
| `batch/02-00..10-struct-*.batch-engine-data.json` | The 11 Content Structures (depot-scoped Object Definitions), one file each — each carries a pinned `className` (see below) |
| `batch/03-00..10-entries-*.batch-engine-data.json` | The 260 Content Entries, one file per structure |
| `batch/04-00..03-blueprint-*.batch-engine-data.json` | The 4 Search Blueprints — `className: com.liferay.search.experiences.rest.dto.v1_0.SXPBlueprint`, `createStrategy: UPSERT` / `updateStrategy: UPDATE`, so a redeploy on an instance that already has them updates in place rather than duplicating |

The guide's own `AI Hub - Quickstart 2 - RAG/resources/*.json` files are a
**different, historical** set of 4 Blueprint JSON exports (different ERCs,
different titles, random-per-instance `className` values) from whichever
instance the guide's author originally built them on — kept for reference
only, never imported by this CET. Do not import them.

## Pinned Content Structure class names

Confirmed live and documented in full in `rules/object-api-quirks.md`: a
custom Object Definition's `className` (the `com.liferay.object.model.ObjectDefinition#<code>`
suffix everything else in this platform normally treats as random and
per-instance) can be **supplied explicitly at creation time** — via a plain
REST `POST`, or, confirmed separately, via a real Batch Engine import task —
as long as it starts with the literal prefix
`com.liferay.object.model.ObjectDefinition#` and is unique within the
company. Every one of this corpus's 11 `batch/02-*-struct-*.batch-engine-data.json`
files does exactly this:

| Content Structure | ERC | Pinned `className` |
| --- | --- | --- |
| NMGProduct | `NMG-STRUCT-PRODUCT` | `#NMGPRODUCT` |
| NMGTechSpec | `NMG-STRUCT-TECHSPEC` | `#NMGTECHSPEC` |
| NMGSupportArticle | `NMG-STRUCT-SUPPORT` | `#NMGSUPPORT` |
| NMGKnownIssue | `NMG-STRUCT-KNOWNISSUE` | `#NMGKNOWNISSUE` |
| NMGReleaseNote | `NMG-STRUCT-RELEASE` | `#NMGRELEASE` |
| NMGProcedure | `NMG-STRUCT-PROCEDURE` | `#NMGPROCEDURE` |
| NMGFAQ | `NMG-STRUCT-FAQ` | `#NMGFAQ` |
| NMGPolicy | `NMG-STRUCT-POLICY` | `#NMGPOLICY` |
| NMGSalesPlaybook | `NMG-STRUCT-SALES` | `#NMGSALES` |
| NMGCustomerCase | `NMG-STRUCT-CASE` | `#NMGCASE` |
| NMGCorporateArticle | `NMG-STRUCT-CORPORATE` | `#NMGCORPORATE` |

(Full prefix: `com.liferay.object.model.ObjectDefinition#NMGPRODUCT`, etc.)

**This means every fresh deploy of this CET — on any instance, at any
time — creates these 11 Content Structures with the exact same `className`
every time.** That is the property the 4 Search Blueprints below rely on:
`searchableAssetTypes` never needs to be resolved live, because it never
changes.

`className` is **create-time-only** — confirmed via `javap` that
`ObjectDefinitionLocalServiceImpl#updateCustomObjectDefinition` has no
`className` parameter at all. If a structure ever needs a different pinned
code, or an already-live instance needs to adopt pinning after the fact, the
only way is to delete and recreate the Object Definition (and its Content
Entries) — a real reprovision, not a redeploy.

## Search Blueprints — batch-imported alongside the corpus, fully static

`SXPBlueprint` (Search Experiences' Blueprint DTO,
`com.liferay.search.experiences.rest.dto.v1_0.SXPBlueprint`) is a real,
confirmed batch-importable resource — see `rules/ai-hub-agent-builder.md`
§ 5. Each of the 4 `batch/04-*-blueprint-*.batch-engine-data.json` files
wraps one complete Blueprint body: stable, human-readable
`externalReferenceCode`, and `configuration.generalConfiguration.searchableAssetTypes`
already filled in with the pinned `className` values from the table
above — no placeholder, no live resolution step, no separate creation call,
because those values are now fixed platform facts, not something this
instance happens to have generated.

Structure-to-Blueprint mapping:

| Blueprint | ERC | Content Structures |
| --- | --- | --- |
| Northstar product & engineering | `NMG-BLUEPRINT-PRODUCT-ENGINEERING` | `NMG-STRUCT-PRODUCT`, `NMG-STRUCT-KNOWNISSUE`, `NMG-STRUCT-TECHSPEC`, `NMG-STRUCT-RELEASE` |
| Northstar Support & operations | `NMG-BLUEPRINT-SUPPORT-OPERATIONS` | `NMG-STRUCT-SUPPORT`, `NMG-STRUCT-FAQ` |
| Northstar Corporate & Policy | `NMG-BLUEPRINT-CORPORATE-POLICY` | `NMG-STRUCT-CORPORATE`, `NMG-STRUCT-POLICY`, `NMG-STRUCT-PROCEDURE` |
| Sales & Customer | `NMG-BLUEPRINT-SALES-CUSTOMER` | `NMG-STRUCT-SALES`, `NMG-STRUCT-CASE` |

**`NMG-STRUCT-PROCEDURE` sits under Corporate & Policy, not Support &
Operations.** An earlier pass at this mapping guessed Support & Operations
from the domain's prose description ("troubleshooting, remediation,
installation, maintenance") and got it wrong — confirmed by decoding the
`searchableAssetTypes` of 4 Blueprints that had already been hand-built on
this instance (via the Search Experiences UI, before this corpus adopted
pinned class names) before this table was written. Trust the table above,
not a fresh guess from the domain descriptions in the guide's own README.

The 4 Blueprint ERCs were deliberately renamed from their original random
UUIDs (`c4736127-...`, etc.) to the `NMG-BLUEPRINT-*` form above — easier to
read, type, and paste into an AI Hub Search Worker's
`blueprintExternalReferenceCode` field than a UUID.

`configuration.generalConfiguration.scope: ["NMG-KNOWLEDGE"]` never needed
any of this treatment — a Space's external reference code is a stable,
human-chosen identifier (`rules/cms-catalog.md`), unlike an Object
Definition's `className`, so that value was always portable across
instances.

**`createStrategy: UPSERT` / `updateStrategy: UPDATE`** (the same pair every
other batch file in this CET uses) means a redeploy on an instance that
already has these 4 Blueprints (matched by `externalReferenceCode`) updates
them in place rather than creating duplicates — confirmed live: after
already having 4 live Blueprints with these ERCs, a second full `blade gw
deploy` left `totalCount` at 4, with every `searchableAssetTypes` array and
numeric `id` unchanged.

## Depends on

Nothing external — self-contained. `nmg-corpus-batch` is the sole artifact
for the `NorthstarKnowledgeBase` concept, and the sole artifact for the
`NorthstarSearchBlueprints` concept, in `solution-index.yaml`.

## Deploy and verify

```bash
blade gw ":client-extensions:nmg-corpus-batch:deploy"
```

Success signal: `STARTED nmgcorpusbatch_<version> [<bundle-id>]` in
`bundles/logs/liferay.<date>.log`, preceded by 37 `Successfully deployed
batch engine file` lines (Space, 9 picklists, 9 folders, 11 structures, 11
entries files, 4 Blueprints) with no `ERROR` lines in that window.

Confirmed live end to end on `dxp-2026.q3.1`, 2026-09-22:

- On a genuinely fresh instance: all 33 corpus files processed in one
  continuous deploy, all 11 structures carrying their pinned `className`
  (table above), and all 260 entries verified live afterward
  (`GET /o/c/<pluralPath>/scopes/<space-scope-id>` per structure, `totalCount`
  summing to 260).
- The 4 Blueprint batch files (`04-00`..`04-03`), added in a later session,
  redeployed cleanly onto that same instance alongside the 33 corpus files
  (37 total) with the new `Liferay.Search.Experiences.REST.everything` OAuth
  scope in place — no bundle failure, no OAuth registration error. A
  follow-up `GET /o/search-experiences-rest/v1.0/sxp-blueprints` confirmed
  exactly 4 Blueprints (ids unchanged from an earlier one-time manual
  creation, `totalCount: 4` — no duplicates), each with its friendly
  `NMG-BLUEPRINT-*` ERC and a `searchableAssetTypes` array matching the
  pinned `className` values one for one.

No separate step, script, or manual `curl` loop is needed on any subsequent
redeploy or reprovision of this CET, on any instance — the Blueprints are
created (or, if already present, updated in place) by the same `blade gw
deploy` that creates the rest of the corpus.
