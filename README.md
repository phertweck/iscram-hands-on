# Management of Sensor Data with Open Standards
Lab session for the ISCRAM tool talk: 'Management of Sensor Data with Open Standards'. In this session you'll learn how to use the [FROST-Server](https://github.com/FraunhoferIOSB/FROST-Server) as implementation of the SensorThings-API standard

## Preparation
There are several possibilities to run/access a FROST-Server for the lab session:
* Use docker to run a local instance (If you do not have Docker see "Installation using Docker" below)
* Use our online available test instance: [https://iscram-frost.docker01.ilt-dmz.iosb.fraunhofer.de/](https://iscram-frost.docker01.ilt-dmz.iosb.fraunhofer.de/) (MQTT available on port 30020)

If possilbe, we suggest to use the provided docker image.

Further possibilities (only recommended if you're familiar with that technology):
* Download the precompiled war file and run Tomcat and PostgreSQL with PostGIS locally. Instructions [here](https://github.com/FraunhoferIOSB/FROST-Server/wiki/Installation).
* If you have access to a kubernetes cluster you can use our [helm chart](https://fraunhoferiosb.github.io/helm-charts/)

### Installation using Docker
1) Install docker on your machine:
    1) For Linux: [Docker CE](https://docs.docker.com/install/linux/docker-ce/ubuntu/)
    1) For Windows with Hyper-V: [Docker Desktop for Windows](https://docs.docker.com/docker-for-windows/install/)
    2) For Windows without Hyper-v: [Docker Toolbox](https://docs.docker.com/toolbox/overview/)
    3) For Mac: [Docker Desktop for Mac](https://docs.docker.com/docker-for-mac/install/)
2) On Linux you need to install [docker-compose](https://docs.docker.com/compose/install/). Automatically available on Windows and Mac.
3) Download the docker-compose file from this repository.
4) Start the server with `docker-compose up`
5) Browse to: ``http://localhost:8080/FROST-Server/v1.0``

## Exercises

### 1. Getting an overview
The aim of this first exercise is to explore the available entities.
1) Browse to: ``http://localhost:8080/FROST-Server/v1.0`` (or to the online available test instance). You will see one data collection for each entity of the data model. If available we recommend to use Firefox. For other browsers you might need to install an add-on to visualize JSON data.
2) Add sample data to the server:
    - On **Linux** you can use `curl` to do the requests: `curl -X POST -H "Content-Type: application/json" -d @demoData/demoEntities1.json http://localhost:8080/FROST-Server/v1.0/Things`
    - For **Windows** we recommend to use [Postman](https://www.getpostman.com/):
        - Create a POST-request to `http://localhost:8080/FROST-Server/v1.0/Things`
        - Set the `Content-Type`-header to `application/json`
        - Use the contet of `demoEntities.json` file as body for the request
3) Explore the newly created data and the relations, e.g. starting with `http://localhost:8080/FROST-Server/v1.0/Things` and follow the provided links. E.g.
    - Getting all things: `v1.0/Things`
    - Getting a specific thing: `v1.0/Things(1)`
    - Getting the related entities (all observations for the first datastream of the first thing): `v1.0/Things(1)/Datastreams(1)/Observations`
4) Explore the URL parameters
    - Getting only 4 observations and the total count: `v1.0/Observations?$top=4&$count=true`
    - Getting only the description and id for all things: `v1.0/Things?$select=@iot.id,description`
    - Getting all Observations sorted by phenomenonTime, newest first: `v1.0/Observations?$orderby=phenomenonTime desc`



### 2. Exploring the filter capabilitites
After you've learned how the entities and their relations can be access, the aim of the second exercise is to explore the OData filtering capabilities. The examples are taken from the offical [documentation](
https://github.com/FraunhoferIOSB/FROST-Server/wiki/Example-queries).

