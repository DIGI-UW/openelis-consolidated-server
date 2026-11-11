# OpenELIS Consolidated Server

Two OpenELIS Global 3.3.0 instances (labs OE-A and OE-B) that send their data to one central SHR
(shared health record, a HAPI FHIR server) through [FHIR Data Pipes](https://github.com/google/fhir-data-pipes).

Guide: [Consolidated Server](https://uwdigi.atlassian.net/wiki/spaces/oeg/pages/1776943112)

| Part | File | Address |
|---|---|---|
| SHR (HAPI FHIR 5.5.1) | `docker-compose.yml` | <http://localhost:8090/fhir/> |
| OpenELIS instance 1 (OE-A) | `docker-compose-OE1.yml` | <https://localhost> (`admin` : `adminADMIN!`) |
| OpenELIS instance 2 (OE-B) | `docker-compose-OE2.yml` | <https://localhost:444> (`admin` : `adminADMIN!`) |
| Pipeline controller, instance 1 | `docker-compose-OE1.yml` | <http://localhost:8095> |
| Pipeline controller, instance 2 | `docker-compose-OE2.yml` | <http://localhost:8096> |

Each OpenELIS instance is the openelis-docker 3.3.0 stack without the analyser Bridge, with its own
database, keystores and configuration. Ports and passwords are in `.env`.

## Start

```bash
docker compose up -d                              # SHR; creates the shr-pipeline-network
docker compose -f docker-compose-OE1.yml up -d    # OpenELIS 1 and its pipeline controller
docker compose -f docker-compose-OE2.yml up -d    # OpenELIS 2 and its pipeline controller
```

Docker warns about orphan containers because the three files share one project; that is expected.

Then, for each pipeline controller: open it and click **Run Full** once. Incremental runs then copy new and
changed resources every 2 minutes (`incrementalSchedule` in `configs/fhirpipes/config*/application.yaml`).

If the controller then shows **Last run failed!** and its log has `HTTP 409 ... client-assigned ID constraint
failure`, click **Run Full** again. OpenELIS gives a site's Organization and Location the same id; the first run
creates both in the SHR at the same moment and the SHR refuses one of them. The second run finds the other one
there and completes. An incremental run that hits it (a new site) is retried by the next one.

## How it is wired

- Each OpenELIS writes its orders, patients, samples and results as FHIR resources to its own FHIR server
  (`org.openelisglobal.fhirstore.uri`), which stores them in the OpenELIS database.
- Its pipeline controller reads them from that database (`fhirFetchMode: HAPI_JDBC`,
  `configs/fhirpipes/config*/hapi-postgres-config_local.json`) and writes them to the SHR
  (`sinkFhirServerUrl`).
- `org.openelisglobal.facility.*` in `configs/openelis/properties*/common.properties` names each lab
  (`OE-A` in Uganda, `OE-B` in Kenya). OpenELIS publishes it as an Organization with a `facility_id`
  identifier, so the SHR can tell the labs apart.

| Instance 1 | Instance 2 | Purpose |
|---|---|---|
| `configs/openelis/properties/` | `configs/openelis/properties2/` | `common.properties` (FHIR store, facility), `SystemConfiguration.properties` |
| `configs/openelis/nginx/nginx.conf` | `configs/openelis/nginx/nginx2.conf` | Proxy |
| `configs/openelis/database/data` | `configs/openelis/database/data2` | Database files |
| `configs/fhirpipes/config` | `configs/fhirpipes/config2` | Pipeline: source database, SHR address, resource list, schedule |

`configs/openelis/configuration`, `menu`, `translation`, `branding` and `odoo` are shared by both instances.

## Upgrading from OpenELIS 3.2

Pull the new images and run the three `up -d` commands again. Each instance upgrades its database on
start. The second instance's settings moved from `configs/openelis/properties/common2.properties` to
`configs/openelis/properties2/common.properties`: copy any change you made there.

## Resources

The pipeline controllers start with a 2 GB Java heap each (`PIPELINE_HEAP` in `.env`). The whole setup
needs about 8 GB of memory for Docker.
