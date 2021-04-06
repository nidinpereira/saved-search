# Saved Search Alerts
Architecture for a saved search feature that notifies a user when results for their saved searches are available.

---  
## Intro   

A service that lets a user save searches and get alerts or email notifications when results for a particular saved search are available.

Our goal is to build a service that queries data from multiple services without negatively effecting the performance of the other services, is able to scale efficiently as well. 

---

## Table of Contents
[Database](#database)  
[Architecture](#architecture)  
[Interface Design](#Interface&#32Design)  
[Scalability](#scalability)

---

### Database

For our saved search feature we will do for a simple no-sql datastore like mongodb. the only responsibility of this database will be store the query and email information along with the processed results, ready to be queued to be sent to the notification service.
the data store will have the following schema.

![alt text](https://firebasestorage.googleapis.com/v0/b/database-example-34cc0.appspot.com/o/Saved%20Search%20Database.png?alt=media&token=e22e9658-09ed-4396-a1fd-e52953e04d39)

it will contain based info about the user (userId and email), the query and the processed search results.

We will use Elastic Search as our Search Engine as it is a mature open source solution purpose built to handle search and provides high performant, low latency search results with configurable accuracy.   
Some features of Elastic Search include  
 * Distributed full text search
 * High availability
 * Multitenancy
 * Geo Search
 * Horizontal Scaling

Elastic Search is completely based on JSON and is a NoSql database. It supports distribution of data on multiple servers out of the box which allows us to scale horizontally with minimal effort.

Elastic Search provides indexing and search functionalities using RESTful APIs which makes integrating and using the service easy and seamless.

---

### Architecture

![alt text](https://firebasestorage.googleapis.com/v0/b/database-example-34cc0.appspot.com/o/Saved%20Search%20Architecture.png?alt=media&token=e3edab58-3ac7-45c7-aeaf-48f6f60a3f0e "Architecture Diagram")

In order to make our saved search feature dynamic and not require any modifications when new services are introduced into the system, we will use a centralized search service that will be dedicated to searching through data from all the different services available in the system.
using a centralized search service has many benefits over have a separate search interface in every service.

#### Search Service

For our centralized search service we will use elastic search. we will use an event driven architecture to feed data into the centralized search service.
Whenever a new searchable entity is created in any one of the services, it will push the event to a message queuing system, to which the search service will be subscribed to.

the search engine should be optimized to return query results as quickly as possible. 

when saving data into the search service, we will include all the relevant data for that service, for example for cars, we will store mileage, engine variant, price, car type, etc, and for laptops we will store cpu type, memory, model, etc.
when retrieving results we will parse and format it into a structure that only shows relevant information like name, description, product type, price and links to the corresponding product display page.

#### Saved Search Service

the saved search service will have multiple components.

A Job scheduler, this could be a linux machine running a cron service, or a lambda configured to run at specified intervals to fetch saved search results which have not been processed yet.

once the job scheduler starts it will assign all unprocessed jobs into a batch processing queue. Each node of the batch processing cluster will query the central search service and save the results into the saved-search service datastore as well as send it to a topic in the message queuing system that will be subscribed to by the notification service, which will be responsible for sending the emails to the customer.

---

### Interface Design

Our saved search api will be accessible through the api gateway to the front end using two RESTful APIs.
we will use a message queueing system for passing data to and from other services.


#### Saved Search

##### list of saved searches for a user

Method: GET  
endpoint: /saved-searches
request headers:  
```
Authorization: {{refreshToken}}
```
response:
```
{
    success: true,
    "data": [
        {
        "id": "jPQapzBULU87aD66GTTz4g65wIEv8",
        "user_id": "dJYapzBUO7PRD66GTTz4g65wICx1",
        "user_email": "nidinpereira@gmail.com",
        "query":"BMW Z4 10,000KM",
        "created_at": "2021-04-04T02:23:56.000Z",
        "updated_at": "2021-04-05T09:44:78.000Z",
        "is_request_completed": false,
        "result_count": 168,
        "results": [{
            "id": "qCQapzBULU87aD66GTTz4g65wICr2',
            "name": "BMW Z4",
            "description": "BMW Z4 2018 white 6,000km driven",
            "price": "20,000 AED",
            "productType: "Car",
            "url": "https://dubai.dubbizle.com/cars/qCQapzBULU87aD66GTTz4g65wICr2",
            "deep_link: "dbzl://cars/qCQapzBULU87aD66GTTz4g65wICr2"
        }]
        }, {
        "id": "jPQapzBULU87aD66GTTz4g65wIEv9",
        "user_id": "dJYapzBUO7PRD66GTTz4g65wICx1",
        "user_email": "nidinpereira@gmail.com",
        "query":"Lenovo Thinkpad 14",
        "created_at": "2021-03-08T21:54:11.000Z",
        "updated_at": "2021-04-05T09:44:78.000Z",
        "is_request_completed": false
        "result_count": 223,
        "results": [{
            "id": "erDapzBULU87aD66GTTz4g65wICd5',
            "name": "Lenovo",
            "description": "Lenovo Thinkpad Carbon 14inch in brand new condition",
            "price": "900 AED",
            "productType: "Laptop",
            "url": "https://dubai.dubbizle.com/laptops/erDapzBULU87aD66GTTz4g65wICd5",
            "deep_link": "dbzl://laptops/erDapzBULU87aD66GTTz4g65wICd5"
        }]
        },
    ]
}
```

##### Save Search

Method: POST  
endpoint: /saved-searches
request headers:
```
Authorization: {{refreshToken}}
```

request body:
```
{
    "user_id": "dJYapzBUO7PRD66GTTz4g65wICx1",
    "user_email": "nidinpereira@gmail.com",
    "query": "Macbook Pro 2019"
}
```

response:
```
{
    "success": "true",
    "message": "Query saved successfully"
}
```

#### Centralized Search

##### Search Results

Method: GET  
endpoint: /search?query=BMW


response:
```
{
    "success": "true",
    "data":[{
            "id": "qCQapzBULU87aD66GTTz4g65wICr2',
            "name": "BMW Z4",
            "description": "BMW Z4 2018 white 6,000km driven",
            "price": "20,000 AED",
            "productType: "Car",
            "url": "https://dubai.dubbizle.com/cars/qCQapzBULU87aD66GTTz4g65wICr2",
            "deep_link: "dbzl://cars/qCQapzBULU87aD66GTTz4g65wICr2"
        }]
}
```

---

### Scalability
Since the system will be handling hundreds of thousands of requests, we need to consider horizontal as well as vertical scaling.

#### Message Queues
Since directly sending data to the search service and email service will cause sudden spikes and denial of service, we will use a message queing system to queue the tasks to be performed, and the consumers can process the request based on availability. 
This will also enable us to have multiple services subscribed to the same topic.
 To optimize the high volume of data that needs to be handled by the message brokers we need to do the following:  
  * Never directly subscribe to a topic. all messages are sent through the queue.

### Centralized Search
Querying all the different services at regular intervals for thousands of saved requests can cause spikes and downtimes in the indiviual services. In order to minimize the load on the indiviual services, instead of a pull approach, we will use an event based architecture, where when any service saves an entry it will send the data to the search service. 

The search service can be optimized by
 * query using bulk requests
 * increase indexing time
 * choose a shard size between 10GB and 65GB and less than 20 shards per GB of memory heap
 * have shards accross multiple clusters to get quicker results
 * delete unused indices

### Search Service

Since the search service can have hundreds of thousands of saved searches, the search service will consist of a job scheduler, that periodically runs checks for saved searches and assigns all the saved searches that need to be processed into a queue.
the queue will be processed by a load balancer that will dynamically start n number of containers or lambdas as required by the number of jobs available in the queue.

---
