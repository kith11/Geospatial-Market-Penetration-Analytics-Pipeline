my-data-pipeline/
│
├── .env                  # Local secret keys (Apify token, MotherDuck token, R2 credentials)
├── .gitignore            # Tells git to ignore your env files, venv, and cache
├── README.md             # Project documentation and architecture details
├── requirements.txt      # Your Python dependencies
│
├── data/                 # LOCAL TEST DATA ONLY (Never commit this folder to Git)
│   ├── raw/              # Temporary storage for raw JSONs scraped from Apify
│   └── silver/           # Temporary storage for cleaned local Parquet files
│
├── src/                  # Main source code directory
│   ├── __init__.py
│   │
│   ├── pipelines/        # Polars ETL transformation scripts
│   │   ├── __init__.py
│   │   ├── extract.py    # Code to call Apify API and pull raw data
│   │   ├── transform.py  # Pure Polars logic (cleaning, filtering data types)
│   │   └── load.py       # Functions to push data to Cloudflare R2 and MotherDuck
│   │
│   └── utils/            # Shared helper functions
│       ├── __init__.py
│       └── cloud_auth.py # Boto3 client configuration for Cloudflare R2
│
├── flows/                # Prefect Workflow Orchestration scripts
│   └── main_flow.py      # The main pipeline file that chains extract -> transform -> load
│
└── app/                  # Frontend Analytics Layer
    └── dashboard.py      # Your Streamlit application pulling data from MotherDuck
=======
# Geospatial-Market-Penetration-Analytics-Pipeline
>>>>>>> 489542b90546d72b14ec0fc7346c851b6a9b3825
