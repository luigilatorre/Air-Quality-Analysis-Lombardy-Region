# ARPA Data Analysis: Data Extraction and Initial Cleaning

This repository contains an analysis of weather, air and water pollution data from ARPA Lombardy (Regional Agency for the Protection of the Environment).

## Table of Contents
1. [Introduction](#introduction)
2. [Data Source](#data-source)
3. [Accessing the Data via API](#accessing-the-data-via-api)
4. [Data Extraction](#data-extraction)
5. [Data Enrichment](#data-enrichment)
6. [Merging Datasets](#merging-datasets)

## Introduction

ARPA Lombardia's air quality detection network consists of fixed stations that provide continuous data at regular time intervals. The pollutant species monitored include NOX, SO2, CO, O3, PM10, PM2.5, and benzene.

## Data Source

The main dataset used in this analysis can be found [here](https://www.dati.lombardia.it/Ambiente/Dati-sensori-aria-dal-2018/g2hp-ar79/about_data). It contains air quality data from 2018 to the present day.

## Accessing the Data via API

The data can be accessed via an API endpoint. The base URL for the API is:

```
https://www.dati.lombardia.it/resource/g2hp-ar79.csv
```

To retrieve more than the default 1000 rows, we can use the `$limit` and `$offset` parameters:

```
https://www.dati.lombardia.it/resource/g2hp-ar79.csv?$limit=50000&$offset=0
```

## Data Extraction

Q: How can we download all the data from the API, given the 50,000 row limit per request?

A: We can use a loop to make multiple API calls, incrementing the offset each time. Here's the code to do that:

```python
import pandas as pd

url = "https://www.dati.lombardia.it/resource/g2hp-ar79.csv"
list_data = []  # Initialize an empty list to store chunks of data
offset_n = 0  # Initialize the offset parameter to zero
large_number = 18000000  # Set a large number (should be larger than the rows in the dataset)

while offset_n < large_number:
    new_url = f"{url}?$limit=50000&$offset={offset_n}"
    temp_df = pd.read_csv(new_url)
    
    if temp_df.empty:
        break
    
    list_data.append(temp_df)
    offset_n += 50000

final_df = pd.concat(list_data, ignore_index=True)
print(final_df.head())

final_df.to_csv('final_data.csv', index=False)
```

## Data Enrichment

Q: How can we enrich our dataset with additional information about the sensors?

A: We can download a lookup table containing information about each sensor:

```python
stazioni = pd.read_csv("stazioni-aria.csv")
stazioni.head(1)
```

## Merging Datasets

Q: How do we combine the main dataset with the sensor information?

A: We can merge the two datasets using the 'idsensore' column as the key:

```python
merged_df = pd.merge(final_df, stazioni, on="idsensore", how="left")
merged_df.head(1)
merged_df.shape

merged_df.to_csv("final-merged-arpa.csv")
```

