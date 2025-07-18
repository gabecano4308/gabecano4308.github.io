---
title: "Using Postgres and dbt With an Open Data Philly Dataset"
date: 2025-07-18
permalink: /posts/2025-07-18-dbt_map_proj/
excerpt: How to use dbt and Postgres to transform Philadelphia open data into analytics-ready tables.
tags:
---

It feels like dbt is getting name-dropped everywhere on the Internet lately. Every Data Engineer job 
posting I've read in the last few months has asked for dbt experience. My data tool FOMO being 
sufficiently tickled, I decided to read up on what all the fuss is about and do a mini 
project/walkthrough.

After studying the docs, I realized that dbt does a bunch of different things. There's a 
cloud-based dbt and a "dbt core" for local use. As the org has grown, so has its product 
offerings and features. However, to simplify dbt into one sentence, I'd say it's a 
tool for transforming, testing, and documenting data using SQL and YAML config files. 
There's a heavy emphasis on modularity and version control to make analytics workflows more 
reliable and scalable. Having described the gist of dbt, I'll move onto the project at hand. 

## Project Directory Structure
```
map_proj/
├── models/
│   ├── staging/
│   │   ├── _schema_staging.yml          # DBT schema file for staging model metadata
│   │   └── meal_site_staging.sql        # Cleans up raw data
│   ├── mart/
│   │   ├── _schema_mart.yml             # DBT schema file for mart model metadata
│   │   ├── meal_sites_by_zip.sql        # Gives the num of meal sites per zip code
│   │   └── senior_sites_contact.sql     # Gives info on meal sites for seniors
├── dbt_project.yml                      # DBT config for models, materializations, schemas
├── get_free_meal_data.py                # Script to pull and load GeoJSON from web into Postgres
├── docker-compose.yml                   # Defines and runs PostGIS container
└── .env                                 # Environment variables (Postgres credentials for docker)

~/.dbt/
└── profiles.yml                         # Lists database connection details for dbt projects
```

