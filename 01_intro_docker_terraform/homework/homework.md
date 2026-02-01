# Module 1 Homework: Docker & SQL
---
[Homework assignment](https://github.com/DataTalksClub/data-engineering-zoomcamp/blob/main/cohorts/2026/01-docker-terraform/homework.md)
In this homework we'll prepare the environment and practice Docker and SQL

## Question 1. Understanding Docker images
Run docker with the python:3.13 image. Use an entrypoint bash to interact with the container.

What's the version of pip in the image?
> Answer
```bash
pip 25.3
```

> Command
```bash
docker run -it --rm --entrypoint bash python:3.13
pip --version
```

## Question 2. Understanding Docker networking and docker-compose
> Answer
```bash
db:5432
```

## Question 3. Counting short trips
For the trips in November 2025 (lpep_pickup_datetime between '2025-11-01' and '2025-12-01', exclusive of the upper bound), how many trips had a trip_distance of less than or equal to 1 mile?

### Prepare Postgres

We will use the green taxi trips data for November 2025:

```bash
wget https://d37ci6vzurychx.cloudfront.net/trip-data/green_tripdata_2025-11.parquet
```

You will also need the dataset with zones:

```bash
wget https://github.com/DataTalksClub/nyc-tlc-data/releases/download/misc/taxi_zone_lookup.csv
```
```bash
# Step 1. Create folder for Postgres to store data in

mkdir ~/ny_taxi_project
cd ~/ny_taxi_project

# Step 2. Create tables in Postgres
  psql -h localhost -U root -d ny_taxi
    
    # Download CSV and parquet file
    curl -L -O https://github.com/DataTalksClub/nyc-tlc-data/releases/download/misc/taxi_zone_lookup.csv

    curl -O https://d37ci6vzurychx.cloudfront.net/trip-data/green_tripdata_2025-11.parquet

    # Verify files
    ls -lh
    head taxi_zone_lookup.csv

    # Check schema of green_tripdata table (in separate terminal)
    pip install pandas pyarrow
    import pandas as pd
    from sqlalchemy import create_engine
    engine = create_engine("postgresql://root:root@host.docker.internal:5433/ny_taxi")
    df = pd.read_parquet("green_tripdata_2025-11.parquet")
    print(df.dtypes)

# Step 3. Start Postgres

docker run -it --rm \
  -e POSTGRES_USER="root" \
  -e POSTGRES_PASSWORD="root" \
  -e POSTGRES_DB="ny_taxi" \
  -v ny_taxi_postgres_data:/var/lib/postgresql/data \
  -p 5432:5432 \
  postgres:18

# Check if Postgres is running
docker ps

# Step 4. Create two tables

    # Enter postgres container:
    docker exec -it postgres psql -U root -d ny_taxi

    # Create taxi_zones table:

    CREATE TABLE taxi_zones (
        locationid INTEGER,
        borough TEXT,
        zone TEXT,
        service_zone TEXT
    );

    # Create green_tripdata table:

    CREATE TABLE green_tripdata (
        vendorid INTEGER,
        lpep_pickup_datetime TIMESTAMP,
        lpep_dropoff_datetime TIMESTAMP,
        store_and_fwd_flag TEXT,
        ratecodeid INTEGER,
        pulocationid INTEGER,
        dolocationid INTEGER,
        passenger_count INTEGER,
        trip_distance FLOAT,
        fare_amount FLOAT,
        extra FLOAT,
        mta_tax FLOAT,
        tip_amount FLOAT,
        tolls_amount FLOAT,
        improvement_surcharge FLOAT,
        total_amount FLOAT,
        payment_type INTEGER,
        trip_type INTEGER,
        congestion_surcharge FLOAT
    );

# Step 5. Load data into Postgres
    # Load taxi_zones (csv)
        # Copy CSV into container
        docker cp taxi_zone_lookup.csv postgres:/var/lib/postgresql/taxi_zone_lookup.csv
        
        # Run \copy to load data:
        docker exec -i postgres psql -U root -d ny_taxi -c "\copy taxi_zones FROM '/var/lib/postgresql/taxi_zone_lookup.csv' CSV HEADER;"

        # Verify if loaded successfully:
        docker exec -it postgres psql -U root -d ny_taxi -c "SELECT COUNT(*) FROM taxi_zones;"

    # Load green_tripdata (parquet)
        # Run Python container with project folder mounted
        docker run -it --rm \
        -v ~/ny_taxi_project:/app \
        python:3.13 bash

        # Install Python packages
        pip install pandas pyarrow sqlalchemy psycopg2-binary

        # Create ingestion script
        /app/ingest_parquet.py:
        import pandas as pd
        from sqlalchemy import create_engine

        # Connect to Postgres container and load data
        engine = create_engine("postgresql://root:root@host.docker.internal:5433/ny_taxi")


        df = pd.read_parquet("green_tripdata_2025-11.parquet")
        df.to_sql("green_tripdata", engine, if_exists="append", index=False)

```

>Command:
```sql
SELECT COUNT(1)
FROM green_tripdata
WHERE lpep_pickup_datetime BETWEEN '2025-11-01' AND '2025-12-01'
AND trip_distance <= 1;
```
>Answer:
```
8,007
```

## Question 4. Longest trip for each day
Which was the pick up day with the longest trip distance? Only consider trips with trip_distance less than 100 miles (to exclude data errors).

Use the pick up time for your calculations.
>Command:
```sql
SELECT
    DATE_TRUNC('day', lpep_pickup_datetime) AS pickup_date,
    trip_distance
FROM green_tripdata
WHERE lpep_pickup_datetime BETWEEN '2025-11-01' AND '2025-12-01'
  AND trip_distance < 100
ORDER BY trip_distance DESC
LIMIT 1;
```
>Answer:
```
2025-11-14
```

## Question 5. Biggest pickup zone
Which was the pickup zone with the largest total_amount (sum of all trips) on November 18th, 2025?

>Command:
```sql
WITH total_amount_per_pu_location AS (
    SELECT 
        "PULocationID" AS pu_location_id,
        SUM(total_amount) AS sum_amount
    FROM green_tripdata
    WHERE DATE(lpep_pickup_datetime) = '2025-11-18'
    GROUP BY "PULocationID"
)
SELECT
    tz."Zone",
    gt.sum_amount
FROM total_amount_per_pu_location AS gt
LEFT JOIN taxi_zones AS tz
ON gt.pu_location_id = tz."LocationID"
ORDER BY gt.sum_amount DESC
LIMIT 1;
```
>Answer:
```
East Harlem North
```

## Question 6. Largest tip
For the passengers picked up in the zone named "East Harlem North" in November 2025, which was the drop off zone that had the largest tip?

>Command:
```sql
SELECT
    tz_drop."Zone" AS dropoff_zone,
    gt.tip_amount AS total_tips
FROM green_tripdata AS gt
LEFT JOIN taxi_zones AS tz_pick
    ON gt."PULocationID" = tz_pick."LocationID"
LEFT JOIN taxi_zones AS tz_drop
    ON gt."DOLocationID" = tz_drop."LocationID"
WHERE tz_pick."Zone" = 'East Harlem North'
  AND gt.lpep_pickup_datetime BETWEEN '2025-11-01' AND '2025-11-30'
ORDER BY total_tips DESC
LIMIT 1
;
```
>Answer:
```
LaGuardia Airport
```

## Question 7. Terraform Workflow
Which of the following sequences, respectively, describes the workflow for:

1. Downloading the provider plugins and setting up backend,
2. Generating proposed changes and auto-executing the plan
3. Remove all resources managed by terraform

>Answer:
```
terraform init, terraform apply -auto-approve, terraform destroy
```