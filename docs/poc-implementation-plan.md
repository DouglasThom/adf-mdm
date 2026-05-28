# POC instruction plan

This document describes the procedure to document and validate. It is not a build plan for application code, deployment scripts, or infrastructure implementation.

## Concrete approach v1

Use IBM MDM advanced export as the first batch discovery step, then use entity detail, records, or history calls only when the export output does not contain enough information to produce the player-to-group delta row.

1. ADF starts on a schedule and reads the last successful watermark from Snowflake.
2. ADF requests an IBM MDM export for player entities updated since that watermark.
3. IBM MDM creates an export file containing changed candidate entities.
4. ADF polls the export job status until the file is ready.
5. ADF downloads the export file and writes the raw response or file contents to a Snowflake holding table.
6. Snowflake derives the candidate player-to-group changes from the holding table.
7. If a candidate row does not include previous and current group values, ADF calls IBM MDM entity history or entity detail APIs for that entity ID and stores those raw responses too.
8. Snowflake writes one curated output row per changed player ID.
9. ADF advances the watermark only after all raw and curated Snowflake writes for the batch succeed.

### Example delta window

Use UTC timestamps for the POC unless the target MDM environment proves otherwise.

| Field | Example value |
| --- | --- |
| Previous watermark | `2026-05-28T13:00:00Z` |
| Batch end time | `2026-05-28T14:00:00Z` |
| Export filter | `entity_last_updated >= previous_watermark AND entity_last_updated < batch_end_time` |
| Player entity filter | `{player_entity_type}`, resolved per environment, such as `person-prod` or `person-dev` |

### Example IBM MDM export request shape

This is an illustrative request shape based on IBM's advanced export documentation, not a ready-to-run payload. Confirm the exact environment-specific entity type, property names, and supported operators in the target MDM API explorer before using it in ADF.

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
        "values": ["{player_entity_type}"]
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

### Example enrichment calls

Use these only after the candidate export identifies the entity ID.

| Need | IBM MDM API pattern | Example placeholders |
| --- | --- | --- |
| Current member records for an entity | `GET /mdm/v1/entities/{id}/records` | `{id}=person-prod-12345`, `entity_type={player_entity_type}`, `crn={mdm_service_crn}` |
| Entity change history | `GET /mdm/v1/entities/{id}/history` | `start_time={previous_watermark}`, `end_time={batch_end_time}`, `offset=0`, `limit=50` |
| Export status | `GET /mdm/v1/data_exports/{export_id}` | `{export_id}` from the export creation response |
| Export file download | `GET /mdm/v1/data_exports/{export_id}/download` | Download only after status is successful |

### Example Snowflake holding columns

Document these as the target table design for the POC. The exact DDL can be written later if executable artifacts are explicitly requested.

| Column | Purpose |
| --- | --- |
| `batch_id` | Groups all raw blocks from one ADF run. |
| `block_id` | Identifies one export file, page, or enrichment response. |
| `source_api` | Names the source call, such as `data_exports`, `entity_history`, or `entity_records`. |
| `request_url` | Stores the IBM MDM request URL without secrets. |
| `request_window_start_utc` | Stores the inclusive lower watermark. |
| `request_window_end_utc` | Stores the exclusive upper watermark. |
| `entity_id` | Stores the MDM entity ID when the response is entity-specific. |
| `response_payload` | Stores the raw JSON payload or file row/block as Snowflake semi-structured data. |
| `response_hash` | Supports replay and duplicate detection. |
| `adf_run_id` | Ties Snowflake rows back to the ADF run. |
| `loaded_at_utc` | Records when the response was stored. |

### Example curated output columns

| Column | Purpose |
| --- | --- |
| `batch_id` | Source batch that produced the curated row. |
| `player_source_system` | Source system for the player record, if available. |
| `source_player_id` | Source player identifier from MDM. |
| `entity_id` | MDM entity ID that changed. |
| `previous_global_player_id` | Prior group/global player ID, if available. |
| `current_global_player_id` | Current group/global player ID, if available. |
| `representative_player_id` | MDM representative player ID after the change. |
| `change_type` | `NO_GROUP_TO_GROUP`, `GROUP_TO_GROUP`, or `GROUP_TO_NO_GROUP`. |
| `mdm_change_timestamp_utc` | MDM update or effective timestamp used for ordering. |
| `extracted_at_utc` | ADF extraction timestamp. |

## Objective 1: Confirm supported IBM MDM batch API path

Focused guide: [Objective 1 API path confirmation](objective-1-api-path.md).

