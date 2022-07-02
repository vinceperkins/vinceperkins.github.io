---
title: Building a Flask Web App to Track Public Transit
author: Vincent Perkins
date: 2022-06-24 11:33:00 +0800
categories: [Blogging, Web Programming, Data Engineering]
tags: [GPS, latitude, longitude, flask, debezium, MySQL]
math: true
mermaid: true
image:
  path: /posts/20220624/mapbox.png
  width: 800
  height: 500
  alt: Mapbox render output
---

The Massachusetts Bay Transportation Authority offers free access to an API that allows a user  to track locations of busses within the MBTA system. The goal of this project was threefold: 
  1. To demonstrate a use case of location-based applications
     1. The MBTA API will determine the position of buses (longitude and latitude) along Route 1 and the output will visualized using a flask web application running Mapbox. 
  2. Use web-development tools (Flask and Mapbox API) to visualize the data 
     1. A MySQL database will store the information retrieved from the MBTA API. We will periodically make calls to the MBTA API, parse the JSON data returned by the API, and insert new rows into the  My SQL table.
  3. Demonstrate change data capture (CDC) between two databases (MySQL to MongoDB using Debezium).
     1. Change data capture (CDC)  is necessary to propagate changes from the master database to other databases. We will use Debezium to monitor changes to the MySQL database and propagate the changes to a MongoDB database.


## Setup Application Databases 

### Create Docker Network

We will use Docker to create the databases needed. First, we will setup a docker network **MBTANetwork** to allow MySQL and MongoDB containers to communicate.
```console
    (bt_3) C:\Users\VP1050>docker network create MBTANetwork
```

### Create MySQL Container
Next, within a .sql file we lay the groundwork for a sql database that will store the data from the MBTA API.

```sql
CREATE DATABASE IF NOT EXISTS MBTAdb;

USE MBTAdb;

DROP TABLE IF EXISTS mbta_buses;

CREATE TABLE mbta_buses (
    record_num INT AUTO_INCREMENT PRIMARY KEY,
    id varchar(255) not null,
    latitude decimal(11,8) not null,
    longitude decimal(11,8) not null,
    label varchar(255) default null,
    current_status varchar(255),
    current_stop_sequence int,
    occupancy_status varchar(255),
    speed decimal,
    updated_at varchar(255),
    direction_id int
);
```
We reference the SQL file within the dockerfile that will build our MySQL container:

```dockerfile
FROM mysql:8.0

ENV MYSQL_DATABASE=MBTAdb \
    MYSQL_ROOT_PASSWORD=JarvinDB

ADD MBTA.sql /docker-entrypoint-initdb.d

EXPOSE 3306
```
Next we run the **docker build** command and then **docker run** command for the MySQL container:

```console
C:\Users\VP1050\OneDrive\Documents> docker build -t mysqlmbtamasterimg . 

C:\Users\VP1050\OneDrive\Documents> docker run --name mysqlserver -p 3300:3306 --network MBTANetwork =d mysqlmbtamasterimg
```

If using docker desktop, we should see:
![mysqldocker](/posts/20220624/mysqldocker.png){: width="1086" height="542"}


### Create Python Client for Mysql Database <a class="anchor" id="sqlclient"></a>

```python
'''mysqldb.py'''
import urllib.request, json
import mysqldb
import sys
import mysql.connector

def insertMBTARecord(mbtaList):
    mydb = mysql.connector.connect(
    port = 3300, #local port
    user="root",
    password="------",
    host="127.0.0.1",
    db="MBTAdb",
    use_pure = True
    )

    mycursor = mydb.cursor()
    #complete the following line to add all the fields from the table
    sql = """insert into `mbta_buses` (
        id, latitude, longitude, current_status, current_stop_sequence, occupancy_status, speed, updated_at, direction_id, label)
        values (%s,%s,%s,%s,%s,%s,%s,%s,%s,%s);"""
        
        

    for mbtaDict in mbtaList:
        #complete the following line to add all the fields from the table
        val = (
            mbtaDict['id'], mbtaDict['latitude'], mbtaDict['longitude'], 
            mbtaDict['current_status'], mbtaDict['current_stop_sequence'], mbtaDict['occupancy_status'], 
            mbtaDict['speed'], mbtaDict['updated_at'], mbtaDict['direction_id'], mbtaDict['label']
            )
        mycursor.execute(sql, val)
        #mycursor.execute(sql_1, mbtaDict['bus_label'])

    mydb.commit()
```
### Create Python Client for MBTA API
We can create a python script that gets data from the MBTA API and converts the JSON expressions to a python dictionary. We will use the insertMBTARecord() function from the mysqldb.py to insert the data into the MySQL database.

