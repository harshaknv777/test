# Design Exercise

## Sources:
### two data sources
•	data source A delivers data every day, with a 3-day delay

•	data source B delivers data every 14 days

•	both sources deliver data via HTTP or FTP as a direct file download

•	both sources use different formats, so data model from A cannot be used to denote data from source B
### one weather source
•	weather source C delivers either the current weather or a 7-day forecast

•	this source delivers data via a JSON REST API

•	the data can be retrieved using two endpoints - one for the current weather and one for the forecast

•	the forecast endpoint takes two parameters: latitude and longitude

•	the current endpoint takes three parameters: latitude, longitude, and optionally a timestamp
### one market data source
•	data source D delivers near real-time market data, available every 15 seconds

•	this source delivers data via a JSON API for streaming (e.g. like the Twitter streaming API)

•	each data set includes a coordinate with a latitude and longitude, where each coordinate represents a 50-mile market radius ("region").


## Approach

#### Sources A and B
* The data from the sources A and B would be ingested into a data lake
* The data pipelines for sources A and B would be batch processes that would consume the files and bulk load into the data lake
* Each source has its own pipeline independent of the other
* From the data lake the data would be pushed into a datawarehouse with the required transformation logic
* There would be different aggregates (datamarts) built on top depending on the different granularities and business cases to be answered, one such granularity based on the usecases is "region"
* Having a datamart at a region level would make sense as the mentioned use cases expect data at this level
* These aggregates would be built combining both the sources A and B, so these aggreagtes would be either completely recomputed or delta would be computed after every refresh of either of the source
* The workflow tool should be configured such a way to handle the dependency management and identification of which processes need to be triggered based on the source of data

#### Weather source

As the weather data needs to be consumed using a JSON REST API, and the end point needs latitude, longitude, timestamp parameters, it would make sense to :

* Maintain a configuration of latitude and longitude values that have business significance for our warehouse
* Also, based on the business use cases, configure the minimal interval (daily/hourly/minute) for collecting the data
* The data pipeline process consumes the above configurations and accordingly consumes the weather information for the configured latitudes and longitudes at configured interval
* The data would get appended into the target in this pipeline

#### Market data source

As source data is delievered via streaming API

* It would be appropriate to use a data streaming pipeline like Spark streaming or cloud streaming tools like GCP DataFlow
* Data would be streamed into the target
* It may be appropriate to use timeseries database as the target

## Use cases:
### Using a REST API, I want to get the average weather, average market interest, and accompanied data for a region
* From the above 3 targets created, we can create a consolidated aggregate at region level
* The REST API would expect the parameters of region, start and end times; accordingly the service would compute the average values from the above region aggregate for the requested start and end times for the requested region

### I want to get all daily weather data and daily market prices of the past 15 years
* Assuming that the average values can be provided, daily data can be extracted from the weather and market targets aggregated at day level and dumped into a target repository from where the requestor can consume the data

### I want to get data for a single region of the past 15 years, run analyses on it, and store the results for consumption by a REST API
* As we are already maintaining the data at region level, the data for the required region can be pulled using the REST API (from usecase 1) and perform adhoc analysis and push the final results into another repository which becomes the source for a new REST API

### Bonus challenge
#### As a customer, I want to upload proprietary data Z, and then run custom analyses on data Z and the data from data sources A and B, and access it through the REST API
* An interface to upload the proprietary data which also intelligently identifies the granularities of the uploaded data so that it can be mapped to the existing data from sources A and B
* The overall mapped data can be accessed through a REST API
* If the granularities of the uploaded data and data sources do not match, the data from one of them needs to be aggregated depending on which has lower granularity
