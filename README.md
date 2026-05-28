# ADF + IBM Cloud Pak for Data MDM Delta POC

This repo contains a batch-oriented proof-of-concept plan for retrieving changed player-to-group mappings from IBM Cloud Pak for Data Master Data Management (MDM) with Azure Data Factory (ADF) and storing the curated deltas in Snowflake.

This is a documentation-only project. The repo is intended to capture instructions, checklists, decisions, validation steps, and assumptions; it is not intended to contain application code, build tooling, deployment scripts, or infrastructure implementation.

## POC goal

Prove that an ADF pipeline can call IBM MDM REST APIs on a schedule and retrieve only player records whose entity/group membership changed, including:

- player moved from no group to a group
- player moved from one group to another group
- player left a group

## Planned approach

Use ADF to run a watermark-based REST API batch. Each run requests a bounded delta window from IBM MDM entity history or export APIs, writes each raw response block directly to a Snowflake holding table, processes results in blocks below the confirmed API limit, and writes one curated output row per changed player ID to Snowflake.

## Planning files

- [POC instruction plan](docs/poc-implementation-plan.md)
- [Setup checklist](docs/setup-checklist.md)
- [Decision log template](docs/decision-log.md)

## Current assumptions to validate

- IBM Cloud Pak for Data target version is 5.3.x.
- IBM MDM record type for players is `person` or a project-specific player record type.
- IBM MDM entity type for player groups is `person_entity` or a project-specific player entity type.
- Delta extraction should be based on MDM entity membership changes, not full point-in-time export.
- ADF will orchestrate REST API calls, pagination, block processing, direct Snowflake loading, and Snowflake watermark updates.