```python
'''MBTAApiClient.py'''
import urllib.request, json
import mysqldb
import sys

def callMBTAApi():
    mbtaDictList = []
    mbtaUrl = 'https://api-v3.mbta.com/vehicles?filter[route]=1&include=trip'
    try:
        with urllib.request.urlopen(mbtaUrl) as url:
            data = json.loads(url.read().decode()) # Convert bytes to a str
            for bus in data['data']:
                busDict = dict()
                # complete the fields below based on the entries of your SQL table
                busDict['id'] = bus['id']
                busDict['latitude'] = bus['attributes']['latitude']
                busDict['longitude'] = bus['attributes']['longitude']
                busDict['label'] = bus['attributes']['label']
                busDict['current_status'] = bus['attributes']['current_status']
                busDict['current_stop_sequence'] = bus['attributes']['current_stop_sequence']
                busDict['occupancy_status'] = bus['attributes']['occupancy_status']
                busDict['speed'] = bus['attributes']['speed']
                busDict['updated_at'] = bus['attributes']['updated_at']
                busDict['direction_id'] = bus['attributes']['direction_id'] 
                mbtaDictList.append(busDict)
        mysqldb.insertMBTARecord(mbtaDictList) 
    except:
        print(sys.exc_info())
    return mbtaDictList  

#print(callMBTAApi())
```

### Setup the MongoDB Database

We will setup a MongoDB database to transfer data from the previously established MySQL database. We can run the following command:

```console
C:\Users\VP1050\OneDrive\Documents> docker run -p 27017:27017 --name some-mongo --network MBTANetwork -d mongo
```

Within Docker, the container will show as below: 

![mysqlmongo](/posts/20220624/mongodocker.png){: width="1086" height="542"}


## Setup Web Application Environment
### Create Python Flask Web Server to call MBTA API
The server file will import the previously created **MBTAApiClient.py** file. 
```python
'''server.py'''
from threading import Timer
from flask import Flask, render_template
import time
import json
import MBTAApiClient

# ------------------
#    BUS LOCATION  
# ------------------

# Initialize buses list by doing an API call to the MBTA database below
buses = MBTAApiClient.callMBTAApi()

def update_data():
    global buses
    buses = MBTAApiClient.callMBTAApi() #this variable captures the list of buses and their attributes

def status():
    for bus in buses:
        print(bus)

def timeloop(): #The bus list and data is output to the console and updated every 10 seconds. 
    print(f'--- ' + time.ctime() + ' ---')
    status()
    update_data()
    Timer(10, timeloop).start()
timeloop()
```
The webserver portion of the **server.py** file defines two web pages: the rendered map, and the json dump of the "buses" python dictionary. 

```python
'''server.py'''
# ----------------
#    WEB SERVER  
# ----------------

# create application instance
app = Flask(__name__)

# root route - landing page
@app.route('/')
def root():
    return render_template('index.html')

# root route - landing page
@app.route('/location')
def location():
    return (json.dumps(buses))


# start server - note the port is 3000
if __name__ == '__main__':
    app.run(port=3000)
```

### Setup the Webpage to Disaply Mapbox

Map box requires an access token to create a map. Below is the format for javascript code. See the github repository.

```js
// get and add you Mapbox access token at https://account.mapbox.com/ here
mapboxgl.accessToken = 'pk.---.---';

var map = new mapboxgl.Map({
    container: 'map',
    style: 'mapbox://styles/mapbox/streets-v11',
    center: [-71.091542,42.358862],
    zoom: 12
});

```

## Running the Web Application

We run the application by running the previously created **server.py** file. 

```console
C:\Users\VP1050\OneDrive\Documents> python3 server.py
```
The output should show the timestamp and each bus with the accompanying attributes. The insertMBTARecord() function in the [MBTAApiClient.py](#sqlclient) file will save the output to the MySQL database.
![mysqldocker](/posts/20220624/server_output.png){: width="1086" height="542"}

We can also check for the visual representation of bus location updates by going to localhost:3000 in a web browser.
![mapbox_output](/posts/20220624/mapbox.png){: width="1086" height="542"}


## Setup Change Data Capture with Debezium

### Create Debezium Container

We will create a third docker container to communicate updates from the MySQL database to the MongoDB database. 

```dockerfile
FROM maven:3.6.3-openjdk-11 AS maven_build

COPY app /tmp/

WORKDIR /tmp/
```

Build the dockerfile outlined above and run the container.

```console
C:\Users\VP1050\OneDrive\Documents> docker build -t debezium_1
C:\Users\VP1050\OneDrive\Documents> docker run -it -rm --debeziumserver --network MBTANetwork debezium_1 bash

```

### Edit Debezium Configuration files

Note the settings in the DebeziumConnectorConfig.java file within *\DebeziumCDC\app\src\main\java\mit\edu\tv\config*. The database.port and database.password should be the same as the docker container running the MySQL database:
```java
        return io.debezium.config.Configuration.create().
            .with("database.port", 3306)
            .with("database.password", "MyNewPass")
```

The java files within *\DebeziumCDC\app\src\main\java\mit\edu\tv\listener* allow the MongoDB to listen to changes within the MySQL database. 

### Confirm changes in the MongoDB database

[Download](https://www.mongodb.com/try/download/shell) the Compass client and connect to port 27017. View the changes being populated in the database. 

![Compass_Connection](/posts/20220624/compass_con.png){: width="1086" height="542"}
![Compass](/posts/20220624/compass.png){: width="1086" height="542"}


## Conclusion

The flow chart below summarizes the process of capturing data from the MBTA API and propagating that data to a MySQL database and MongoDB database. The project files can be downloaded on [Github](https://github.com/vinceperkins/mbta-transit-data-tracking-web-app). 

![Dataflow](/posts/20220624/dataflow.png){: width="1086" height="542"}

