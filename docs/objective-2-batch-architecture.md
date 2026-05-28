# Objective 2: Minimum batch delta architecture

This objective selects the smallest documented architecture for retrieving player-to-group deltas with ADF and IBM MDM.

## Architecture decision

Use a scheduled ADF batch pipeline with a Snowflake watermark. Each run asks IBM MDM advanced export for candidate player entities updated inside one bounded time window, using the environment-specific modified `person` entity type, stores the raw export in Snowflake, enriches only incomplete candidate rows, and writes curated player-to-group delta rows back to Snowflake.

This is a batch retrieval guide, not an event streaming design and not a direct database access design.

## Selected flow

1. ADF starts on a schedule.
2. ADF reads the last successful watermark from Snowflake.
3. ADF sets a fixed batch end timestamp for the run.
4. ADF submits an IBM MDM advanced export request for the watermark window.
5. ADF polls the export job until it succeeds or fails.
6. ADF downloads the export output.
7. ADF stores the raw export output in a Snowflake holding table.
8. Snowflake derives candidate player-to-group changes from the raw holding table.
9. ADF calls entity records or entity history only for candidate entities that need more detail.
10. Snowflake writes one curated row per changed player ID.
11. ADF advances the Snowflake watermark only after the raw and curated writes succeed.

## Architecture boundaries

| Area | Decision |
| --- | --- |
| Trigger | Scheduled ADF pipeline only. |
| Delta boundary | Snowflake-managed watermark timestamp window. |
| Primary MDM access | IBM MDM advanced export. |
| Player entity type | Environment-specific modified `person` entity value, such as `person-prod` or `person-dev`. |
| Enrichment MDM access | Entity records or entity history for candidate entities only. |
| Raw storage | Snowflake holding table. |
| Curated output | Snowflake player-to-group delta table. |
| Checkpoint | Snowflake watermark table. |
| Transformation location | Snowflake, using raw holding data and enrichment responses. |

## Explicitly out of scope

- Kafka, IBM Event Streams, Azure Event Hubs, webhooks, or other event subscriptions.
- Direct Neo4j access.
- Full-dataset export unless the selected delta export path cannot produce a usable candidate set.
- Golden-record survivorship logic.
- Recalculating the representative player ID unless MDM does not provide it.
- Application code, deployment scripts, or infrastructure-as-code.

## Why this is the minimum approach

- ADF handles orchestration and scheduling.
- IBM MDM advanced export handles batch candidate discovery.
- Snowflake holds replayable raw data, curated deltas, and the watermark.
- Per-entity API calls are limited to enrichment, so the design remains batch-first.
- The player entity type is a documented environment parameter, not a hardcoded universal value.
- The fallback is simple: if updated-entity export is not enough, export current mappings and diff against the prior POC snapshot in Snowflake.

## Open dependencies from Objective 1

The architecture is selected, but these details still need confirmation before the guide can become a final runbook:

- Advanced export is enabled in the target environment.
- The exact environment values for the modified `person` entity type are confirmed, expected to include `person-prod` and `person-dev`.
- Entity records and entity history endpoints are available in the target API explorer.
- The export record/file limit is confirmed as 10,000, 100,000, or tenant-specific.

## Objective 2 exit criteria

Objective 2 is complete when the guide records:

- ADF scheduled execution as the only trigger.
- Snowflake watermark as the batch boundary and checkpoint.
- IBM MDM advanced export as the primary delta candidate source.
- Modified `person` player entity type documented as environment-specific, with `person-prod` and `person-dev` as expected values.
- Snowflake holding table as the raw landing location.
- Snowflake curated table as the output.
- Entity records/history calls as enrichment only.
- Event streaming, direct Neo4j access, full export, and survivorship logic as out of scope.
