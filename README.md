# Data Pipeline with Reddit, Airflow, Celery, Postgres, S3, AWS Glue, Athena, and Redshift

This project provides a comprehensive data pipeline solution to extract, transform, and load (ETL) Reddit data into a Redshift data warehouse. The pipeline leverages a combination of tools and services including Apache Airflow, Celery, PostgreSQL, Amazon S3, AWS Glue, Amazon Athena, and Amazon Redshift.

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [System Setup](#system-setup)

## Overview

The pipeline is designed to:

1. Extract data from Reddit using its API.
2. Store the raw data into an S3 bucket from Airflow.
3. Transform the data using AWS Glue and Amazon Athena.
4. Load the transformed data into Amazon Redshift for analytics and querying.

## Architecture
![RedditDataEngineering.png](assets%2FRedditDataEngineering.png)
1. **Reddit API**: Source of the data.
2. **Apache Airflow & Celery**: Orchestrates the ETL process and manages task distribution.
3. **PostgreSQL**: Temporary storage and metadata management.
4. **Amazon S3**: Raw data storage.
5. **AWS Glue**: Data cataloging and ETL jobs.
6. **Amazon Athena**: SQL-based data transformation.
7. **Amazon Redshift**: Data warehousing and analytics.

## High-Level Design (HLD)

```mermaid
flowchart LR
    reddit[(Reddit API)]

    subgraph orchestration[Airflow on Docker Compose]
        scheduler[Airflow Scheduler]
        broker[(Redis Broker)]
        worker[Celery Worker]
        web[Airflow Webserver]
        metadata[(PostgreSQL
        Airflow Metadata)]
    end

    subgraph pipeline[ETL Pipeline]
        extract[Extract and transform Reddit posts]
        csv[(Local CSV
        data/output)]
        upload[Upload raw CSV]
    end

    subgraph aws[AWS Analytics Layer]
        s3[(Amazon S3
        raw/)]
        glue[AWS Glue]
        athena[Amazon Athena]
        redshift[(Amazon Redshift)]
    end

    web --> metadata
    scheduler --> metadata
    scheduler --> broker
    broker --> worker
    worker --> extract
    reddit --> extract
    extract --> csv
    csv --> upload
    upload --> s3
    s3 --> glue
    glue --> athena
    athena --> redshift
```

## Low-Level Design (LLD)

```mermaid
sequenceDiagram
    autonumber
    participant S as Airflow Scheduler
    participant D as reddit_dag.py
    participant R as Reddit API / PRAW
    participant P as reddit_pipeline.py
    participant E as reddit_etl.py
    participant F as Local CSV
    participant X as Airflow XCom
    participant U as aws_s3_pipeline.py
    participant A as aws_etl.py / s3fs
    participant B as Amazon S3

    S->>D: Trigger etl_reddit_pipeline (@daily)
    D->>P: reddit_pipeline(file_name, subreddit, time_filter, limit)
    P->>E: connect_reddit(CLIENT_ID, SECRET, user_agent)
    E-->>P: Reddit client
    P->>E: extract_posts(client, "dataengineering", "day", 100)
    E->>R: Request top subreddit posts
    R-->>E: Post objects
    E-->>P: List of selected POST_FIELDS
    P->>E: transform_data(DataFrame)
    Note over E: Convert timestamps, booleans, author,
    Note over E: comment counts, scores, and titles
    P->>E: load_data_to_csv(DataFrame, output_path)
    E->>F: Write reddit_YYYYMMDD.csv
    P-->>D: Return CSV path
    D->>X: Store task return_value
    D->>U: upload_s3_pipeline(ti)
    U->>X: Pull reddit_extraction return_value
    X-->>U: CSV path
    U->>A: connect_to_s3()
    A-->>U: S3 filesystem client
    U->>A: create_bucket_if_not_exist(bucket)
    U->>A: upload_to_s3(path, bucket, filename)
    A->>B: Put object at bucket/raw/filename
    B-->>A: Upload result
```


## Prerequisites
- AWS Account with appropriate permissions for S3, Glue, Athena, and Redshift.
- Reddit API credentials.
- Docker Installation
- Python 3.9 or higher

## System Setup
1. Clone the repository.
   ```bash
    git clone https://github.com/0xs1d/Reddit-airflow-aws.git
   ```
2. Create a virtual environment.
   ```bash
    python3 -m venv venv
   ```
3. Activate the virtual environment.
   ```bash
    source venv/bin/activate
   ```
4. Install the dependencies.
   ```bash
    pip install -r requirements.txt
   ```
5. Rename the configuration file and the credentials to the file.
   ```bash
    mv config/config.conf.example config/config.conf
   ```
6. Starting the containers
   ```bash
    docker-compose up -d
   ```
7. Launch the Airflow web UI.
   ```bash
    open http://localhost:8080
   ```