## ETL Data From the Web Into a Postgres Database
Before getting into the dbt part of the project, some setup is needed. The first step is to find 
some actual data. I found a table of free meal sites in Philadelphia [here](https://opendataphilly.org/datasets/free-food-sites/). Next, I'll set up a database using this `docker-compose.yml`, which ups one in a container 
based on an existing postgis docker image:

```yaml
# docker-compose.yml for postgis database

services:
  postgis:
    image: postgis/postgis:15-3.5  # https://hub.docker.com/r/postgis/postgis/
    container_name: postgis_db
    restart: always
    ports:
      - "5432:5432"     # maps localhost port 5432 to container's port 5432
    environment:        # vals of the env variables are filled using the .env file
      POSTGRES_USER:    
      POSTGRES_PASSWORD: 
      POSTGRES_DB: 
    volumes:           # allows data to persist when making new containers
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

Here are environment variables for connecting to the database listed in a .env file:

```conf
# env variables for connecting to the database

POSTGRES_USER=postgres
POSTGRES_PASSWORD=mysecretpassword
POSTGRES_DB=gisdb
POSTGRES_PORT=5432
POSTGRES_HOST=localhost
```

Running `docker compose up --build -d` should get the database running (`-d` means run in background). 
I'd like to connect to the database from inside the container. So, I grab the container ID from the 
output of `docker ps` and paste it into this command: `docker exec -it <container ID> bash`. Once in 
the container, I connect to the database like so, including the username and database name from my
`.env` file: `psql -U $POSTGRES_USER -d $POSTGRES_DB`. Here's a screenshot of what doing all these 
things successfully looks like:

![create_and_enter_db](/images/create_and_enter_db.png)

Now I need a script that takes the free meal table from the web page and saves it to the database. The below
python script uses the `requests` library to grab the data via an API request. `geopandas` then formats the 
API response as a table and adds a column for time of ingestion. The script then connects to the database with 
the appropriate credentials and adds the data to a table named `free_meal_sites`:

```python
import os
import geopandas as gpd
import requests
from dotenv import load_dotenv
from datetime import UTC, datetime
from sqlalchemy import create_engine, text

# prepare API URL and params
url = "https://hub.arcgis.com/api/v3/datasets/5825a32bb8844bb097f7a16d4fbf4f23_0/downloads/data"
params = {
    "format": "geojson",
    "spatialRefId": 4326,
    "where": "1=1"
}

print("Grabbing data via API...")
try:
    # Get the data
    response = requests.get(url, params=params)
    response.raise_for_status()

except requests.exceptions.RequestException as e:
    print("Couldn't grab data:", e)
    exit(1)

gdf = gpd.read_file(response.content)
gdf["ingestion_timestamp"] = datetime.now(UTC).strftime("%Y-%m-%d %H:%M:%S")
print(f"Table dimensions: {gdf.shape[0]} rows, {gdf.shape[1]} cols")

# Convert CRS to WGS84
gdf = gdf.to_crs(epsg=4326)

# Get PostGIS credentials from .env file
load_dotenv(override=True)
user = os.environ["POSTGRES_USER"]
password = os.environ["POSTGRES_PASSWORD"]
host = os.environ["POSTGRES_HOST"]
port = os.environ["POSTGRES_PORT"]
dbname = os.environ["POSTGRES_DB"]

try:
    # Create a connection to PostGIS db
    engine = create_engine(f"postgresql://{user}:{password}@{host}:{port}/{dbname}")
    
    # truncates table in db if it already exists
    # can't drop table because upstream tables depend on it
    query = """
    DO $$ 
    BEGIN 
        IF EXISTS 
            (SELECT FROM pg_tables 
            WHERE schemaname = 'public' 
            AND tablename = 'free_meal_sites') 
        THEN 
            TRUNCATE TABLE public.free_meal_sites; 
        END IF; 
    END $$;"""

    with engine.begin() as conn:
        conn.execute(text(query))

    # add gdf as a table in postgis -- append to truncated table if it existed already
    gdf.to_postgis("free_meal_sites", engine, index=False, if_exists="append")
    print("Added free_meal_sites to database")

except Exception as e:
    print("Database operation failed:", e)
    exit(1)
```

I now run the script from the terminal with `python get_free_meal_data.py`. Now I can go back into the  
database as described earlier and make sure the new table is listed in there by entering `\d free_meal_sites`
(`\d` is short for "describe" in psql). This gives me information on the table columns:

![describe_table](/images/describe_table.png)

## Make a Data Analytics Pipeline With dbt

Now that I have my table in the Postgres database, I'll use dbt to create downstream tables,
incorporate testing, and generate documentation. 

Before diving into the code, it’s important to understand what a model is in dbt. A model is a SQL script 
that transforms data as part of a dbt pipeline run, with its behavior governed by YAML config files. 
In this project, I’ll create one "staging" model and two "mart" models. These terms — "staging" and "mart" —  
are common dbt conventions used to organize and modularize transformation steps. A staging model prepares raw 
source data through operations such as renaming columns, handling null values, casting data types, etc.. It 
serves as a clean "stage" before more complex transformations happen. On the other hand, a mart model reshapes 
and aggregates data into a format that's ready for direct use by actual users like analysts or case workers. 
(Another common convention is "intermediate" models, which fall between staging and mart, but I’ll skip that 
due to the simplicity of the project.) 

Below is the YAML schema file for the staging model along with code for the staging model itself. 
`_schema_staging.yml` includes a `sources` section describing the `free_meal_sites` data that was loaded
into the Postgres database earlier, as well as a `models` section containing built-in tests and documentation
for each column in the `meal_site_staging` model output. 

Notice the line in the `meal_site_staging.sql` file that says `FROM {{ source("raw", "free_meal_sites") }}`. 
This is a Jinja macro used by dbt that grabs the appropriate table according to the `sources` section of the `_schema_staging.yml` file (raw is the name, free_meal_sites is the table). It ultimately translates to 
`FROM raw.free_meal_sites`, but using the macro allows for cross-platform compatibility (different databases
may have different naming syntax) and for dbt to track lineage in the DAG (Directed Acyclic Graph, which shows 
how data flows from source tables through transformations to final outputs), among other things.

`models/staging/_schema_staging.yml`:
```yaml
version: 2

sources:
  - name: raw
    schema: public
    tables:
      - name: free_meal_sites
        description: "Raw data of meal sites imported from external GeoJSON"

models:
  - name: meal_site_staging
    description: "Staged version of free meal sites with cleaned geometry and ingestion metadata."
    columns:
      - name: objectid
        description: "Unique identifier from the source data."
        tests:
          - not_null

      - name: site_name
        description: "Name of the meal site."
        tests:
          - not_null

      - name: category
        description: "Broad classification of the meal site (e.g., Food, Health)."

      - name: category_type
        description: "Specific classification such as 'Senior Meal Site'."

      - name: website
        description: "Website for the meal site (if available)."

      - name: address
        description: "Street address of the site."
        tests:
          - not_null

      - name: zip_code
        description: "ZIP code where the site is located."
        tests:
          - not_null

      - name: geom
        description: "Point geometry in EPSG:4326."
        tests:
          - not_null

      - name: ingestion_day
        description: "Day of the week the data was ingested (0=Sunday, 6=Saturday)."

      - name: ingestion_timestamp
        description: "Human-readable timestamp when the data was ingested." 
```

`models/staging/meal_site_staging.sql`:
```sql
-- meal_site_staging.sql

SELECT
    objectid,
    site_name,
    category,
    category_type,
    website,
    address,
    ST_SetSRID(geometry, 4326)::geometry(Point, 4326) AS geom,
    zip_code,
    EXTRACT(DOW FROM NOW()) AS ingestion_day,
    TO_CHAR(NOW(), 'YYYY-MM-DD HH24:MI:SS') AS ingestion_timestamp
FROM {{ source("raw", "free_meal_sites") }}
WHERE objectid IS NOT NULL
    AND site_name IS NOT NULL
    AND geometry IS NOT NULL
    AND address IS NOT NULL
    AND zip_code IS NOT NULL
```

Mart models are structured similarly to staging. There's a YAML file for documenting and testing
the models and two simple sql files that query from the staging model.

`models/mart/_schema_mart.yml`:
```yaml
version: 2

models:
  - name: meal_sites_by_zip
    description: "Aggregated count of meal sites by ZIP code."
    columns:
      - name: zip_code
        description: "ZIP code grouped on."
        tests:
          - not_null
          - unique

      - name: site_count
        description: "Number of meal sites in the ZIP code."
        tests:
          - not_null


  - name: senior_sites_contact
    description: "Subset of meal sites filtered to only include senior meal sites."
    columns:
      - name: objectid
        description: "Unique identifier from the source data."
        tests:
          - not_null

      - name: site_name
        description: "Name of the senior meal site."
        tests:
          - not_null

      - name: category
        description: "General category of the meal site."

      - name: category_type
        description: "Specific type, should be 'Senior Meal Site'."
        tests:
          - accepted_values:
              values: ['Senior Meal Site']

      - name: website
        description: "Website for the senior site."

      - name: address
        description: "Street address of the senior site."
        tests:
          - not_null

      - name: zip_code
        description: "ZIP code of the senior site."
        tests:
          - not_null

      - name: geom
        description: "Point geometry in EPSG:4326."
        tests:
          - not_null 
```

`models/mart/meal_site_sites_by_zip.sql`:
```sql
-- meal_sites_by_zip.sql

SELECT
    zip_code,
    COUNT(*) AS site_count
FROM {{ ref('meal_site_staging') }}
GROUP BY zip_code
```

`models/mart/senior_sites_contact.sql`:
```sql
-- senior_sites_contact.sql

SELECT
    objectid,
    site_name,
    category,
    category_type,
    website,
    address,
    geom,
    zip_code
FROM {{ ref('meal_site_staging') }}
WHERE category_type = 'Senior Meal Site'
```

In both models, the `FROM` line uses the `ref()` Jinja macro. This macro tells dbt not only 
where to query, but helps it determine dependencies when running the pipeline (e.g., the staging 
model must be run before the mart models can be run). How does dbt know where to scan all these models 
to determine if the name inside `ref()` is indeed there? This is where `dbt_project.yml` comes in. This 
YAML file is required with dbt and can be thought of as the central configuration for the whole project.  
The value corresponding to `model-paths` is where dbt looks for all the models in the project. See my
`dbt_project.yml` with comments here:

```yaml
name: map_proj

version: 1.0                  # version of this project, defined by user
config-version: 2             # config version, defined by dbt

model-paths: ["models"]       # folder(s) where dbt will look for model sql files
target-path: "target"         # where compiled artifacts will be stored

profile: gis_profile          # name of profile in profiles.yml to use for connection

models:                       # this section controls model behavior
  map_proj:                   # default conf for models in map_proj (can be overriden)
    staging:                  # models in staging dir are made... 
      +schema: staging        # in the staging schema and...
      +materialized: view     # materialized as views
    mart:                     # models in mart dir are made... 
      +schema: mart           # in the mart schema and...
      +materialized: table    # materialized as tables
```

The `profile` key in `dbt_project.yml` introduces the concept of the `profiles.yml`, which stores 
the connection configs for all local dbt projects. This file typically lives in the `~/.dbt/` 
directory. Here is the `profiles.yml` for this project:

```yaml
gis_profile:
  target: dev
  outputs:
    dev:
      type: postgres
      host: localhost
      user: postgres
      password: mysecretpassword
      port: 5432
      dbname: gisdb
      schema: public
```

When running dbt, the engine checks the `dbt_project.yml` for the value of the `profile` key (which
in this case is `gis_profile`), and looks for that matching entry in the `profiles.yml` to get the
corresponding database credentials. 

Now that all the pertinent files have been shown and explained, I can see dbt in action. I'll begin by 
entering `dbt debug` to check dependencies and test the database connection, then enter `dbt run` to 
actually run the models:

![dbt_run](/images/dbt_run.png)

That command boots up connections, builds out DAG/documentation, compiles SQL files, runs them in
the correct order, and wraps up. Running `dbt test` will then go through each table or view, ensure 
they exist, and conduct the tests outlined in the model schema files:

![dbt_test](/images/dbt_test.png)

Now I'll go back to my psql terminal and check that the new data are indeed living in Postgres by 
running `\d public_staging.*` and `\d public_mart.*`:

![tables_in_pgres](/images/tables_in_pgres.png)

Let's query one of the tables to double-check values look okay by first entering `\pset format aligned` 
(makes output prettier), followed by the query `SELECT * FROM public_mart.senior_sites_contact LIMIT 5;`:

![query_res](/images/query_res.png)

Results look good! Now, I'll check out the fancy documentation that dbt made for me by running 
`dbt docs generate` followed by `dbt docs serve`. These commands gather the latest metadata and 
open up a webpage with an interactive DAG visualization and searchable documentation. Here are two 
screenshots of what the documentation pages look like:

Mart model page:
![dbt_docs_mart](/images/dbt_docs_mart.png)

Lineage graph:
![dbt_docs_dag](/images/dbt_docs_dag.png)

In this walkthrough, I took a simple open data source about free meal sites and used it to demonstrate some key dbt features. While this example is straightforward, it showcases why dbt has become so popular. dbt brings version control, modular code, automated testing, and seamless documentation to analytics workflows. Instead of having a mess of SQL scripts with unclear dependencies, dbt gives you a structured project where transformations are clearly defined, making it easier to maintain data quality and understand how data flow. For larger orgs with a bunch of sources and complex transformations, I can clearly see how these features become even more valuable. 

