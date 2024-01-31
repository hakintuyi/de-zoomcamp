# Docker + Postgres

## Commands

Below are the commands used as taught from the #dezoomcamp video

Downloading the data

```bash
wget https://github.com/DataTalksClub/nyc-tlc-data/releases/download/green/green_tripdata_2019-09.csv.gz
```

### Running Postgres with Docker

#### Windows

> Note: Ensure docker desktop is running before performing the commands below

Running Postgres on Windows (note the full path)

```bash
docker run -it \
  -e 'POSTGRES_USER=agent' \
  -e 'POSTGRES_PASSWORD=password1' \
  -e 'POSTGRES_DB=ny_taxi2019' \
  -v 'c:/Users/USER/Desktop/DE/projects/ny_taxi/ny_taxi2019_pg-data:/var/lib/postgresql/data' \
  -p 5432:5432 \
  postgres:16
```

### CLI for Postgres

- Windows Subsystem for Linux (WSL2) was used along side with to execute this project due to absence of `pgcli` on my Windows OS

Installing `pgcli` using conda in wsl2

```bash
conda install -c conda-forge pgcli
```

Using `pgcli` to connect to Postgres

```bash
pgcli -h localhost -p 5432 -u agent -d ny_taxi2019
```

### pgAdmin

Running pgAdmin

```bash
docker run -it \
  -e PGADMIN_DEFAULT_EMAIL="admin@admin.com" \
  -e PGADMIN_DEFAULT_PASSWORD="root" \
  -p 8080:80 \
  dpage/pgadmin4
```

### Running Postgres and pgAdmin together

Create a network

```bash
docker network create pg_network
```

Run Postgres

```bash
docker run -it \
  -e 'POSTGRES_USER=agent' \
  -e 'POSTGRES_PASSWORD=password1' \
  -e 'POSTGRES_DB=ny_taxi2019' \
  -v 'c:/Users/USER/Desktop/DE/projects/ny_taxi/ny_taxi2019_pg-data:/var/lib/postgresql/data' \
  -p 5432:5432 \
  --network=pg_network \
  --name pg_database \
  postgres:13
```

Run pgAdmin

```bash
docker run -it \
  -e 'PGADMIN_DEFAULT_EMAIL=admin@admin.com' \
  -e 'PGADMIN_DEFAULT_PASSWORD=root' \
  -p 8080:80 \
  -v 'c:/Users/USER/Desktop/DE/projects/ny_taxi/pgadmin-data:/var/lib/pgadmin' \
  --network=pg_network \
  --name 'pg_admin' \
  dpage/pgadmin4
```

### Data ingestion

Running locally

```bash
URL="https://github.com/DataTalksClub/nyc-tlc-data/releases/download/green/green_tripdata_2019-09.csv.gz"

python ingest_data.py \
  --user=agent \
  --password=password1 \
  --host=localhost \
  --port=5432 \
  --db=ny_taxi2019 \
  --table_name=green_taxi_data \
  --url=${URL}
```

Build the image

```bash
docker build -t taxi_ingest:v001 .
```

Run the script with Docker

```bash
URL="https://github.com/DataTalksClub/nyc-tlc-data/releases/download/green/green_tripdata_2019-09.csv.gz"

docker run -it \
  --network=pg_network \
  taxi_ingest:v001 \
    --user=agent \
    --password=password1 \
    --host=pg_database \
    --port=5432 \
    --db=ny_taxi2019 \
    --table_name=green_taxi_data \
    --url=${URL}
```

### Docker-Compose

Run it:

```bash
docker-compose up
```

Run in detached mode:

```bash
docker-compose up -d
```

Shutting it down:

```bash
docker-compose down
```

For the full [docker-compose.yml](docker-compose.yml)

### SQL QUERY

Join tables on zone location

```bash
SELECT
	lpep_pickup_datetime AS "pickup_day",
	lpep_dropoff_datetime AS "dropoff_day",
	CONCAT(zpu."Borough", ' / ', zpu."Zone") AS "pickup_location",
	CONCAT(zdo."Borough", ' / ', zdo."Zone") AS "dropoff_location",
	total_amount
FROM
	green_taxi_trips t,
	zones zpu,
	zones zdo
WHERE
	t."PULocationID" = zpu."LocationID" AND
	t."DOLocationID" = zdo."LocationID"
LIMIT 100
```

**Count records**

How many taxi trips were totally made on September 18th 2019?

Tip: started and finished on 2019-09-18.

Remember that lpep_pickup_datetime and lpep_dropoff_datetime columns are in the format timestamp (date and hour+min+sec) and not in date.

```bash
SELECT
	CAST(lpep_pickup_datetime AS date) AS "pickup_date",
	CAST(lpep_dropoff_datetime AS date) AS "dropoff_date",
	COUNT(1) AS "count"
FROM
	green_taxi_trips t
WHERE
	CAST(lpep_pickup_datetime AS date) = '2019-09-18' AND
	CAST(lpep_dropoff_datetime AS date) = '2019-09-18'
GROUP BY
	1, 2
ORDER BY
	"count" DESC;
```

**Longest trip for each day**

Which was the pick up day with the longest trip distance? Use the pick up time for your calculations.

Tip: For every trip on a single day, we only care about the trip with the longest distance.

```bash
SELECT
	CAST(lpep_pickup_datetime AS date) AS "pickup_date",
	MAX(trip_distance) AS "max_distance"
FROM
	green_taxi_trips t
WHERE
	CAST(lpep_pickup_datetime AS date) BETWEEN '2019-09-16' AND '2019-09-26'
GROUP BY
	"pickup_date"
ORDER BY
	"max_distance" DESC;
```

**Three biggest pick up Boroughs**

Consider lpep_pickup_datetime in '2019-09-18' and ignoring Borough has Unknown

Which were the 3 pick up Boroughs that had a sum of total_amount superior to 50000

```bash
SELECT
	CAST(lpep_pickup_datetime AS date) AS "pickup_date",
	zpu."Borough" AS "district",
	COUNT(1) AS "count",
	SUM(total_amount) AS "sum_total_amount"
FROM
	green_taxi_trips t,
	zones zpu
WHERE
	CAST(lpep_pickup_datetime AS date) = '2019-09-18'
GROUP BY
	"pickup_date", "DOLocationID", "district"
ORDER BY
	"count" DESC;
```

**Largest tip**

For the passengers picked up in September 2019 in the zone name Astoria which was the drop off zone that had the largest tip.

```bash
SELECT
	CAST(lpep_pickup_datetime AS date) AS "pickup_date",
	CAST(lpep_dropoff_datetime AS date) AS "dropoff_date",
	CONCAT(zpu."Borough", ' / ', zpu."Zone") AS "pickup_location",
	zdo."Zone" AS "dropoff_zone",
	MAX(tip_amount) AS "largest_tips",
	MAX(passenger_count) AS "max_passenger"
FROM
	green_taxi_trips t,
	zones zpu,
	zones zdo
WHERE
	CAST(lpep_pickup_datetime AS date) BETWEEN '2019-09-01' AND '2019-09-30' AND
	t."PULocationID" = zdo."LocationID" AND
	t."DOLocationID" = zdo."LocationID" AND
	zpu."Zone" = 'Astoria' AND
	zdo."Zone" IN ('Central Park', 'Jamaica', 'JFK Airport', 'Long Island City/Queens Plaza')
GROUP BY
	"pickup_date", "dropoff_date", "pickup_location", "dropoff_zone"
ORDER BY
	"largest_tips" DESC
```
