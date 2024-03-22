# Feast: A Feature Store for Machine Learning

*This is an overview of the main Feast functionalities.
Please, refer to the [official documentation](https://docs.feast.dev/v/master) for additional details.*

## Overview

### What is a Feature Store?

- A feature store acts as a centralized repository for managing and serving features to ML models. 
- Features provide the necessary input variables for algorithms to learn at training time and make predictions at inference time.
- Managing features involves disparate data sources, complex pipelines, and fragmented processes, potentially leading to inefficiencies, inconsistencies, and increased development overheads.
- A feature store addresses these challenges by offering a unified platform to consistently define, store, and serve features.

### What is Feast (and what it is not)?

- Feast is an **open source feature store** that re-uses existing infrastructure to manage and serve machine learning features to realtime models.
- Today, Feast primarily addresses **timestamped structured data**.
- An **offline store** is used to process historical data for batch scoring and model training.
- An **online store** is instead used for real-time inference.
- Both at training and scoring time, data leakage is avoided by generating point-in-time correct feature sets.
- Feast is neither an ETL/ELT nor an orchestration tool. These steps should be handled upstream by ad-hoc tools such as dbt and Airflow.
- Feast is not meant to replace data warehouses / databases. It is a layer to serve data from existing data sources.
- Feast enables users to push streaming features, but does not pull from streaming sources or manage streaming pipelines.

## Architecture

Feast architecture consists of 6 main components:

- **Registry**: it is used to store feature definitions registrered with the feature store.
- **Offline store**: it persists batch data that has been ingested into Feast. 
  - used to generate training datasets and for batch scoring.
- **Online store**: database that stores only the latest feature values.
  - populated through materialization jobs or stream ingestion.
- **Batch Materialization Engine**: it is used to load data into the online store from the offline store.
- **Stream Processor**: it is used to ingest feature data from streams and write it into the online or offline stores.
- **Python SDK/CLI**: to manage features, build and retrieve data from the offline store, materialize features into the
  online store, retrieve data from the online store.

## Main Concepts

- The top-level namespace within Feast is a **project**.
- Users can define **feature views** within a project.
  - Features relate to one or more entities.
  - Feature views must always have a data source, used when generating the training dataset and materializing feature 
    values in the online store.
- Multiple feature views can be grouped and exposed via a **feature service**.

### Data Sources

Feast is in charge of loading time-stamped tabular data sources (raw data in a database) and retrieve/serve features 
from the sources. There are 3 types of suppoerted sources:

- **Batch Data Sources**: living in data warehouses / data lakes.
  - `materialize_incremental` command fetches the latest values for all entities in the batch source and ingests these 
    values into the online store.
  - materialization can be called through the CLI or scheduled at regular intervals.
- **Stream Data Sources**: Feast does not natively integrate data streaming, but it facilitates streaming via:
  - **Push Sources**: allow users to push features into Feast for training, batch scoring (offline store) and/or 
    realtime serving (online store).
    - Push sources must have a batch source specified (used to retrieve historical features). 
  - **[Alpha] Stream sources**: allow users to register metadata from Kafka/Kinesis sources. The user is responsible to 
    ingest from these sources.
- **[Experimental] Request sources**: data available at request time only, useful for on-demand feature views which 
  allow *light* feature engineering and combining features across sources.

```python
driver_stats_source = FileSource(
    name="driver_hourly_stats_source",
    path="~/pycharm_projects/feast_demo/feature_repo/data/driver_stats.parquet",
    timestamp_field="event_timestamp",
    created_timestamp_column="created",
)

# Defines a way to push data (to be available offline, online or both) into Feast.
driver_stats_push_source = PushSource(
    name="driver_stats_push_source",
    batch_source=driver_stats_source,
)
```

### Entity

An entity is a collection of semantically related features. 
Users define entities to map to the domain of their use case.

```python
driver = Entity(
    name='driver', 
    join_keys=['driver_id']
)
```

- The entity name is used to uniquely identify the entity.
- The join key is used to identify the primary key on which feature values are joined together during feature retrieval.

### Feature Views

A feature view represents a group of features in a data source, related to zero or more entities.

```python
driver_stats_fv = FeatureView(
    name="driver_hourly_stats",  # two feature views in a single project cannot have the same name
    entities=[driver],  # related entity
    ttl=timedelta(days=365),  # time to live (optional)
    schema=[
        Field(name="conv_rate", dtype=Float32),
        Field(name="acc_rate", dtype=Float32),
        Field(name="avg_daily_trips", dtype=Int64, description="Average daily trips"),
    ],  # schema of the feature view (fields and data types)
    online=True,  # this feature view will be served for online retrieval
    source=driver_stats_source,  # data source
    tags={"team": "driver_performance"},  # metadata
)
```

### Feature Services

A feature service is an object that represents a logical group of features from one or more feature views. 
Users can expect to create one feature service per machine learning model version. Naming conventions should be agreed.

```python
driver_activity_v1 = FeatureService(
    name="driver_activity_v1",
    features=[driver_stats_fv]
)
```

The feature service can be used to retrieve both offline and online features:

```python
from feast import FeatureStore
feature_store = FeatureStore('.')  # conf in local dir
feature_service = feature_store.get_feature_service("driver_activity_v1")
entity_df = ...

training_df = feature_store.get_historical_features(
    entity_df=entity_df,  
    features=feature_store.get_feature_service("driver_activity_v1"),
).to_df()

online_features = feature_store.get_online_features(
    entity_rows=[  # no timestamp required because only the most recent features are retrieved for the following entity values
        {"driver_id": 1001},
        {"driver_id": 1004},
    ],
    features=feature_store.get_feature_service("driver_activity_v1"),
).to_df()
```

## Getting started

### Installation

To install Feast, simply use pip:

```bash
pip install feast
```

- Specific dependencies are required when using Feast on top of Snowflake, BigQuery, RedShift, etc...

### Create a feature repository

To create a feast repository, run the following command:

```bash
feat init <feast_repo_name>
```

This command generates a feature_repo directory containing:

- `data/`: directory containing raw demo parquet data
- `example_repo.py`: demo feature definitions
- `feature_store.yaml`: demo setup configuring where data sources are
- `test_workflow.py`: explain how to run all key Feast commands, including defining, retrieving, and pushing features

#### Feature store yaml configuration

```yaml
project: my_project
registry: data/registry.db
provider: local
offline_store: file
online_store:
  type: sqlite
  path: data/online_store.db
entity_key_serialization_version: 2
```

- `project`: feature project name
- `registry`: central catalog of all the feature definitions and their related metadata. It supports four different backends:
	- `Local`: Used as a local backend for storing the registry during development
	- `S3`: Used as a centralized backend for storing the registry on AWS
	- `GCS`: Used as a centralized backend for storing the registry on GCP
	- [Alpha] `Azure`: Used as centralized backend for storing the registry on Azure Blob storage.
- `provider`: the provider value sets default offline and online stores. Valid values are:
	- `local`: use a SQL registry or local file registry. By default, use a file / Dask based offline store + SQLite online store
	- `gcp`: use a SQL registry or GCS file registry. By default, use BigQuery (offline store) + Google Cloud Datastore (online store)
	- `aws`: use a SQL registry or S3 file registry. By default, use Redshift (offline store) + DynamoDB (online store)
- `offline_store`: compute layer to process historical data. It is used for:
	- Building training datasets from time-series features
	- Materializing (loading) features into an online store
- `online_store`: low latency store of the latest feature values (for powering real-time inference)
	- Feature values are loaded from data sources into the online store through `materialization`, which can be triggered through the materialize command.
	- Features can also be written directly to the online store via push sources.

### Register feature definitions and deploy the feature store

To have Feast deploy your infrastructure, run feast apply from your command line while inside a feature repository:

```bash
feast apply
```

## Demo

[Jupyter Notebook](./notebooks/demo.ipynb)