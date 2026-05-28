# Setup checklist

## IBM Cloud Pak for Data and MDM

- [ ] Confirm IBM Cloud Pak for Data version is 5.3.x.
- [ ] Confirm IBM Master Data Management service is installed and running.
- [ ] Confirm MDM project ID.
- [ ] Confirm MDM service CRN.
- [ ] Confirm MDM API base URL.
- [ ] Confirm API authentication method and token generation flow.
- [ ] Confirm user or service account has `mdm-oc.data.read`.
- [ ] Confirm user or service account has `mdm-oc.model.read`.
- [ ] Run MDM data health check against `target=neo4j`.
- [ ] Confirm player source record type.
- [ ] Confirm player entity type.
- [ ] Confirm field that contains source player ID.
- [ ] Confirm field that contains global player ID.
- [ ] Confirm field or rule that identifies representative player ID.
- [ ] Confirm entity history retention is enabled and long enough for the planned batch cadence.
- [ ] Confirm documented page-size, pagination, export, and rate limits in the target environment.
- [ ] Confirm whether any required endpoint has a 100,000-record limit.

## IBM MDM REST API delta access

- [ ] Confirm endpoint for listing or exporting changed entities within a time window.
- [ ] Confirm endpoint for retrieving one entity's history.
- [ ] Confirm endpoint for retrieving current entity detail by entity ID.
- [ ] Confirm response field that identifies entity membership changes.
- [ ] Confirm response field that identifies previous group/global player ID.
- [ ] Confirm response field that identifies current group/global player ID.
- [ ] Confirm response field that identifies representative player ID.
- [ ] Confirm pagination parameters.
- [ ] Confirm timestamp field to use for the ADF watermark.
- [ ] Confirm timezone used by the MDM API.

## Azure

- [ ] Create Azure resource group for the POC.
- [ ] Create Azure Data Factory instance.
- [ ] Create Azure Storage account.
- [ ] Create raw landing container for MDM API responses.
- [ ] Create curated container or table for player group delta output.
- [ ] Create storage location or table for the ADF watermark.
- [ ] Create Key Vault.
- [ ] Store IBM API credential in Key Vault.
- [ ] Enable ADF managed identity.
- [ ] Grant ADF access to Key Vault secrets.
- [ ] Grant ADF write access to raw landing storage.
- [ ] Grant ADF write access to curated output storage.
- [ ] Grant ADF read and write access to the watermark store.

## Proof data

- [ ] Identify 5 to 10 player IDs for a controlled POC.
- [ ] Include at least one no-group to group case.
- [ ] Include at least one group to different-group case.
- [ ] Include at least one group to no-group case if MDM supports the scenario.
- [ ] Record expected global player ID before each change.
- [ ] Record expected representative player ID before each change.
- [ ] Record expected global player ID after each change.
- [ ] Record expected representative player ID after each change.
