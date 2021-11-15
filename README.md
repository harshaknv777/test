# 1st part -Ingest data into a relational database from JSON files

For this task, a script has been prepared in Python which reads from the source and pushes data into Postgres database.
The highlevel flow of the script is as below:

* The flow of the script is divided into multiple functions
* As a first step, function "read_source_data" is called which reads from the source JSON
* This function
*   identifies the columns of data and seggregates metadata columns and actual data columns
*   the details of column names and datatypes are extracted from the source data automatically
*   prepares appropriate column lists to be used for table creation as well as data insertion
*   the final dataframe with the required columns is prepared and returned along with column lists
* Next, a database connection is established by calling a function. The database details are hard-coded in this function (in real time they can be read from a configration file). For this exercise, I have created a database "hexad" in my local postgres installation
* Using the connection established, a new table "air_quality" is created in the database using the column list created in the first function
* Passing the final dataframe and the table details the dataframe records are pushed into the database, all the statements are dynamically prepared based on the source data

# 2nd part -Answer some questions using SQL

1. Sum value of "Number of days with maximum 8-hour average ozone concentration over the National Ambient Air Quality Standard" per year

#### Query

`SELECT reportyear
     , SUM(value) AS year_sum_value
  FROM air_quality
 WHERE measurename = 'Number of days with maximum 8-hour average ozone concentration over the National Ambient Air Quality Standard'
 GROUP BY reportyear
 ORDER BY reportyear::INTEGER;`
 
 2. Year with max value of "Number of days with maximum 8-hour average ozone concentration over the National Ambient Air Quality Standard" from year 2008 and later (inclusive)

#### Query

`WITH max_value AS (SELECT MAX(value) max_value
                     FROM air_quality
                    WHERE measurename = 'Number of days with maximum 8-hour average ozone concentration over the National Ambient Air Quality Standard'
                      AND reportyear::INTEGER >= 2008
                  )
SELECT string_agg(reportyear,',') as years_with_max_value  --agg the year to cover the case of multiple years having max value
  FROM air_quality,max_value
 WHERE measurename = 'Number of days with maximum 8-hour average ozone concentration over the National Ambient Air Quality Standard'
   AND reportyear::INTEGER >= 2008
   AND value = max_value.max_value;`

3. Max value of each measurement per state

#### Query

`SELECT statename,measurename,MAX(value)
  FROM air_quality
 GROUP BY statename,measurename
 ORDER BY statename,measurename;`
 
4. Average value of "Number of person-days with PM2.5 over the National Ambient Air Quality Standard (monitor and modeled data)" per year and state in ascending order

#### Query

`SELECT reportyear,statename,AVG(value) AS avg_value
  FROM air_quality
 WHERE measurename='Number of person-days with PM2.5 over the National Ambient Air Quality Standard (monitor and modeled data)'
 GROUP BY reportyear,statename
 ORDER BY AVG(value),reportyear::INTEGER,statename;`
 
5. State with the max accumulated value of "Number of days with maximum 8-hour average ozone concentration
over the National Ambient Air Quality Standard" overall years

#### Query

`WITH acc_value AS (SELECT statename,SUM(value) accumulated_value
                     FROM air_quality
                    WHERE measurename = 'Number of days with maximum 8-hour average ozone concentration over the National Ambient Air Quality Standard'
                    GROUP BY statename
                  )
SELECT statename 
  FROM (SELECT statename,RANK()over(ORDER BY accumulated_value DESC) AS value_rank
          FROM acc_value) rnk
 WHERE value_rank = 1;`

6. Average value of "Number of person-days with maximum 8-hour average ozone concentration over the National
Ambient Air Quality Standard" in the state of Florida

#### Query

`SELECT AVG(value)
  FROM air_quality
 WHERE measurename = 'Number of days with maximum 8-hour average ozone concentration over the National Ambient Air Quality Standard'
   AND statename = 'Florida';`

7. County with min "Number of days with maximum 8-hour average ozone concentration over the National Ambient
Air Quality Standard" per state per year

#### Query

`WITH agg AS(SELECT reportyear::INTEGER,statename,countyname,SUM(value) AS sum_value
              FROM air_quality
             WHERE measurename = 'Number of days with maximum 8-hour average ozone concentration over the National Ambient Air Quality Standard'
             GROUP BY reportyear,statename,countyname
             ORDER BY reportyear::INTEGER,statename,countyname
           )
SELECT reportyear,statename,countyname
  FROM (SELECT reportyear,statename,countyname,sum_value
             , ROW_NUMBER() OVER (PARTITION BY reportyear,statename ORDER BY sum_value) AS rnk
          FROM agg
         ORDER BY reportyear,statename)a
 WHERE rnk=1
 ORDER BY reportyear,statename;`

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
* From the data lake the data would be pushed into a data warehouse with the required transformation logic
* There would be different aggregates (datamarts) built on top depending on the different granularities and business cases to be answered, one such granularity based on the use cases is "region"
* Having a datamart at a region level would make sense as the mentioned use cases expect data at this level
* These aggregates would be built combining both the sources A and B, so these aggregates would be either completely recomputed or delta would be computed after every refresh of either of the source
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
* As we are already maintaining the data at region level, the data for the required region can be pulled using the REST API (from use case 1) and perform adhoc analysis and push the final results into another repository which becomes the source for a new REST API

### Bonus challenge
#### As a customer, I want to upload proprietary data Z, and then run custom analyses on data Z and the data from data sources A and B, and access it through the REST API
* An interface to upload the proprietary data which also intelligently identifies the granularities of the uploaded data so that it can be mapped to the existing data from sources A and B
* The overall mapped data can be accessed through a REST API
* If the granularities of the uploaded data and data sources do not match, the data from one of them needs to be aggregated depending on which has lower granularity
