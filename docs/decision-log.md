# Decision log

## Required decisions

| Decision | Options | Selected |
| --- | --- | --- |
| Batch delta source | Entity history API; advanced export API filtered by update window; current export with ADF-side diffing | TBD |
| ADF API pattern | Web activity plus pagination loop; Copy activity REST connector; Azure Function wrapper called by ADF | TBD |
| Raw holding format | Snowflake VARIANT per response block; relational columns plus VARIANT payload; flattened relational rows | TBD |
| Raw holding location | Snowflake holding table | Snowflake holding table |
| Batch boundary | Watermark timestamp window; fixed page count; fixed record count | TBD |
| Checkpoint | Snowflake watermark table; ADF metadata table | TBD |
| Curated destination | Snowflake table derived from the holding table | TBD |
| Dataset selection | Entity type filter; record type filter; source-system filter; API query filter | TBD |
| Representative player ID rule | Use MDM-provided representative ID field; derive from member record ordering; call entity detail API | TBD |
| Fallback if history API cannot return deltas | Advanced export plus update-window filter; export current group mapping and diff against previous snapshot | TBD |

## Initial recommendation for POC

| Decision | Recommended selection |
| --- | --- |
| Batch delta source | Entity history API if it can filter by update window; otherwise advanced export API filtered by update window |
| ADF API pattern | Copy activity REST connector for simple pagination; Web activity loop if Copy cannot express the API pagination |
| Raw holding format | Snowflake VARIANT per response block with batch metadata columns |
| Raw holding location | Snowflake holding table |
| Batch boundary | Watermark timestamp window plus API pagination |
| Checkpoint | Snowflake watermark table |
| Curated destination | Snowflake table populated from the holding table |
| Dataset selection | Filter to the player entity type first, then source-system filter if needed |
| Representative player ID rule | Use MDM-provided representative ID field if present; otherwise call entity detail API |
| Fallback if history API cannot return deltas | Export changed/current mappings and diff against the prior POC snapshot |
