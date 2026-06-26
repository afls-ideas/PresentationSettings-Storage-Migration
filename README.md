# Intelligent Content (Presentation Settings) — Storage & Migration

> **TL;DR:** The Admin Console **Intelligent Content → Presentation Settings** tile stores its configuration as **data records** in the `LifeSciMetadata*` object family — **not** as metadata, and **not** in the `LifeSciConfigRecord` family used by most other Admin Console tiles. Because they are data records, you migrate them with data tooling (`sf data` / Data Loader), not a metadata deploy.
>
> **Verified:** 2026-06-26 against live orgs (Tooling/Data API describe + queries).

---

## Where it is stored

The settings live across **four related objects**:

| # | Object | Role | Example value |
|---|--------|------|---------------|
| 1 | **`LifeSciMetadataCategory`** | The category bucket | `Name = PresentationSettings` |
| 2 | **`LifeSciMetadataRecord`** | The settings record (one per scope: Org / Profile / User) | `Name = PresentationSettings_OrgLevel`, `RecordApiName = PresentationSettings_OrgLevel`, `IsOrgLevel = true`, `IsActive = true` |
| 3 | **`LifeSciMetadataFieldValue`** | The individual settings (children of the record) | 22 rows — e.g. `EnableLaserPointer = true`, `DisablePresenterTracking = false` |
| 4 | **`LifeSciMetadataAssignment`** | Profile / User assignment for non-org-level records | `AssignmentLevel = Profile`, `AssignedToId = <ProfileId/UserId>` |

### Relationship shape

```
LifeSciMetadataCategory (PresentationSettings)
  └── LifeSciMetadataRecord (PresentationSettings_OrgLevel)        ← via LifeScienceMetadataCategoryId
        ├── LifeSciMetadataFieldValue  (×22 settings)              ← via LifeScienceMetadataRecordId
        └── LifeSciMetadataAssignment  (0 for org-level;           ← via LifeSciMetadataRecordId
                                        1+ for profile/user overrides)
```

> ⚠️ **Field-name gotcha:** the lookup on **field-values** is `LifeScienceMetadataRecordId` (note the full "…ence…"), while on **assignments** it is `LifeSciMetadataRecordId` (short form) — they are **not** spelled the same.

### The 22 org-level settings

| # | `FieldName` | # | `FieldName` |
|---|-------------|---|-------------|
| 1 | `EnableDrawing` | 12 | `EnableRetakeWithCopyingLastResponses` |
| 2 | `EnableLaserPointer` | 13 | `AdvancedAccountSearchEnabled` |
| 3 | `DisablePresenterTracking` | 14 | `ContentRatingMainView` |
| 4 | `DisableParticipantTracking` | 15 | `ContentRatingBottomBar` |
| 5 | `GeolocationTrackingEnabled` | 16 | `NextBestMessageEnabled` |
| 6 | `LockdownEnabled` | 17 | `ShowSequenceNames` |
| 7 | `MobileLockFeatureDefaultIsLocked` | 18 | `VisualIndicator` |
| 8 | `EnableSendingByEmail` | 19 | `TrainingModeEnabled` |
| 9 | `ManageCustomPresentation` | 20 | `ExtendedSearchPresentationField` |
| 10 | `EnableControlsCollapse` | 21 | `ExtendedSearchPresPageProductField` |
| 11 | `EnableRetake` | 22 | `TargetingContext` |

Each value is held in a typed column on `LifeSciMetadataFieldValue` — `HasBooleanValue`, `TextValue`, `PicklistValue`, `IntegerValue`, `NumberValue`, or `LongTextValue` — selected by the row's `DataType`.

### Two common points of confusion

1. **Not `LifeSciConfigRecord`.** Most Admin Console tiles (Quick Actions, Custom Actions, DbSchema, Application Settings, …) store config in the `LifeSciConfig*` family. Intelligent Content does **not** — it uses the parallel `LifeSciMetadata*` family.
2. **The `DbSchema_PresentationSettings` red herring.** There **is** a `LifeSciConfigRecord` named `DbSchema_PresentationSettings` (`Type = CONFIGURATION`, `SObject = LifeSciMetadataRecord`, `Category = PresentationSettings`). That record only tells the **mobile sync engine** to sync the settings down to the device — it does **not** hold the settings themselves.

