# Objective 1: API path confirmation

This objective answers one question: which IBM MDM REST API path should ADF use to discover changed player-to-group mappings in a batch?

## Working conclusion

Start with IBM MDM advanced export as the batch discovery path. Use entity records or entity history only after export has identified candidate entity IDs that need enrichment.

This keeps the POC batch-oriented and avoids scanning Neo4j directly.

## Confirmed so far

| Item | Status | Notes |
| --- | --- | --- |
| Cloud Pak for Data version | Confirmed | Target version is 5.3.x. |
| API reference baseline | Confirmed for planning | Use the IBM Master Data Management API reference plus the Cloud Pak for Data 5.3.x MDM docs. Validate endpoint details in the target environment's API explorer before documenting final request values. |
| MDM API service grouping | Confirmed from IBM docs | IBM documents Configuration, Entity Maintenance, Job, Matching, Migration, and Model services. |
| Relevant Entity Maintenance endpoints | Confirmed from IBM docs | IBM documents `data_exports`, entity records, entity history, export status, and export download endpoints. Confirm availability in the target API explorer. |
| Timestamp property for entity deltas | Confirmed from IBM docs | IBM documents `entity_last_updated` for entity exports from a point in time. |
| Timestamp property for record deltas | Confirmed from IBM docs | IBM documents `record_last_updated` for record exports from a point in time. |
| Direct Neo4j access | Confirmed out of scope | IBM will not allow direct Neo4j access. ADF must use supported MDM REST APIs. |

## Still open

| Item | How to confirm | Notes |
| --- | --- | --- |
| Advanced export enabled in the target environment | Open the target MDM data API explorer and confirm `POST /mdm/v1/data_exports` is available. Then submit a small test export from a narrow time window, confirm the job is created, poll `GET /mdm/v1/data_exports/{export_id}`, and confirm `GET /mdm/v1/data_exports/{export_id}/download` returns a file after success. | The user or IBM environment owner must run this in the actual tenant. |
| Player entity type | Confirm from the MDM data model or API explorer. | `person_entity` is only an example until confirmed. |
| Entity records endpoint availability | Confirm `GET /mdm/v1/entities/{id}/records` in the target API explorer. | This is not the primary extraction path. Use it only for enrichment if the batch export lacks required fields. |
| Entity history endpoint availability | Confirm `GET /mdm/v1/entities/{id}/history` in the target API explorer. | This is also enrichment only, not batch discovery. |
| Export record limit | Confirm with IBM or the target API explorer, then test with a controlled export near the expected limit. | The public advanced export page checked here does not state a 10,000 or 100,000 record cap. Treat the limit as unknown until the target environment confirms it. |
| Rate limits | Confirm with IBM or the target environment owner. | No project-specific rate limit has been confirmed. |

## What to confirm

| Check | Why it matters | Expected POC answer |
| --- | --- | --- |
| Cloud Pak for Data version | Confirms which IBM docs apply. | Confirmed: target environment is 5.3.x. |
| MDM API reference applies | Confirms endpoint paths and required query parameters. | Use IBM MDM API reference and validate in the 5.3.x target API explorer. |
| MDM APIs are grouped into expected services | Confirms the right API area to use. | Confirmed from IBM docs. Use Entity Maintenance/data APIs for this objective. |
| Entity Maintenance can retrieve records, entities, history, and exports | Confirms the required API family exists. | Confirmed from IBM docs; still validate target-environment availability. |
| Advanced export is enabled | Provides batch discovery of changed candidates. | Open: confirm `data_exports` endpoints are available in the target environment. |
| Export can filter by update timestamp | Supports watermark-based extraction. | Confirmed from IBM docs: use `entity_last_updated` for entity exports or `record_last_updated` for record exports. |
| Export can filter to player entity type | Keeps the batch small and relevant. | Use the confirmed player entity type, such as `person_entity` only if that is correct. |
| Entity records endpoint is available | Provides current member records when export rows are incomplete. | `GET /mdm/v1/entities/{id}/records` is available. |
| Entity history endpoint is available | Provides change context for one candidate entity. | `GET /mdm/v1/entities/{id}/history` is available. |
| API limits are known | Prevents oversized batches. | Open: confirm whether the export cap is 10,000, 100,000, or tenant-specific. |
| Neo4j direct access is unnecessary | Keeps the solution aligned with supported APIs. | Confirmed: direct Neo4j access is not allowed. |

## Recommended API path

1. Submit an advanced export request for entities updated in the bounded watermark window.
2. Poll export status until IBM MDM reports the export is complete or failed.
3. Download the export file.
4. Store the raw export content in Snowflake.
5. Identify candidate player entities from the export.
6. For candidate entities missing required fields, call entity records or entity history.
7. Store those enrichment responses in Snowflake too.

Entity records and entity history are not the main extraction mechanism. They are only follow-up calls for the subset of exported candidate entities where the batch export does not contain enough information to produce the curated delta row.

## How to confirm advanced export is enabled

1. Open the target MDM data API explorer for the 5.3.x environment.
2. Find the Entity Maintenance or data API section.
3. Confirm `POST /mdm/v1/data_exports` exists.
4. Confirm `GET /mdm/v1/data_exports/{export_id}` exists.
5. Confirm `GET /mdm/v1/data_exports/{export_id}/download` exists.
6. Confirm the user or service account has the required read permission, expected to include `mdm-oc.data.read`.
7. Submit a small export request for a narrow window, such as one hour.
8. Poll the export job until it succeeds or fails.
9. Download the export file and record the observed row count, file format, file size, and job duration.

## Example request shape to validate

This example is intentionally generic. Replace `person_entity`, timestamps, file name, and property names with values confirmed from the target MDM environment.

```json
{
  "export_type": "ENTITY",
  "file_name": "player_entity_delta_20260528T140000Z",
  "format": "JSON",
  "search_criteria": {
    "search_type": "entity",
    "filters": [
      {
        "type": "ENTITY",
        "values": ["person_entity"]
      }
    ],
    "query": {
      "operation": "and",
      "expressions": [
        {
          "property": "entity_last_updated",
          "condition": "GREATER_THAN_EQUAL",
          "value": "2026-05-28T13:00:00Z"
        },
        {
          "property": "entity_last_updated",
          "condition": "LESS_THAN",
          "value": "2026-05-28T14:00:00Z"
        }
      ]
    }
  }
}
```

## What not to decide yet

- Do not choose the final player entity type until the data model is confirmed.
- Do not assume the representative player ID field name.
- Do not assume the export contains previous group membership.
- Do not design the Snowflake parsing logic yet; that belongs with later objectives.

## Objective 1 exit criteria

Objective 1 is complete when the guide records:

- Confirmed CPD and MDM version. Current status: confirmed as 5.3.x.
- Confirmed MDM API service group and Entity Maintenance/data API family. Current status: confirmed from IBM docs.
- Confirmed advanced export endpoint and request shape. Current status: open in target environment.
- Confirmed timestamp property for the watermark. Current status: confirmed from IBM docs as `entity_last_updated` for entity export.
- Confirmed player entity type filter.
- Confirmed enrichment endpoints for records and history.
- Confirmed relevant API limits. Current status: open; public docs checked here do not state whether the export cap is 10,000 or 100,000 records.
- Confirmation that direct Neo4j access is not needed. Current status: confirmed out of scope.
