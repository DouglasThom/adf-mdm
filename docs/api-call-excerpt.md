# API call excerpt: auth, export, download, and pagination

This excerpt is for quick API Explorer testing. It intentionally uses placeholders for tenant-specific values, credentials, service CRNs, entity type names, and field names.

## API Explorer

- Public IBM MDM API reference: https://cloud.ibm.com/apidocs/mdm
- Target environment API Explorer: open the MDM API Explorer from the actual Cloud Pak for Data 5.3.x tenant and confirm these endpoints exist before using them in ADF.

Record these values from the target environment:

| Placeholder | Meaning |
| --- | --- |
| `{cpd_base_url}` | Cloud Pak for Data base URL. |
| `{mdm_base_url}` | MDM data API base URL, as shown by API Explorer. |
| `{mdm_service_crn}` | MDM service CRN if the endpoint requires `crn`. |
| `{access_token}` | Bearer token returned by the auth flow. |
| `{player_entity_type}` | Environment-specific modified `person` entity type, such as `person-prod` or `person-dev` if confirmed. |
| `{export_id}` | Export ID returned by the `data_exports` create response. |
| `{entity_id}` | Candidate entity ID returned by the export. |

## 1. Authenticate

Use the target tenant's documented Cloud Pak for Data auth method. For CPD username/password or API key auth, the usual shape to validate is:

```http
POST {cpd_base_url}/icp4d-api/v1/authorize
Content-Type: application/json

{
  "username": "{username}",
  "password": "{password}"
}
```

or, if the tenant uses API keys:

```http
POST {cpd_base_url}/icp4d-api/v1/authorize
Content-Type: application/json

{
  "username": "{username}",
  "api_key": "{api_key}"
}
```

Use the returned token as:

```http
Authorization: Bearer {access_token}
```

Do not put the username, password, API key, or bearer token in ADF pipeline JSON. Store credentials in Azure Key Vault.

## 2. Submit a small advanced export

This is the primary batch query. Use a narrow timestamp window while testing.

```http
POST {mdm_base_url}/mdm/v1/data_exports?crn={mdm_service_crn}
Authorization: Bearer {access_token}
Content-Type: application/json
Accept: application/json

{
  "export_type": "ENTITY",
  "file_name": "player_entity_delta_test_20260528T140000Z",
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

Expected result: a response containing an export identifier. Record the exact response field name in the target tenant; this guide refers to it as `{export_id}`.

## 3. Poll export status

```http
GET {mdm_base_url}/mdm/v1/data_exports/{export_id}?crn={mdm_service_crn}
Authorization: Bearer {access_token}
Accept: application/json
```

Poll until the target API reports a terminal status:

- success/completed: continue to download
- failed/error/canceled: stop and store the failure payload

Record the exact status field name and success value from the API Explorer response.

## 4. Download the export file

```http
GET {mdm_base_url}/mdm/v1/data_exports/{export_id}/download?crn={mdm_service_crn}
Authorization: Bearer {access_token}
Accept: application/json
```

Store the raw downloaded response before transforming it. For the POC, this raw response is the first block loaded into the Snowflake holding table.

## 5. Test enrichment pagination with 10 rows

Advanced export produces a file, not a normal page-by-page result set. Use `limit=10` pagination testing on enrichment endpoints such as entity history after the export gives you a candidate `{entity_id}`.

First page:

```http
GET {mdm_base_url}/mdm/v1/entities/{entity_id}/history?crn={mdm_service_crn}&start_time=2026-05-28T13:00:00Z&end_time=2026-05-28T14:00:00Z&offset=0&limit=10
Authorization: Bearer {access_token}
Accept: application/json
```

Second page:

```http
GET {mdm_base_url}/mdm/v1/entities/{entity_id}/history?crn={mdm_service_crn}&start_time=2026-05-28T13:00:00Z&end_time=2026-05-28T14:00:00Z&offset=10&limit=10
Authorization: Bearer {access_token}
Accept: application/json
```

Third page:

```http
GET {mdm_base_url}/mdm/v1/entities/{entity_id}/history?crn={mdm_service_crn}&start_time=2026-05-28T13:00:00Z&end_time=2026-05-28T14:00:00Z&offset=20&limit=10
Authorization: Bearer {access_token}
Accept: application/json
```

Pagination rule:

1. Start with `offset=0&limit=10`.
2. Store the raw response.
3. Increase `offset` by `10` after each successful call.
4. Stop when the response has fewer than `10` results, has no next link, or reaches the returned total count if the API provides one.
5. For production sizing, use the confirmed maximum safe `limit`; the current planning assumption is that entity history supports up to `limit=50`, but the target API Explorer must confirm it.

If current entity records also support `offset` and `limit` in the target tenant, test them the same way:

```http
GET {mdm_base_url}/mdm/v1/entities/{entity_id}/records?crn={mdm_service_crn}&offset=0&limit=10
Authorization: Bearer {access_token}
Accept: application/json
```

## ADF shape

The ADF pipeline should mirror these calls:

1. Web activity: authenticate or retrieve token from Key Vault-backed process.
2. Web activity: submit `POST /mdm/v1/data_exports`.
3. Until activity: poll `GET /mdm/v1/data_exports/{export_id}`.
4. Copy or Web activity: download `GET /mdm/v1/data_exports/{export_id}/download`.
5. Copy activity: store raw export output in Snowflake.
6. ForEach activity: for candidate entities needing enrichment, call history or records with `offset`/`limit` pagination.
7. Script or stored procedure activity: curate output and advance the watermark only after successful raw and curated writes.

