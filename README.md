# GDELT Conflict Dataset 1.0 (2021)

[Kaggle Dataset.](https://www.kaggle.com/vladproex/gdelt-conflict-dataset-10)

#### Disclaimer

I built this dataset as a personal project. It comes with no guarantees. Feel free to use it for your own project or article, but please acknowledge me. And remember to include a citation to the GDELT Project and their [website.](https://www.gdeltproject.org/)

I may release new versions in the future, but no guarantees.

If you analyze the data, please let me know. I would love to see what you find out.  

## Introduction

This repository documents the sourcing of the GDELT Conflict Dataset 2021. 

The [GDELT Project](https://www.gdeltproject.org/) provides constant monitoring of news media across the world. Its archives stretch back to January 1, 1979. The database is supposedly being updated every 15 minutes. Their mission is to "to construct a catalog of human societal-scale behavior and beliefs across all countries of the world".

The GDELT Conflict Dataset leverages GDELT to examine the evolution of conflict in the past 40 years. It aggregates information on more than 83 million events extracted from media reports in 258 countries for the period 1979-2021. The events are grouped in 32 categories describing conflict actions at various scales, such as `Confiscate property, 'Carry out suicide bombing', 'Occupy territory'.

Hopefully this dataset can help us gain better insight in the evolution of conflict. 

## Data Model

The GDELT Project is organized around the CAMEO event coding ontology. Every event is coded as "[Actor 1] does [Action] on [Actor 2]". 

CAMEO identifies a wide variety of event actions divided in macro-categories. For the purpose of this dataset I selected the following categories:

```
17 COERCE
18 ASSAULT
19 FIGHT
20 USE UNCONVENTIONAL MASS VIOLENCE
```

The last category had no returns on the GDELT database. 

This dataset is aggregated by event code, year and country. 

For example, it allows you to see the number of 'Assassinate' events that occurred in USA in 1985:

```python
import pandas as pd

df = pd.read_csv('gdelt_conflict/gdelt_conflict_1_0.csv')
result = df[(df.Year == 1985) &
            (df.CountryCode == 'US') &
            (df.EventCode == 186)]

print(result.SumEvents) 
```

The dataset includes additional features from GDELT: the Goldstein scale score, the average and total number of mentions, and the average of the average tone. Read below to find out more about these features.

For more information see:

[CAMEO event codes](https://www.gdeltproject.org/data/lookups/CAMEO.eventcodes.txt).

[CAMEO Codebook](http://data.gdeltproject.org/documentation/CAMEO.Manual.1.1b3.pdf).

[GDELT documentation](https://www.gdeltproject.org/data.html#documentation).

## Extraction

I accessed the GDELT database Google Big Query as `gdelt-bq`. I used this query:

```sql
SELECT Year, 
	   ActionGeo_CountryCode AS CountryCode,
	   EventRootCode,
	   EventCode,
	   COUNT(GLOBALEVENTID) AS SumEvents,
	   ANY_VALUE(GoldsteinScale) AS GoldsteinScale,
	   AVG(NumMentions) as AvgNumMentions,
	   SUM(NumMentions) as SumNumMentions,
	   AVG(AvgTone) as AvgAvgTone,
FROM `gdelt-bq.full.events`
WHERE 
	EventRootCode IN ("17", "18", "19", "20") 
	AND Year >= 1979
GROUP BY Year, ActionGeo_CountryCode, EventRootCode, EventCode
ORDER BY Year
```

If you're wondering why I used `ANY_VALUE()` on `GoldsteinScale` , it's because the Goldstein scale score is the same for each `EventCode`. 

## Processing

In the processing phase I added labels for countries and event codes and removed a minority (~3%) of records with null values. 

I also added the feature `TotalEvents`. GDELT provides normalization files recording the total number of events broken down by year and country. According to GDELT, "this is important for normalization tasks, to compensate the exponential increase in the availability of global news material over time." Find more in the [GDELT documentation](https://www.gdeltproject.org/data.html#documentation).

**Warning**. The normalization files provided by GDELT are built for GDELT 1.0. However, I'm almost sure that this dataset comes from GDELT 2.0. The documentation says: 

> Due to GDELT 2.0's live updating, we do not currently make normalization files available for GDELT 2.0, but you can easily construct your own normalization files by performing a basic summation over the 15 minute update files.

It was not feasible for me to build new normalization values. Take the ones provided with a grain of salt. 

The code for the processing is accessible in the notebook. 

## Features

* **Year**

* **CountryCode**

* **CountryName**

* **SumEvents**: count of events by event code, year and country

* **TotalEvents**: total events by year and country according to the GDELT 1.0 normalization files

* **NormalizedEvents1000**: proportion of events on the normalization total (over 1000), i.e. `SumEvents/TotalEvents * 1000`

* **EventRootCode**: CAMEO event root code 

* **EventRootDescr**: CAMEO event root description

* **EventCode**: CAMEO event code

* **EventDescr**: CAMEO event description

* **GoldsteinScale**: Goldstein scale for the event, showing the theoretical impact on the stability of a country from -10 to +10

* **AvgNumMentions**:  Average of the mentions received by each event among all source documents

* **SumNumMentions**: Total of the mentions received by each event among all source documents

* **AvgAvgTone**: Average of AvgTone, which describes the average tone of all documents mentioning an event from -100 (extremely negative) to 100 (extremely positive)

  ## Feature descriptions

  The following entries from the database documentation will give more insight into the features. 

  ##### GoldsteinScale:

  > Each CAMEO event code is assigned a numeric score from -10 to +10, capturing the theoretical potential impact that type of event will have on the stability of a country. This is known as the Goldstein Scale. This field specifies the Goldstein score for each event type. NOTE: this score is based on the type of event, not the specifics of the actual event record being recorded – thus two riots, one with 10 people and one with 10,000, will both receive the same Goldstein score. This can be aggregated to various levels of time resolution to yield an approximation of the stability of a location over time

  ##### NumMentions:

  > This is the total number of mentions of this event across all source documents. Multiple references to an event within a single document also contribute to this count. This can be used as a method of assessing the “importance” of an event: the more discussion of that event, the more likely it is to be significant. The total universe of source documents and the density of events within them vary over time, so it is recommended that this field be normalized by the average or other measure of the universe of events during the time period of interest. NOTE: this field is updated over time if news articles published later discuss this event (for example, in the weeks after a major bombing there will likely be numerous news articles published mentioning the original bombing as context to new developments, while on the one-year anniversary there will likely be further coverage). At this time the daily event stream only includes new event records found each day and does not include these updates; a special “updates” stream will be released in Summer 2014 that will include these

  ##### AvgTone:

  > This is the average “tone” of all documents containing one or more mentions of this event. The score ranges from -100 (extremely negative) to +100 (extremely positive). Common values range between -10 and +10, with 0 indicating neutral. This can be used as a method of filtering the “context” of events as a subtle measure of the importance of an event and as a proxy for the “impact” of that event. For example, a riot event with a slightly negative average tone is likely to have been a minor occurrence, whereas if it had an extremely negative average tone, it suggests a far more serious occurrence. A riot with a positive score likely suggests a very minor occurrence described in the context of a more positive narrative (such as a report of an attack occurring in a discussion of improving conditions on the ground in a country and how the number of attacks per day has been greatly reduced)

  

  ## History 

  2021/04/10 - Version 1.0 build.