---

## These are DATA records (not metadata)

This is the key fact for anyone trying to move these settings between orgs.

**Evidence (from `describe` + Metadata API type listing):**

- All four objects report `createable = true`, `updateable = true`, `deletable = true`, and are queryable via the Data/SOQL API.
- The Metadata API exposes **`LifeSciConfigCategory`** and **`LifeSciConfigRecord`** as deployable metadata types — but **`LifeSciMetadata*` is *not* in the metadata-types list**.

**What that means:**

| | `LifeSciConfig*` (most tiles) | `LifeSciMetadata*` (Presentation Settings) |
|---|---|---|
| Nature | Metadata components | **Data records** |
| Move between orgs via | `sf project deploy` / change sets / package | **`sf data` / Data Loader (DML)** |
| Lives in source control as | Metadata XML | **Not metadata** — exported as data (CSV/JSON tree) |
| ID stability across orgs | Developer-name based | **Record Ids differ per org — match on API-name fields** |

So: **you cannot migrate Presentation Settings with a metadata deploy.** You move them as data, parent-before-child, re-resolving cross-org references.

---

## How to migrate them

Because it's a parent→child tree and record Ids differ per org, relate children to parents by **API-name** (`RecordApiName`, `AssignmentApiName`), not raw Id.

**Insert order:**

1. **`LifeSciMetadataCategory`** — usually already present in the target (seeded on package install). Query first; only insert if missing.
2. **`LifeSciMetadataRecord`** — set `LifeScienceMetadataCategoryId` to the *target* category Id; carry `IsOrgLevel`, `IsActive`, `Type`.
3. **`LifeSciMetadataFieldValue`** — the 22 settings; preserve `FieldName`, the typed value column, and `DataType`; point `LifeScienceMetadataRecordId` at the *target* record.
4. **`LifeSciMetadataAssignment`** — only for profile/user overrides. **`AssignedToId` will not match across orgs** — re-resolve by profile name / username in the target.

### Option A — `sf data export/import tree` (handles parent-child + ID remapping)

```bash
# Export from source
sf data export tree --target-org SOURCE \
  --query "SELECT Id, Name, RecordApiName, Type, IsOrgLevel, IsActive, LifeScienceMetadataCategoryId \
           FROM LifeSciMetadataRecord WHERE RecordApiName LIKE 'PresentationSettings%'" \
  --prefix pressettings --output-dir ./migrate

# Import into target
sf data import tree --target-org TARGET --plan ./migrate/pressettings-plan.json
```

> Confirm the child-relationship name for `LifeSciMetadataFieldValue` before relying on a single tree export. If it isn't traversable as a subquery, export each object to its own CSV and re-stitch by `RecordApiName` on import.

### Option B — scripted upsert (recommended for config)

Because this is **one org-level record + 22 field values** (plus any overrides), the lowest-risk path is usually to **upsert the field values onto the target org's existing `PresentationSettings_OrgLevel` record** (target orgs typically auto-seed it on install — so update, don't duplicate). Reserve the full tree migration for cases with many profile/user overrides.

### Option C — just re-set in the UI

For a handful of toggles with no overrides, re-entering them in the target org's **Admin Console → Intelligent Content → Presentation Settings** is faster and safer than any data move.

---

## Reference query

`LifeSciMetadata*` does **not** require the Tooling API (standard `sf data query` works):

```bash
sf data query --target-org <org> \
  -q "SELECT FieldName, DataType, HasBooleanValue, TextValue, PicklistValue, IntegerValue, NumberValue, LongTextValue \
      FROM LifeSciMetadataFieldValue \
      WHERE LifeScienceMetadataRecordId IN \
        (SELECT Id FROM LifeSciMetadataRecord WHERE RecordApiName LIKE 'PresentationSettings%')"
```

> **CLI gotcha:** `sf` prints an "update available" warning to stdout that breaks `json.load` when piped. Write `--json` output to a file first, then parse.
