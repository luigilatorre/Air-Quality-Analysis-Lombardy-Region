# ARPA Lombardy Data Analysis: Air Pollution Insights and Data Processing for Optimizing Air Purifier Placement

This repository contains an analysis of weather, air and water pollution data from ARPA Lombardy (Regional Agency for the Protection of the Environment).

Certainly! Here's an updated Table of Contents that includes the main themes from the second part:

## Table of Contents
1. [Introduction](#introduction)
2. [Data Source](#data-source)
3. [Accessing the Data via API](#accessing-the-data-via-api)
4. [Data Extraction](#data-extraction)
5. [Data Enrichment](#data-enrichment)
6. [Merging Datasets](#merging-datasets)
7. [Data Processing and Initial Analysis](#data-processing-and-initial-analysis)
8. [Focusing on Key Pollutants](#focusing-on-key-pollutants)
9. [Further Data Enrichment](#further-data-enrichment)
10. [Final Data Cleaning and Column Renaming](#final-data-cleaning-and-column-renaming)
11. [Saving the Cleaned Dataset](#saving-the-cleaned-dataset)
12. [ARPA Data Analysis](#arpa-data-analysis)
    - [Loading and Preprocessing](#loading-and-preprocessing)
    - [Data Enrichment with Pollutant Thresholds](#data-enrichment-with-pollutant-thresholds)
    - [Seasonal Variations Analysis](#seasonal-variations-analysis)
    - [Geographical Availability Analysis](#geographical-availability-analysis)
13. [Focus on Nitrogen Dioxide (NO2)](#focus-on-nitrogen-dioxide-no2)
    - [Filtering and Saving NO2 Data](#filtering-and-saving-no2-data)
    - [Monthly and Yearly Trends](#monthly-and-yearly-trends)
    - [Peak Period Analysis](#peak-period-analysis)
    - [Station-Specific Analysis](#station-specific-analysis)
      
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

## Data Processing and Initial Analysis

After extracting and merging the datasets, we proceed with data processing and initial analysis. This section covers data cleaning, focusing on key pollutants, and preparing the dataset for further analysis.

### Loading and Initial Inspection

```python
import pandas as pd

df = pd.read_csv("final-merged-arpa.csv")
df_copy = df.copy()

df_copy.info()
```

### Data Type Conversion

We convert date columns to the appropriate datetime format:

```python
df_copy['data'] = pd.to_datetime(df_copy['data'])
df_copy['datastart'] = pd.to_datetime(df_copy['datastart'])
df_copy['datastop'] = pd.to_datetime(df_copy['datastop'])
```

### Handling Missing and Invalid Data

We check for null values and invalid entries:

```python
null_count = df_copy.isnull().sum()
print(null_count)

total_nulls = df_copy.isnull().sum().sum()
print(f"Total null values in the DataFrame: {total_nulls}")

# Count invalid values (-9999) in 'valore' column
inv = (df['valore'] == -9999).sum()
valid = df.shape[0] - inv
print(f"Number of invalid values: {inv}")
print(f"Number of valid values: {valid}")
```

We then remove rows with invalid values:

```python
indices_to_remove = df_copy[df_copy['valore'] == -9999].index
df_copy.drop(indices_to_remove, axis=0, inplace=True)
```

### Focusing on Key Pollutants

We identify and select the key pollutants of interest:

```python
import seaborn as sns
import matplotlib.pyplot as plt

df_grouped = df_copy.groupby("nometiposensore").size().reset_index(name='count')
df_sorted = df_grouped.sort_values("count", ascending=False)

sns.barplot(x="count", y="nometiposensore", data=df_sorted)
plt.show()

pollutants = ['PM10 (SM2005)', 'Particelle sospese PM2.5', 'Ozono', 'Biossido di Zolfo', 'Biossido di Azoto']
df_pol = df_copy[df_copy['nometiposensore'].isin(pollutants)].copy()
```

### Data Enrichment

We add English names and chemical symbols for the pollutants:

```python
df_pol["nometiposensore"] = df_pol["nometiposensore"].map({
    "Biossido di Azoto": "Nitrogen Dioxide",
    "Biossido di Zolfo": "Sulphur Dioxide",
    "Ozono": "Ozone",
    "PM10 (SM2005)": "PM10",
    "Particelle sospese PM2.5": "PM2.5",
})

df_pol["simbolotiposensore"] = df_pol['nometiposensore'].map({
    'Biossido di Azoto': 'NO2', 
    'Biossido di Zolfo': 'SO2', 
    'Ozono': 'O3', 
    'PM10 (SM2005)': 'PM10', 
    'Particelle sospese PM2.5': 'PM2.5'
})
```

### Final Data Cleaning and Column Renaming

We rename columns for clarity and remove unnecessary ones:

```python
df_cleaned = df_pol.rename(columns={
    'idsensore': 'sensor_id', 
    'data': 'date', 
    'valore': 'value', 
    'idoperatore': 'operator_id', 
    'nometiposensore': 'sensor_name', 
    'simbolotiposensore': 'sensor_symbol', 
    'unitamisura': 'unit_measure', 
    'idstazione': 'station_id', 
    'nomestazione': 'station_name', 
    'quota': 'altitude', 
    'provincia': 'province', 
    'comune': 'municipality', 
    'storico': 'historical', 
    'datastart': 'date_start', 
    'datastop': 'date_stop', 
    'lat': 'lat', 
    'lng': 'long'
})
```

### Saving the Cleaned Dataset

Finally, we save our cleaned dataset for further analysis:

```python
df_cleaned.to_csv('df_cleaned.csv', index=False)
```

This cleaned dataset now contains only the relevant pollutants and necessary information for our analysis on determining the optimal location and operation times for an Air Purifier in the Lombardy region.

## ARPA Data Analysis

In this section, we analyze the pollutants in the dataset.

### Loading and Preprocessing

We load the cleaned dataset and perform necessary type conversions:

```python
import pandas as pd
import seaborn as sns

df = pd.read_csv('df_cleaned.csv', index_col=False)
df1 = df.copy()

df1['date'] = pd.to_datetime(df1['date'], format='ISO8601')
df1['date_start'] = pd.to_datetime(df1['date_start'], format='ISO8601')
df1['date_stop'] = pd.to_datetime(df1['date_stop'], format='ISO8601')
```

### Data Enrichment with Pollutant Thresholds

We load and process a dataset containing thresholds for each pollutant:

```python
df_th = pd.read_csv("pollutant_thresholds_arpa.csv")
df_th = df_th[df_th['Inquinante'].isin(['Biossido di Azoto', 'Biossido di Zolfo', 'Ozono', 'PM10'])]
```

### Seasonal Variations Analysis

We analyze seasonal variations for each pollutant:

```python
df1.loc[:, "year"] = df1["date"].dt.year
df1.loc[:, "month"] = df1["date"].dt.month

df_time_group = df1.groupby(["year", "month", "sensor_name"], as_index=False)["value"].agg("mean")

# Plotting code here (using seaborn)
```

### Geographical Availability Analysis

We investigate data availability for each pollutant across all stations:

```python
df_loc = df.groupby(['station_name', 'sensor_name'], as_index=False).size().reset_index()
df_loc_piv = df_loc.pivot(index='station_name', columns='sensor_name', values='size')

# Heatmap visualization code here
```

## Focus on Nitrogen Dioxide (NO2)

We decide to focus on NO2 for further analysis due to its prevalence and impact.

### Filtering and Saving NO2 Data

```python
df_no2 = df1[df1['sensor_name']=='Nitrogen Dioxide']
df_no2.to_csv('df_no2.csv', index=False)
```

### Monthly and Yearly Trends

We visualize monthly and yearly trends for NO2:

```python
sns.lineplot(df_no2, x='month', y='value', hue='year')
```

### Peak Period Analysis

We analyze NO2 levels during peak periods (October to March):

```python
avg_value_by_station_in_peak_periods = df_no2.loc[(df_no2.month >= 10) | (df_no2.month <= 3)].groupby(
    'station_name'
).agg(
    {
        'value': 'mean',
        'date': 'count'
    }
)
```

### Station-Specific Analysis

We perform a detailed analysis for a specific station (Milano v.Marche):

```python
sns.lineplot(df_no2.loc[df_no2.station_name.eq('Milano v.Marche')], x='month', y='value', hue='year')
```

This analysis provides insights into NO2 pollution patterns, helping to determine the most effective locations and times for air purifier operation in the Lombardy region.