- [x] Confirm the target IBM Cloud Pak for Data release is 5.3.x.
- [x] Use the IBM Master Data Management API reference and Cloud Pak for Data 5.3.x MDM docs as the planning baseline.
- [x] Confirm MDM APIs are grouped into Configuration, Entity Maintenance, Job, Matching, Migration, and Model services.
- [x] Confirm the Entity Maintenance service can retrieve records, entities, entity history, and exports.
- [x] Confirm the advanced export API can filter entity exports by `entity_last_updated`.
- [x] Confirm the player entity model is modified `person`, with environment-specific values such as `person-prod` and `person-dev`.
- [ ] Confirm the advanced export API can filter to the selected environment-specific player entity type.
- [ ] Confirm the entity history endpoint is available for the player entity type and can be called per changed entity ID.
- [ ] Confirm REST API pagination, page-size limits, export limits, and rate limits.
- [x] Confirm direct Neo4j access is not required for this POC.

## Objective 2: Select the minimum batch delta architecture

Focused guide: [Objective 2 batch architecture](objective-2-batch-architecture.md).

- [x] Use ADF scheduled execution as the only trigger.
- [x] Use a stored watermark to define each delta window.
- [x] Call IBM MDM advanced export for each delta window.
- [x] Filter export requests to the environment-specific player entity type where the API supports it.
- [x] Write raw API responses directly to a Snowflake holding table.
- [x] Process only changed player-to-group mappings from the Snowflake holding table.
- [x] Call entity records or entity history APIs only for candidate entities that require enrichment.
- [x] Load curated changed player-to-group rows into Snowflake.
- [x] Do not use Kafka, Event Streams, Event Hubs, or event subscriptions for this POC.
- [x] Do not export the entire dataset unless required as a fallback diffing method.
- [x] Do not implement golden-record survivorship.
- [x] Use the MDM representative player ID as-is.

## Objective 3: Confirm datasets and identifiers

- [ ] List MDM record types from the data model.
- [ ] Select the player record type.
- [ ] List MDM entity types from the data model.
- [x] Select the player entity model as modified `person`.
- [ ] Confirm exact environment-specific player entity type values, expected to include `person-prod` and `person-dev`.
- [ ] Confirm source systems included in the POC.
- [ ] Confirm source player ID field.
- [ ] Confirm global player ID field.
- [ ] Confirm representative player ID field.
- [ ] Confirm API field that shows group membership before the change.
- [ ] Confirm API field that shows group membership after the change.
- [ ] If the delta response is incomplete, identify the entity detail API call needed to enrich each changed entity.

## Objective 4: Document the smallest working ADF batch procedure

- [ ] Document ADF linked service for IBM MDM REST API.
- [ ] Document ADF linked service for Azure Key Vault.
- [ ] Document ADF linked service for Snowflake.
- [ ] Document Snowflake holding table for raw MDM API response blocks.
- [ ] Document Snowflake table for curated player group deltas.
- [ ] Document Snowflake watermark table with an initial test timestamp.
- [ ] Document one pipeline that reads the watermark.
- [ ] Add REST API call example for one bounded export delta window.
- [ ] Add export-status polling loop example.
- [ ] Add export-file download step example.
- [ ] Add pagination or block loop example for enrichment API responses.
- [ ] Keep each block below the confirmed API limit.
- [ ] Document how each raw response block is written to the Snowflake holding table.
- [ ] Document how changed player IDs are parsed from the Snowflake holding table.
- [ ] Document how previous and current global player ID are parsed if present.
- [ ] Document how representative player ID is parsed if present.
- [ ] If required fields are missing, call MDM entity detail or history API for enrichment.
- [ ] Document POC output to Snowflake with one row per changed player ID.
- [ ] Document watermark update only after the Snowflake write succeeds.

### Example ADF activity sequence

| Order | Activity | Purpose |
| --- | --- | --- |
| 1 | Lookup | Read `previous_watermark_utc` from Snowflake. |
| 2 | Set variable | Set `batch_end_utc` to the ADF trigger time or current UTC time. |
| 3 | Web or REST call | Submit IBM MDM `data_exports` request for the bounded window. |
| 4 | Until loop | Poll `data_exports/{export_id}` until status is successful or failed. |
| 5 | Web or Copy activity | Download the export file. |
| 6 | Copy activity | Load raw export content into the Snowflake holding table. |
| 7 | Script or stored procedure activity | Derive curated player-to-group delta rows in Snowflake. |
| 8 | ForEach | For rows requiring enrichment, call entity records or history APIs and store raw responses. |
| 9 | Script or stored procedure activity | Re-run curation for enriched rows. |
| 10 | Script activity | Advance watermark to `batch_end_utc` only if prior steps succeeded. |

## Objective 5: Process blocks safely

