# Streamlit GCP Demo

A simple Streamlit dashboard for Singapore HDB resale data, refactored to support a BigQuery backend. This is a refactor code from the repository https://github.com/thinkdaniel/streamlit-lesson

This repository contains:

- `app_original.py` — the original Streamlit app that loads local CSV data.
- `environment.yml` — Conda environment with required dependencies.
- `README.md` — this project guide.

---

## What this app does

The app displays HDB resale listings and provides interactive filters for:

- Town
- Flat type
- Resale price range
- Date range

It also generates:

- filtered transaction table
- transaction count
- average and median price metrics
- median floor area
- bar charts for town prices and flat type transaction volume
- monthly resale price trend

---

## Getting started

### 1. Install dependencies

Using Conda:

```bash
conda env create -f environment.yml
conda activate streamlit
```


### 2. Create `app.py`

Copy `app_original.py` to `app.py`:

```bash
cp app_original.py app.py
```

### 3. Configure BigQuery access

Create `.streamlit/secrets.toml` with a service account JSON payload:

```toml
[gcp_service_account]
type = "service_account"
project_id = "YOUR_PROJECT_ID"
private_key_id = "..."
private_key = "-----BEGIN PRIVATE KEY-----\n...\n-----END PRIVATE KEY-----\n"
client_email = "..."
client_id = "..."
auth_uri = "https://accounts.google.com/o/oauth2/auth"
token_uri = "https://oauth2.googleapis.com/token"
auth_provider_x509_cert_url = "https://www.googleapis.com/oauth2/v1/certs"
client_x509_cert_url = "..."
```
Please open your service account json file and file in the variables field by field.

> Do not commit secret credentials to version control.

### 4. Apply the BigQuery refactor

Edit `app.py` and replace the local CSV loading section with the BigQuery connection code below.

Remove this block from `app.py`:

```python
DATA_PATH = "./data/resale_data.csv"

@st.cache_data
def load_data(path):
    df = pd.read_csv(path)
    df["month"] = pd.to_datetime(df["month"])
    return df

df = load_data(DATA_PATH)
```

Then add the BigQuery setup code immediately after:

```python
from google.cloud import bigquery
from google.oauth2 import service_account

@st.cache_resource
def get_bigquery_client():
    if "gcp_service_account" not in st.secrets:
        raise ValueError(
            "Add `gcp_service_account` to `.streamlit/secrets.toml` to initialize the BigQuery client."
        )

    credentials_info = st.secrets["gcp_service_account"]
    credentials = service_account.Credentials.from_service_account_info(
        credentials_info
    )
    project = credentials_info.get("project_id")
    return bigquery.Client(credentials=credentials, project=project)

try:
    bq_client = get_bigquery_client()
    st.sidebar.success("BigQuery client initialized")
except Exception as e:
    bq_client = None
    st.sidebar.error(f"BigQuery init error: {e}")

@st.cache_data
def load_data():
    query = st.secrets.get("bq_test_query") or (
        """
        SELECT *
        FROM resale.prices
        """
    )
    df = bq_client.query(query).result().to_dataframe()
    df["month"] = pd.to_datetime(df["month"])
    return df

if bq_client:
    df = load_data()
else:
    st.error("Unable to load data because BigQuery client initialization failed.")
```

### 5. Run the app

```bash
streamlit run app.py
```

Open the local URL shown in the terminal.

### Notes

- The current repository does not include `app.py` yet; `app_original.py` is the base source.
- Make sure the BigQuery dataset and table used in the query exist and are accessible with the provided service account.

---

## Deploy to Streamlit Cloud

To deploy to Streamlit Cloud, you need to copy the content in `.streamlit/secrets.toml` to the secrets in Streamlit app setting. 

---



