# Notifications read/write

[![CircleCI](https://circleci.com/gh/Financial-Times/notifications-rw.svg?style=svg&circle-token=0cb07dfee67b324d66c377356140330149a82421)](https://circleci.com/gh/Financial-Times/notifications-rw)

Notifications read/write is a Dropwizard application which writes ingested content to the notifications collection in MongoDB and supports query requests to supply the notifications feed.

Note on carousel pulishes: carousel publishes are ignored, unless there are no existing notifications for the original transactionId or transactions id derived from it (origialtxid_xxxx).

## Running locally
To compile, run tests and build jar

    mvn clean install

Integration tests require access to a MongoDB installation whose URL must be specified by the environment variable `MONGO_TEST_URL`. If you don't want to run integration tests, run with `-Dshort` flag.

To run locally, run:

    java -jar target/ft-notifications_rw/files/notifications-rw.jar server notifications-rw.yaml


## Building the Docker image locally
You now need to pass build arguments to authenticate against our Nexus server:

    docker build -t coco/notifications-rw --build-arg SONATYPE_USER=upp-nexus --build-arg SONATYPE_PASSWORD=AvailableInLastPass .

## Building/deploying
Check in, push, and wait three minutes: [this Jenkins job](http://ftaps116-lvpr-uk-d:8080/job/notifications-rw/) will build and package the application.

## Write requests
The standard use case for the write endpoints is for an ingester as the client making PUT and DELETE requests to the notifications-rw service. Note that both request types result in _adding_ a new notification document to the MongoDB collection.

### Content PUT
Make a PUT request to `/content/{uuid}` with Content-Type set to application/json.

The request entity may contain the  properties listed below. 
Other fields will be ignored. NB: this request body is compatible with the response for a GET to a content mapper.

| Property Name  	| Description                                                   	| Required? 	| Example Value                                                                   	|
|----------------	|---------------------------------------------------------------	|-----------	|---------------------------------------------------------------------------------	|
| `uuid`         	| Content UUID                                                  	| Y         	| `"3b7b7702-debf-11e4-b9ec-00144feab7de"`                                        	|
| `title`        	| Content Title                                                 	| Y         	| `"Ukraine looks to foreign-born ministers to kick-start reform"`                	|
| `lastModified` 	| Notification date                                             	| Y         	| `"2015-04-15T08:33:02.000Z"`                                                    	|
| `realtime`     	| `true` if the content is experienced by the user in real time 	| N         	| `true`                                                                          	|

As realtime content is presently excluded from the notifications feed, __content that has __`"realtime":true`__ is discarded by the write endpoint.__


### Content DELETE
Make a DELETE request to `/content/{uuid}` with Content-Type set to application/json.

## Read Requests

### Notification feed
Make a GET request to `/content/notifications?since=since_date`, where `since_date` is a UTC date/time in RFC3359 format (yyyy-MM-ddTHH:mm:ssZ).
This returns a JSON object with the following properties:
 
| Property Name   | Description                                         |
|-----------------|-----------------------------------------------------|
| `requestUrl`    | The (rationalised) request URL                      |
| `notifications` | A JSON array of notification items                  |
| `links`         | A JSON array of links, each with a `rel` and `href` |


To walk or poll the notification feed, clients should discover and follow the link with `rel="next"` in the response. The `href` value should be treated as an opaque URL.

#### Other query params

`offset`

The offset value is provided by the api in the  `href` value of the link with `rel="next"`. The client will pass this value back to the api when requesting for the next page of notifications. 
This param is an integer value. It is generated when multiple notifications with the same date fall over either side of a page boundary.

`page`

The page parameter is provided by the api and should be passed back by the client. The client should not attempt to manipulate this value. 
It represents a Base64 encoded value of <since_date>#<offset_value>#<cluster_id>. 
It is used to determine whether to apply a buffer to search since date because the request arrived in a different regional cluster.

`type`

To filter the notifications by "type" [Article, MediaResource, Content, ContentPackage, Audio, all]. The no param default is Article only.

e.g. `/content/notifications?since=2017-03-23T08:00:00.000Z&type=ContentPackage&type=MediaResource` returns MediaResource and ContentPackage notifications only.


### Notification history for a single item
Make a GET request to `/content/notifications/{uuid}`.
This returns a JSON array of notifications for the item (most recent first).

## Schema
### MongoDB notifications collection
The [BSON schema](./config/schema/notification.schema.bson) describes the structure of notifications stored in MongoDB.

### API notifications
The [JSON schema](./config/schema/notifications.schema.json) describes the structure of notifications in the JSON API.

## Logging
This service uses the savoirtech slf4j-json-logger library. Although this is mostly a drop-in replacement for SLF4J, events are queried by monitoring tools from Splunk and the logging pattern should be left as `%m%n` as described in [their README](https://github.com/savoirtech/slf4j-json-logger#logging-configuration).