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

#### Weather source

As the weather data needs to be consumed using a JSON REST API, and the end point needs latitude, longitude, timestamp parameters, it would make sense to :

* Maintain a configuration of latitude and longitude values that have business significance for our warehouse
* Also, based on the business use cases, configure the minimal interval (daily/hourly/minute) for collecting the data
* The data pipeline process consumes the above configurations and accordingly consumes the weather information for the configured latitudes and longitudes at configured interval
* The data would get appended into the target in this pipeline

#### Market data source

As source data is delievered via streaming API

* It would be appropriate to use a data streaming pipeline like Spark streaming or cloud tools like data flow
* Data would be streamed into the target
* It may be appropriate to use timeseries database as the target