- [ ] Define one block as one API page, export chunk, or fixed record-count request.
- [ ] Confirm whether the relevant export limit is 10,000 records, 100,000 records, or tenant-specific.
- [ ] Start with a small block size for the POC.
- [ ] Store raw responses in Snowflake before transformation.
- [ ] Store processing watermark after successful Snowflake write.
- [ ] Store API request URL, request timestamp, page token, holding-table batch ID, and response hash for replay.
- [ ] Re-run a processed block and confirm output is idempotent.
- [ ] Increase block size until it is close to, but below, the confirmed safe limit.

### Example block rules

- Treat the export file as the first block if the whole file is below the confirmed safe limit.
- Split the export into multiple holding blocks if ADF, Snowflake, or MDM file size limits require it.
- For entity history enrichment, use offset pagination with the confirmed maximum limit. The IBM API reference currently shows a maximum `limit` of 50 for entity history, so document `limit=50` unless the target API explorer says otherwise.
- Stop pagination when the response has no additional changes, the next link is absent, or the offset reaches the returned total if the API provides one.
- Store every enrichment response before deriving curated output from it.

## Objective 6: Validate delta correctness

- [ ] Test no-group to group.
- [ ] Test group to different group.
- [ ] Test group to no-group if supported.
- [ ] Confirm unchanged players are absent from the output.
- [ ] Confirm changed player IDs are present exactly once per processed change.
- [ ] Confirm global player ID before the change.
- [ ] Confirm global player ID after the change.
- [ ] Confirm representative player ID after the change.
- [ ] Confirm API update timestamp or effective timestamp is captured.
- [ ] Confirm source record ID and source system are captured.

### Example validation matrix

| Scenario | Setup | Expected curated result |
| --- | --- | --- |
| No group to group | Player has no prior global player ID, then joins group `G100`. | One row with `change_type=NO_GROUP_TO_GROUP`, blank previous group, current group `G100`. |
| Group to different group | Player moves from `G100` to `G200`. | One row with `change_type=GROUP_TO_GROUP`, previous group `G100`, current group `G200`. |
| Group to no group | Player leaves group `G100`. | One row with `change_type=GROUP_TO_NO_GROUP`, previous group `G100`, blank current group. |
| Unchanged player | Player is present in MDM but not updated in the window. | No curated row. |
| Duplicate replay | Same raw block is processed twice. | No duplicate curated row for the same player/change timestamp. |

## Objective 7: Prepare documented handoff for future development

- [ ] Keep IBM credentials in Azure Key Vault.
- [ ] Keep raw API responses in a Snowflake holding table for replay.
- [ ] Keep curated player group delta output in Snowflake.
- [ ] Track watermarks outside the pipeline definition.
- [ ] Capture failed API response payloads separately.
- [ ] Log MDM request IDs, page tokens, holding-table batch IDs, Snowflake query IDs if available, and ADF run IDs.
- [ ] Document confirmed API limits from the target environment.
- [ ] Document selected record type, entity type, and field names.
- [ ] Document known gaps before future production planning.

## Documentation references checked

- IBM Master Data Management API reference: confirms MDM REST API service groups, `data_exports` endpoints, entity records, and per-entity history endpoints.
- IBM advanced master data export documentation: confirms export by last-updated timestamp and deleted entity support.
- Microsoft ADF REST connector documentation: confirms REST pagination patterns, end conditions, and max request controls.
- Microsoft ADF Snowflake V2 connector documentation: confirms Copy, Lookup, Script, and Snowflake sink/source support.

## IBM documentation checked

- IBM Cloud Pak for Data current release: 5.3.x.
- IBM Cloud Pak for Data 5.3.1 MDM history feature checked: 2026-05-28.
- IBM Master Data Management API reference last checked: 2026-05-28.
- IBM Master Data Management API reference last updated by IBM: 2025-04-24.
- IBM Master Data Management advanced export API documentation checked: 2026-05-28.
- Microsoft ADF REST connector documentation checked: 2026-05-28.
- Microsoft ADF Snowflake V2 connector documentation checked: 2026-05-28.

## Source links

- https://cloud.ibm.com/media/docs/pdf/cloud-pak-data/cloud-pak-data.pdf
- https://www.ibm.com/docs/en/cloud-paks/cp-data/5.3.x?topic=new-master-data-management
- https://www.ibm.com/docs/en/cloud-paks/cp-data/5.3.x?topic=data-apis-available-in-master-management
- https://cloud.ibm.com/apidocs/mdm
- https://www.ibm.com/docs/en/cloud-paks/cp-data/5.3.x?topic=data-advanced-master-exports-using-api
- https://learn.microsoft.com/en-us/azure/data-factory/connector-rest
- https://learn.microsoft.com/en-us/azure/data-factory/connector-snowflake
