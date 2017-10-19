---
layout: post
title: Honeycomb
---

What is it ?
------------

The Honeycomb purpose is to store and [serve your collected data](#retrieve-your-data).


## Deploy your own honeycomb

The Honeycomb server can be deployed anywhere, 
given it is [linked to your dashboard](../dashboard#add-a-custom-backend) and provided a _mongodb_ database.

### Download a Honeycomb

The honeycomb is a [scala Play!](https://playframework.com/) server available as a docker image on the [docker hub](https://hub.docker.com/r/apisense/honeycomb/).

### Configuration

The Honeycomb configuration can be changed thanks to environment variables:

- Mandatory configuration:
  - __HONEYCOMB_API_KEY__: Api key generated by the _Dashboard_.
  - __APP_SECRET__: [Play! specific configuration](https://www.playframework.com/documentation/2.5.x/ApplicationSecret) used for cryptographic operations.
- Customize the Honeycomb behaviour:
  - __HONEYCOMB_MAX_DISK_BUFFER__: Maximum size for the uploaded media files (default: 100M).
  - __HONEYCOMB_MAX_MEMORY_BUFFER__: Maximum size for the uploaded json files (default: 500K).
  - __HONEYCOMB_RESPONSE_LIMIT__: Maximum number of element contained in a data page (default: 10, unlimited if negative).
  - __HONEYCOMB_DATA_QUOTA__: Default quota set to each new user by default (default: 500M, unlimited if 0).
- Configure MongoDB:
  - __MONGODB_PORT_27017_TCP_ADDR__: Mongodb server host. (Automatically filled if using a linked docker container).
  - __MONGODB_PORT_27017_TCP_PORT__: Mongodb server port. (Automatically filled if using a linked docker container).
  - __HONEYCOMB_MONGODB_USERNAME__: [Mongodb user](https://docs.mongodb.com/manual/tutorial/enable-authentication/) to be used by the Honeycomb, not used if left empty.
  - __HONEYCOMB_MONGODB_PASSWORD__: Password of the previous user, not used if left empty.
  - __HONEYCOMB_MONGODB_AUTH_SOURCE__: Database containing the users (default: 'admin').
  - __HONEYCOMB_MONGODB_AUTH_MODE__: Authentication mode used by your Mongodb instance (default: 'SCRAM-SHA-1', [available values](https://docs.mongodb.com/manual/core/authentication/#authentication-mechanisms)).
- Use HTTPS, [as documented](https://www.playframework.com/documentation/2.5.x/ConfiguringHttps):
  - __HONEYCOMB_SSL_KEYSTORE_PATH__: Path to the Java keyStore.
  - __HONEYCOMB_SSL_KEYSTORE_TYPE__: Type of the keystore (default: 'JKS').
  - __HONEYCOMB_SSL_KEYSTORE_PASSWORD__: Password for the java keyStore.
  - __HONEYCOMB_SSL_KEYSTORE_ALGORITHM__: Algorithm used by the keyStore.

### Use more storage

You can set complementary storage service for your crops with the following configurations:

- Use InfluxDB:
  - __HONEYCOMB_INFLUXDB_ENABLE__: Enable usage of InfluxDB service. (default: `false`)
  - __INFLUXDB_PORT_8086_TCP_ADDR__:  InfluxDB server host (automatically filled if using a linked docker container).
  - __INFLUXDB_PORT_8086_TCP_PORT__: InfluxDB server port (automatically filled if using a linked docker container).
  - __HONEYCOMB_INFLUXDB_USERNAME__: InfluxBB username.
  - __HONEYCOMB_INFLUXDB_PASSWORD__: InfluxDB password.
- Use Neo4j (This service is actually embedded in Honeycomb and does not need external database):
  - __HONEYCOMB_NEO4J_ENABLE__: Enable usage of Neo4j service. (default: `false`)
  - __HONEYCOMB_NEO4J_PATH__: Data storage location (default: `.neo4j/dbs`).
  - __HONEYCOMB_NEO4J_PAGE_CACHE_MEMORY__: Cache memory settings (default: `256Ko`), see the [perfomance documentation](https://neo4j.com/docs/operations-manual/current/performance/)
  - __HONEYCOMB_NEO4J_LOGS__: Logs storage location (default: `.neo4j/logs`).

### Docker compose example

The following sample will deploy a Honeycomb server listening on port 80 with a linked _mongodb_ database:

    honeycomb:
      image: apisense/honeycomb:latest 
      ports:
        - "80:9000"
        - "443:9443" # Enables to contact the Honeycomb via HTTPS
      volumes:
        - "/data/honeycomb/logs:/opt/docker/logs"
      links:
        - database:mongodb
        # - influx:influx # Uncomment to add an influx database
      environment:
        - HONEYCOMB_API_KEY=myKey
        # - HONEYCOMB_INFLUXDB_ENABLE = true # Uncomment to activate
      restart: unless-stopped

    database:
      image: mongo
      # command: "mongod --auth" # Uncomment this line to activate authentication
      volumes:
        - "/data/honeycomb/mongo:/data/db"
      restart: unless-stopped

    # You can add an influxDB if you need it.
    #influx:
    #  image: image: influxdb:1.3-alpine
    #  volumes:
    #    - "/data/honeycomb/influx:/var/lib/influxdb"


You can try it out by saving this content to a `docker-compose.yml` file and executing the command [docker-compose up](https://docs.docker.com/compose/reference/up/)

## Retrieve your data

### Access private data

If your crop data are under restricted access, you will have to provide a valid access key to the honeycomb.

To do so, add the following header to your request: `Authorization: accessKey $myKey`

### Get everything

To retrieve the data from your collect, you can click on the button `Download data` from the dashboard.
But you also have access to an API to get the raw json using `/api/v1/crop/$cropIdentifier/data`.

The Json you will retrieve is built as follow:

    [
      {
        "metadata": {
          "timestamp": "2015-10-06T16:00:43+02:00",
          "device": "Android"
        },
        "header": {
          "environmentalInfo": "from sync(...) method, 1 per upload"
        },
        "body": [
          { "yourTrace": "from save(...) method, 1 per trace" },
          { "yourTrace": "another saved trace" },
        ]
      }, {
        ...
      }
    ]

<div class="alert alert-warning" role="alert">
    Note that on the data API is paginated to limit the size of each response,
    you can change the requested page as follow `/api/v1/crop/$cropIdentifier/data?page=x` (The pagination starts at page 0).
</div>

### Filter uploaded data

If you want to insert some specifics, calculated values,
you can add a pre-upload filter, applied on each uploaded data (see the section above for the syntax).

    // This filter will process every uploaded data
    rest.setPreUploadTreatment(function(data) { // data will be the uploaded JSON
       var myUploadedDataArray = data.body;
       var myMetadata = data.metadata;
       myMetadata.count = myUploadedDataArray.length;
       return data; // It is recommended to send back the same json structure as the input.
    });

### Filter output

If you don't want to download every collected traces from the server, you can define custom routes to filter the data beforehand.

The creation of these routes can be done in the `Filters` menu, from the collect dashboard.
In this menu, you will find a script editor, in which you can add routes with the following (Javascript) syntax:

    // This filter will only return the metadata of each collected trace.
    // It can be accessed from the route /api/v1/crop/$cropIdentifier/data/meta
    rest.prepareFilter("meta", function(data){ // data will be your raw data json array
      var result = [];
      for each (var ele in data) {
        result.push(ele.metadata);
      }
      return result; // This will be parsed as a Json.
    });


After saving your script, you can access the result from the route `/api/v1/crop/$cropIdentifier/data/meta`.

## Retrieve your media

Some stings will record media files (like pictures, sound, or video).
For obvious reasons, those media will not directly be saved in the data JSON,
but an identifier will be set instead.


### Metadata format:

You can retrieve medatada for every media uploaded on a crop from the route `/api/v1/crop/$cropIdentifier/media`.
Every metadata object will contain the following elements:

    {
      "cropIdentifier": "Zz86D0v6O1CWGldoD5Zg",
      "identifier": "64C8356C-5C91-48E4-AE5D-E88F3AF047A8",
      "size": 142222,
      "upload": 1455618401229,
      "contentType": "image/jpeg",
      "url": "/api/v1/crop/Zz86D0v6O1CWGldoD5Zg/media/64C8356C-5C91-48E4-AE5D-E88F3AF047A8"
    }

With:

- `identifier`: Media identifier set in your sting result.
- `size`: Media size in bytes.
- `upload`: Upload timestamp.
- `contentType`: Mime type found by the honeycomb (may be null if the type isn't recognized).
- `url`: Media location on the honeycomb.

### Retrieve the media file

A raw media file can be retrieved using the route `/api/v1/crop/$cropIdentifier/media/$mediaID`,
where mediaID is the media identifier set in the sting result.

The file will be sent via an `application/octet-stream` content type,
whichever the file type may be.