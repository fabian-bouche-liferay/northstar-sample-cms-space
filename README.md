# nmg-corpus-batch

## Purpose

Reproduces the entire fictional Northstar Mobility Group RAG-evaluation
corpus — the `Northstar Enterprise Knowledge` CMS Space, its 9 folders, 8
picklists, 11 depot-scoped Content Structures, and 260 Content Entries — on
any Liferay instance via the Headless Batch Engine, so the corpus can be
recreated from source instead of only existing as live API state on one
instance.

## Type

`batch`, no framework (pure `batch-engine-data.json` files, no build step).

## Files

| File | Role |
| --- | --- |
| `client-extension.yaml` | CET + OAuth companion (`Liferay.Headless.Batch.Engine.everything`, `Liferay.Object.Admin.REST.everything`, `Liferay.Headless.Object.everything`, `Liferay.Headless.Admin.List.Type.everything`) |
| `batch/00-00-space.batch-engine-data.json` | The `NMG-KNOWLEDGE` Asset Library (Space) |
| `batch/00-01..09-picklist-*.batch-engine-data.json` | The 9 List Type Definitions (BusinessDomain, Country, ProductFamily, Audience, LifecycleStatus, Confidentiality, Severity, IssueStatus, ArticleType) |
| `batch/01-00-folders.batch-engine-data.json` | The 9 Object Entry Folders (stable, human-authored ERCs — `NMG-FOLDER-*`) |
| `batch/02-00..10-struct-*.batch-engine-data.json` | The 11 Content Structures (depot-scoped Object Definitions), one file each |
| `batch/03-00..10-entries-*.batch-engine-data.json` | The 260 Content Entries, one file per structure |

## Depends on

Nothing external — self-contained. `nmg-corpus-batch` is the sole artifact for the `NorthstarKnowledgeBase` concept in `solution-index.yaml`.

## Deploy and verify

```
blade gw ":client-extensions:nmg-corpus-batch:deploy"
```

Success signal: `STARTED nmgcorpusbatch_<version> [<bundle-id>]` in
`bundles/logs/liferay.<date>.log`, followed by 33 `Successfully deployed
batch engine file` lines (Space, 9 picklists, 9 folders, 11 structures, 11
entries files) with no `ERROR`/`Waiting for a service` lines in between.
Confirmed live end to end on `dxp-2026.q3.1`, 2026-09-02, on a genuinely
fresh instance: all 33 files processed in one continuous deploy, all 260
entries verified live afterward (`GET /o/c/<pluralPath>/scopes/NMG-KNOWLEDGE`
per structure, `totalCount` summing to 260).

## `taskItemDelegateName` must be a top-level sibling of `configuration.parameters`, not nested inside it

```json
"configuration": {
    "className": "com.liferay.object.rest.dto.v1_0.ObjectEntry",
    "taskItemDelegateName": "C_<ObjectName>",
    "parameters": {
        "createStrategy": "UPSERT",
        "scopeKey": "NMG-KNOWLEDGE"
    }
}
```

Nesting `taskItemDelegateName` inside `parameters` (the shape shown in this
workspace's own `manage-objects` skill reference example, itself unconfirmed)
is silently accepted with no validation error, but hangs the deploy
indefinitely: confirmed via bytecode disassembly of
`com.liferay.batch.engine.internal.unit.BatchEngineUnitProcessorImpl` and
`BatchEngineUnitReaderImpl` (in `com.liferay.batch.engine.service.jar`) —
`BatchEngineUnitConfiguration.getTaskItemDelegateName()` is read as a
dedicated field, not looked up inside the generic `parameters` map. When
nested inside `parameters` it silently deserializes to `null`, and the
processor then blocks forever (`BatchEngineUnitProcessorImpl: Waiting for a
service matching (...batch.engine.task.item.delegate.name=null...)`) waiting
for a generic/nameless delegate service that doesn't exist — a real service
tagged with the actual delegate name is registered and reachable the whole
time (confirmed: the identical import, issued via a direct
`POST /o/headless-batch-engine/v1.0/import-task/...` REST call instead of
file-based delivery, completes instantly regardless of nesting, since that
code path reads `taskItemDelegateName` from the query string, not this JSON
structure). Isolated with a 3-file minimal reproduction (one Space, one
structure, one entry) before being confirmed fixed at full scale.
