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
docker run -it --rm \
  -e POSTGRES_USER="root" \
  -e POSTGRES_PASSWORD="root" \
  -e POSTGRES_DB="ny_taxi" \
  -v ny_taxi_postgres_data:/var/lib/postgresql/data \
  -p 5432:5432 \
  postgres:18

  # Step 2. Create tables in Postgres
  psql -h localhost -U root -d ny_taxi

  # Step 3. Load this data into Postgres
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