All these examples are not urlencoded, for readability. If you use these examples, depending on your browser (for Firefox and Chrome it's done automatically), don't forget to urlencode.

Additional information about the filter operations and encoding can be found at the end.

1. **Greater than and smaller than**: Retrieve all observations with result greater than 40 and smaller than 41:
    - ```v1.0/Observations?$filter=result ge 40 and result le 41```
    - What's the query for getting the Observations with `2<=id<5`?

1. **Overlapping time frames**: The phenomenonTime and result of the first 1000 observations, ordered by phenomenonTime that overlap with the time frame from 2019-03-14 10:04:00 UTC to 2019-03-14 10:04:00 UTC (3 hours):
    - ```
      v1.0/Observations
      ?$orderby=phenomenonTime asc
      &$top=1000
      &$select=phenomenonTime, result
      &$filter=overlaps(phenomenonTime, 2019-03-14T10:01:00Z/2019-03-14T10:04:00Z)
      ```

1. **The previous day**: The observations of a Datastream, for the last 100 days (P100D = 100 days, P1D = 1 day):
    - ```
      v1.0/Datastreams(1)/Observations?$filter=phenomenonTime gt now() sub duration'P100D'
      ```

1. **The last x days**: The observations of a Datastream, for the last days, where the number of days is configured in properties/days of the Datastream:
    - ```
      v1.0/Datastreams(1)/Observations?$filter=phenomenonTime gt now() sub duration'P1D' mul Datastream/properties/days
      ```

1. **Odd or even**: All observations with an even result
    - ```
      v1.0/Observations?$filter=result mod 2 eq 0
      ```

1. **Filters on related Entities**:
    - Datastreams that have data for the ObservedProperty with id 1
        - ```
          v1.0/Datastreams?$filter=ObservedProperty/@iot.id eq 1
            ```
    - ObservedProperties that are measured at the same station as the ObservedProperty with name Temperature
        - ```
          v1.0/ObservedProperties?$filter=Datastream/Thing/Datastreams/ObservedProperty/name eq 'Temperature'
          ```
1. **Filtering on JSON properties**: Get all things this tyle Cozy
    - ```v1.0/Things?$filter=properties/style eq 'Cozy'```

1. **Threshold detection**: Filtering Observations where result is greater than a threshold stored in the properties of their Thing. The "add 0" is to indicate we want to use a numeric comparison. Since both Observation/result and Thing/properties can be anything, the server would otherwise default to a (safe) string-comparison.
    - ```
      v1.0/Observations?$filter=result gt Datastream/Things/properties/max add 0
        ```
1. **Ordering by function**: Functions work for Ordering
    - ```
      v1.0/Datastreams?$orderby=length(name) desc
      ```

1. **Give me EVERYTHING!**: All things, with their current Locations and Datastreams, and for those Datastreams the ObservedProperty and the last Observation:
    - ```
      v1.0/Things
        ?$select=name,description,@iot.id
        &$expand=
          Locations
            ($select=name,description,location,@iot.id)
          ,Datastreams
            ($select=name,description,@iot.id
            ;$expand=
              Sensor
                ($select=name,description,@iot.id)
              ,ObservedProperty
                ($select=name,description,@iot.id)
              ,Observations
                ($select=result,phenomenonTime,@iot.id
                ;$orderby=phenomenonTime%20desc
                ;$top=2
                )
            )
      ```

### 3. Using MQTT
In the last exercises we queried the FROST-Server. Depending on the use-case, it might be important to get informed, if new data is available on the server. To get notified, you can subscribe the to MQTT server.
- On Linux: `mosquitto_sub -t 'v1.0/Observations'`
- On Windows: You need an MQTT client like e.g. [MQTT.fx](https://mqttfx.jensd.de/). With it connect to `local mosquitto` and then subscribe to topic `v1.0/Observations` (do not use the complete URL)

The MQTT topics, you can subscribe to are the possible URL-paths (e.g. `v1.0/Things(1)/Datastreams(1)/Observations`). Filters are not allowed. A message will be sent as soons there is a change in the collection.

Leave the MQTT subscription to `v1.0/Observations` open, while doing exercise 4.

### 4. Creating, changing and deleting entities
Entities can be created by performing a POST request to the collection. E.g. to create a new Thing the following command can be used for Linux (for Windows use Postman instead):
```
curl -i -X POST \
-H 'Content-Type: text/json; charset=utf-8' \
-d \
'
{
"name" : "Office",
"description" : "My Work Room",
"properties" : {
  "style" : "Business",
  "balcony" : false
},
"Locations" : [
  {
    "@iot.id" : 1
  }
]
}
' \
http://localhost:8080/FROST-Server/v1.0/Things

```
You can see that the location is a reference to an existing location with id 1. It's also possible to create a new location:

```
curl -i -X POST \
-H 'Content-Type: text/json; charset=utf-8' \
-d \
'
{
"name" : "Office",
"description" : "My Work Room",
"properties" : {
  "style" : "Business",
  "balcony" : false
},
"Locations" : [
    {
      "name" : "My Office",
      "description" : "The office room of Fraunhoferstr. 1",
      "encodingType" : "application/vnd.geo+json",
      "location" : {
        "type":"Point",
        "coordinates":[8.425548,49.015196]
      }
    }
]
}
' \
http://localhost:8080/FROST-Server/v1.0/Things
```
New observations can be created accordingly:

```
curl -i -X POST \
-H 'Content-Type: text/json; charset=utf-8' \
-d \
'
{
  "result" : 123,
  "Datastream" : {
    "@iot.id" : 1
  }
}
' \
http://localhost:8080/FROST-Server/v1.0/Observations
```
or directly to the datastream:
```
curl -i -X POST \
-H 'Content-Type: text/json; charset=utf-8' \
-d \
'
{
  "result" : 123
}
' \
http://localhost:8080/FROST-Server/v1.0/Datastreams(1)/Observations
```

If your MQTT subscription was still open, you should have received a message like:
```
{
  "phenomenonTime" : "2019-05-15T10:39:57.724Z",
  "resultTime" : null,
  "result" : 123,
  "Datastream@iot.navigationLink" : "http://localhost:8080/FROST-Server/v1.0/Observations(19)/Datastream",
  "FeatureOfInterest@iot.navigationLink" : "http://localhost:8080/FROST-Server/v1.0/Observations(19)/FeatureOfInterest",
  "@iot.id" : 19,
  "@iot.selfLink" : "http://localhost:8080/FROST-Server/v1.0/Observations(19)"
}
```

To change an entity, you need to do a PATCH (only updated specified fields) or PUT (override all fields, drop not existing ones) request:

```
curl -i -X PATCH \
-H 'Content-Type: text/json; charset=utf-8' \
-d \
'
{
  "description" : "Hi ISCRAM!"
}
' \
http://localhost:8080/FROST-Server/v1.0/Things(1)
```

To delete an entity a HTTP request needs to be done. E.g. to remove the fist thing (together with all objects depending on the thing: Datastreams, Observations!):

```
curl -i -X DELETE http://localhost:8080/FROST-Server/v1.0/Things(1)
```

## References
- [FROST-Server](https://github.com/FraunhoferIOSB/FROST-Server)
- [OGC SensorThings API Part 1: Sensing ](http://docs.opengeospatial.org/is/15-078r6/15-078r6.html)

## Filter operations
| Category         | Filter function      | Symbol |
|------------------|----------------------|--------|
| Comparison       | gt                   | >      |
| Comparison       | ge                   | >=     |
| Comparison       | eq                   | =      |
| Comparison       | le                   | <=     |
| Comparison       | lt                   | <      |
| Comparison       | ne                   | !=     |
| Logical          | and                  |        |
| Logical          | or                   |        |
| Logical          | not                  |        |
| Mathematical     | add                  |        |
| Mathematical     | sub                  |        |
| Mathematical     | mul                  |        |
| Mathematical     | div                  |        |
| Mathematical     | mod                  |        |
| String functions | substringof (p0, p1) |        |
| String functions | endswith (p0, p1)    |        |
| String functions | startswith (p0, p1)  |        |
| String functions | substring(p0, p1)    |        |
| String functions | indexof (p0, p1)     |        |
| String functions | length(p0)           |        |
| String functions | tolower (p0)         |        |
| String functions | toupper (p0)         |        |
| String functions | trim(p0)             |        |
| String functions | concat (p0, p1)      |        |
| Mathematical     | round(n1)            |        |
| Mathematical     | floor(n1)            |        |
| Mathematical     | ceiling(n1)          |        |
| Geospatial       | geo.intersects (g1, g2) |     |
| Geospatial       | geo.length (l1)         |     |
| Geospatial       | geo.distance (g1, g2)   |     |
| Geospatial       | st_equals (g1, g2)      |     |
| Geospatial       | st_disjoint (g1, g2)    |     |
| Geospatial       | st_touches (g1, g2)     |     |
| Geospatial       | st_within (g1, g2)      |     |
| Geospatial       | st_overlaps (g1, g2)    |     |
| Geospatial       | st_crosses (g1, g2)     |     |
| Geospatial       | st_intersects (g1, g2)  |     |
| Geospatial       | st_contains (g1, g2)    |     |
| Geospatial       | st_relate (g1, g2)      |     |
| Date and Time    | now()                   |     |
| Date and Time    | mindatetime ()          |     |
| Date and Time    | maxdatetime ()          |     |
| Date and Time    | date(t1)                |     |
| Date and Time    | time(t1)                |     |
| Date and Time    | year(t1)                |     |
| Date and Time    | month(t1)               |     |
| Date and Time    | day(t1)                 |     |
| Date and Time    | hour(t1)                |     |
| Date and Time    | minute(t1)              |     |
| Date and Time    | second(t1)              |     |
| Date and Time    | fractionalseconds (t1)  |     |
| Date and Time    | totaloffsetminutes (t1) |     |

## Encoding
- Use single quotes for Strings
- No quotation for DateTimes (ISO 8601): `2019-05-20T10:20:42+00:00`

## Authors
- Philipp Hertweck [philipp.hertweck@iosb.fraunhofer.de](mailto:philipp.hertweck@iosb.fraunhofer.de)
- Hylke van der Schaaf [hylke.vanderschaaf@iosb.fraunhofer.de](mailto:hylke.vanderschaaf@iosb.fraunhofer.de)
- Jan Blume [jan-wilhelm.blume@iosb.fraunhofer.de](mailto:jan-wilhelm.blume@iosb.fraunhofer.de)
- Tobias Hellmund [tobias.hellmund@iosb.fraunhofer.de](tobias.hellmund@iosb.fraunhofer.de)
