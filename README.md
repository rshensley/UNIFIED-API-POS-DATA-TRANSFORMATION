# Unified API POS Data Transformation — Multi-Serialized ETL Pipeline

**INFO U762 — Interoperability | Module 6**

## Project Overview

This application is a multi-serialized ETL pipeline using Python, Flask, SOAP/XML, REST/JSON, SQLAlchemy, and SQLite.

The project takes the Fake Trillionaire E-Commerce API provided in the document and implements a separate ETL application that retrieves item data through REST/JSON and order data through SOAP/XML.

The ETL pipeline normalizes both the datasets, apply a location-specific transformation to SKU, stores the results in a SQLite database, and exposes the transformed records through a Flask REST API.

It then converts numeric SKU into strings prefixed with `SPTBG-` without changing the source API.

## Installation and Setup

**Technologies:**

- Python 3.11
- Flask
- Spyne
- Requests
- SQLAlchemy / Flask-SQLAlchemy
- SQLite
- Faker
- XML ElementTree
- lxml
- schedule

Python 3.11 was used because the installed Spyne library had an import compatibility error under Python 3.14.

Clone the repository and navigate into the project directory:

```bash
git clone https://github.com/rshensley/UNIFIED-API-POS-DATA-TRANSFORMATION.git
cd UNIFIED-API-POS-DATA-TRANSFORMATION
```

Create and activate a Python 3.11 virtual environment. The following uses `uv` to manage the Python interpreter:

```bash
uv python install 3.11
uv venv --python 3.11 --seed .venv
source .venv/bin/activate
```

Install the dependencies:

```bash
python -m pip install -r requirements.txt
```

### Start the Source API

From the project root:

```bash
cd source_api
source ../.venv/bin/activate
python main.py
```

The Fake Trillionaire API runs on `http://127.0.0.1:5000`.

### Start the ETL Application

Open a second terminal from the project root:

```bash
cd etl_pipeline
source ../.venv/bin/activate
python app.py
```

The unified REST service runs on `http://127.0.0.1:8080`.

Both applications need to be running when the ETL pipeline retrieves data from the source API.

## API Endpoints

### Source API

| Method | Endpoint | Description |
|---|---|---|
| GET | `/items` | Returns generated items as JSON |
| GET | `/orders` | Returns generated orders as JSON |
| POST | `/order` | Creates a calculated sample order response |
| GET | `/service?wsdl` | Returns the SOAP service contract |
| POST | `/service` | Processes SOAP/XML operations |

### Unified ETL API

| Method | Endpoint | Description |
|---|---|---|
| POST | `/api/unified/trigger-etl` | Executes the ETL pipeline |
| GET | `/api/unified/records` | Returns all unified records |
| GET | `/api/unified/records?source_type=items` | Returns transformed items |
| GET | `/api/unified/records?source_type=orders` | Returns transformed orders |

### Trigger the ETL Pipeline

```bash
curl -fsS -X POST \
  http://127.0.0.1:8080/api/unified/trigger-etl |
  python -m json.tool
```

**Actual response:**

```json
{
    "message": "ETL pipeline executed successfully."
}
```

The pipeline retrieved 100 REST items and 100 SOAP orders and stored 200 transformed records in SQLite.

### Retrieve Transformed Items

```bash
curl -fsS \
  -w '\nItems HTTP status: %{http_code}\n' \
  -o /tmp/unified_items.json \
  'http://127.0.0.1:8080/api/unified/records?source_type=items'
```

**Actual result:** HTTP `200 OK`

### Retrieve Transformed Orders

```bash
curl -fsS \
  -w '\nOrders HTTP status: %{http_code}\n' \
  -o /tmp/unified_orders.json \
  'http://127.0.0.1:8080/api/unified/records?source_type=orders'
```

**Actual result:** HTTP `200 OK`

## Example JSON Responses

The following records were retrieved from the unified REST API in testing: 

### Transformed Item

```json
{
  "created_at": "2026-11-19T17:54:09.092380",
  "currency": "USD",
  "id": 1,
  "quantity": 39,
  "sku_id": "SPTBG-1",
  "sku_name": "Nearly Art",
  "source_type": "items",
  "status": "available",
  "total_price": 7126036.03,
  "transaction_date": "2026-06-22T12:35:30.798033",
  "unit_price": 7126036.03
}
```

### Transformed Order

```json
{
  "created_at": "2026-11-19T17:54:09.092428",
  "currency": "USD",
  "id": 101,
  "quantity": 3,
  "sku_id": "SPTBG-86",
  "sku_name": "College Enter",
  "source_type": "orders",
  "status": "failed",
  "total_price": 21577039.44,
  "transaction_date": "2026-10-30T04:44:37.062572",
  "unit_price": 7192346.48
}
```

## Validation and Testing

### Source API Testing

The REST item and order endpoints were tested using `curl`

| Test | Observed Result |
|---|---|
| REST GET `/items` | 100 items returned |
| REST POST `/order` | Valid calculated order response |
| REST GET `/orders` | 100 orders returned |
| SOAP WSDL | All three expected operations found |
| SOAP `list_orders` | 100 order SKU fields returned |

### Transformation and Database Testing

Both normalization functions were tested against the retrieved source records before loading any records into SQLite.

The database schema was also inspected to confirm that `sku_id` was stored as `VARCHAR`.

| Test | Observed Result |
|---|---|
| REST items transformed | 100 |
| SOAP orders transformed | 100 |
| SQLite items persisted | 100 |
| SQLite orders persisted | 100 |
| Total records persisted | 200 |
| String SKU validation | Passed for all 200 records |
| Orders with unmatched item SKUs | 0 |

### Final REST API Testing

Both endpoints called using `curl`, and the returned JSON files were examined to validate the transformed records.

**Results:**

```text
Items HTTP status: 200
Orders HTTP status: 200

=== Verification ===
Items: 100 records, transformation valid: True
Orders: 100 records, transformation valid: True

All 200 records verified successfully.
```

## Project Files

- `source_api/main.py` — Fake Trillionaire REST/JSON and SOAP/XML source application
- `etl_pipeline/config.py` — REST and SOAP endpoint configuration
- `etl_pipeline/ingestion.py` — Retrieves JSON items and SOAP/XML orders
- `etl_pipeline/models.py` — SQLAlchemy unified database model
- `etl_pipeline/etl_core.py` — Data normalization, SKU transformation, and SQLite loading
- `etl_pipeline/app.py` — Flask REST API for triggering ETL and retrieving unified records
- `requirements.txt` — Python dependencies and installed versions
- `.gitignore` — Excludes virtual environments, caches, and generated databases
- `README.md` — Project documentation, testing results, and reflection

## Reflection

This assignment showed how a single ETL pipeline can integrate data delivered through different serialization formats. I recreated the Fake Trillionaire source API and verified its REST/JSON and SOAP/XML endpoints using curl. I implemented the ingestion, transformation, and SQLite persistence layers while keeping the source API unchanged. The main change was changing the SKU identifier to a string and adding the `SPTBG-` prefix to both item and order records.

One unexpected issue was that Spyne failed to import under Python 3.14, so I had to recreate the virtual environment using Python 3.11.

After executing the pipeline, curl returned HTTP 200 for both unified endpoints. Database checks confirmed 100 items and 100 orders, all with correctly transformed identifiers, and no unmatched order SKUs. This is a good reason to verify actual data transformations and persistence instead of just relying on successful HTTP responses.
