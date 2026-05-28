# POC implementation plan

## Objective 1: Confirm supported IBM MDM batch API path

- [ ] Confirm the target IBM Cloud Pak for Data release is 5.3.x.
- [ ] Confirm the IBM Master Data Management API reference applies to the target environment.
- [ ] Confirm MDM APIs are grouped into Configuration, Entity Maintenance, Job, Matching, and Model services.
- [ ] Confirm the Entity Maintenance service can retrieve records, entities, entity history, and exports.
- [ ] Confirm the entity history endpoint is available for the player entity type.
- [ ] Confirm the advanced export API is available as a fallback batch source.
- [ ] Confirm REST API pagination, page-size limits, export limits, and rate limits.
- [ ] Confirm direct Neo4j access is not required for this POC.

## Objective 2: Select the minimum batch delta architecture

- [ ] Use ADF scheduled execution as the only trigger.
- [ ] Use a stored watermark to define each delta window.
- [ ] Call IBM MDM REST APIs for each delta window.
- [ ] Filter requests to the player entity type where the API supports it.
- [ ] Land raw API responses in Azure storage.
- [ ] Process only changed player-to-group mappings from the landed responses.
- [ ] Do not use Kafka, Event Streams, Event Hubs, or event subscriptions for this POC.
- [ ] Do not export the entire dataset unless required as a fallback diffing method.
- [ ] Do not implement golden-record survivorship.
- [ ] Use the MDM representative player ID as-is.

## Objective 3: Confirm datasets and identifiers

- [ ] List MDM record types from the data model.
- [ ] Select the player record type.
- [ ] List MDM entity types from the data model.
- [ ] Select the player entity type.
- [ ] Confirm source systems included in the POC.
- [ ] Confirm source player ID field.
- [ ] Confirm global player ID field.
- [ ] Confirm representative player ID field.
- [ ] Confirm API field that shows group membership before the change.
- [ ] Confirm API field that shows group membership after the change.
- [ ] If the delta response is incomplete, identify the entity detail API call needed to enrich each changed entity.

## Objective 4: Build the smallest working ADF batch

- [ ] Create ADF linked service for IBM MDM REST API.
- [ ] Create ADF linked service for Azure Key Vault.
- [ ] Create ADF linked service for Azure storage.
- [ ] Create watermark storage with an initial test timestamp.
- [ ] Create one pipeline that reads the watermark.
- [ ] Add REST API call for one bounded delta window.
- [ ] Add pagination or block loop for API responses.
- [ ] Keep each block below the confirmed API limit.
- [ ] Land each raw response block in Azure storage.
- [ ] Parse changed player IDs from the landed response.
- [ ] Parse previous and current global player ID if present.
- [ ] Parse representative player ID if present.
- [ ] If required fields are missing, call MDM entity detail or history API for enrichment.
- [ ] Write POC output with one row per changed player ID.
- [ ] Update the watermark only after the output write succeeds.

## Objective 5: Process blocks safely

- [ ] Define one block as one API page, export chunk, or fixed record-count request.
- [ ] Confirm whether the relevant API limit is 100,000 records in the target environment.
- [ ] Start with a small block size for the POC.
- [ ] Store raw responses before transformation.
- [ ] Store processing watermark after successful output write.
- [ ] Store API request URL, request timestamp, page token, and response file path for replay.
- [ ] Re-run a processed block and confirm output is idempotent.
- [ ] Increase block size until it is close to, but below, the confirmed safe limit.

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

## Objective 7: Prepare transition to serious development

- [ ] Keep IBM credentials in Azure Key Vault.
- [ ] Keep raw API responses for replay.
- [ ] Keep transformation output separate from raw landing data.
- [ ] Track watermarks outside the pipeline definition.
- [ ] Capture failed API response payloads separately.
- [ ] Log MDM request IDs, page tokens, response file paths, and ADF run IDs.
- [ ] Document confirmed API limits from the target environment.
- [ ] Document selected record type, entity type, and field names.
- [ ] Document known gaps before production build.

## IBM documentation checked

- IBM Cloud Pak for Data current release: 5.3.x.
- IBM Cloud Pak for Data 5.3.1 MDM history feature checked: 2026-05-28.
- IBM Master Data Management API reference last checked: 2026-05-28.
- IBM Master Data Management API reference last updated by IBM: 2025-04-24.
- IBM Master Data Management advanced export API documentation checked: 2026-05-28.

## Source links

- https://cloud.ibm.com/media/docs/pdf/cloud-pak-data/cloud-pak-data.pdf
- https://www.ibm.com/docs/en/cloud-paks/cp-data/5.3.x?topic=new-master-data-management
- https://www.ibm.com/docs/en/cloud-paks/cp-data/5.3.x?topic=data-apis-available-in-master-management
- https://cloud.ibm.com/apidocs/mdm
- https://www.ibm.com/docs/en/cloud-paks/cp-data/5.3.x?topic=data-advanced-master-exports-using-api
