

# Architecture

This application builds on the components created in
[previous tutorials](https://github.com/FIWARE/tutorials.Subscriptions/). It will make use of two FIWARE components -
the [Orion Context Broker](https://fiware-orion.readthedocs.io/en/latest/) and the
[IoT Agent for UltraLight 2.0](https://fiware-iotagent-ul.readthedocs.io/en/latest/). Usage of the Orion Context Broker
is sufficient for an application to qualify as _“Powered by FIWARE”_. Both the Orion Context Broker and the IoT Agent
rely on open source [MongoDB](https://www.mongodb.com/) technology to keep persistence of the information they hold. We
will also be using the dummy IoT devices created in the
[previous tutorial](https://github.com/FIWARE/tutorials.IoT-Sensors/)

Therefore the overall architecture will consist of the following elements:

-   The FIWARE [Orion Context Broker](https://fiware-orion.readthedocs.io/en/latest/) which will receive requests using
    [NGSI-v2](https://fiware.github.io/specifications/OpenAPI/ngsiv2)
-   The FIWARE [IoT Agent for UltraLight 2.0](https://fiware-iotagent-ul.readthedocs.io/en/latest/) which will receive
    southbound requests using [NGSI-v2](https://fiware.github.io/specifications/OpenAPI/ngsiv2) and convert them to
    [UltraLight 2.0](https://fiware-iotagent-ul.readthedocs.io/en/latest/usermanual/index.html#user-programmers-manual)
    commands for the devices
-   The underlying [MongoDB](https://www.mongodb.com/) database :
    -   Used by the **Orion Context Broker** to hold context data information such as data entities, subscriptions and
        registrations
    -   Used by the **IoT Agent** to hold device information such as device URLs and Keys
-   The **Context Provider NGSI** proxy is not used in this tutorial. It does the following:
    -   receive requests using [NGSI-v2](https://fiware.github.io/specifications/OpenAPI/ngsiv2)
    -   makes requests to publicly available data sources using their own APIs in a proprietary format
    -   returns context data back to the Orion Context Broker in
        [NGSI-v2](https://fiware.github.io/specifications/OpenAPI/ngsiv2) format.
-   The **Stock Management Frontend** is not used in this tutorial will it does the following:
    -   Display store information
    -   Show which products can be bought at each store
    -   Allow users to "buy" products and reduce the stock count.
-   A webserver acting as set of [dummy IoT devices](https://github.com/FIWARE/tutorials.IoT-Sensors/tree/NGSI-v2) using
    the
    [UltraLight 2.0](https://fiware-iotagent-ul.readthedocs.io/en/latest/usermanual/index.html#user-programmers-manual)
    protocol running over HTTP.

Since all interactions between the elements are initiated by HTTP requests, the entities can be containerized and run
from exposed ports.

![](https://fiware.github.io/tutorials.IoT-Agent/img/architecture.png)

The necessary configuration information for wiring up the IoT devices and the IoT Agent can be seen in the services
section of the associated `docker-compose.yml` file:

## Dummy IoT Devices Configuration

```yaml
tutorial:
    image: quay.io/fiware/tutorials.context-provider
    hostname: iot-sensors
    container_name: fiware-tutorial
    networks:
        - default
    expose:
        - '3000'
        - '3001'
    ports:
        - '3000:3000'
        - '3001:3001'
    environment:
        - 'DEBUG=tutorial:*'
        - 'PORT=3000'
        - 'IOTA_HTTP_HOST=iot-agent'
        - 'IOTA_HTTP_PORT=7896'
        - 'DUMMY_DEVICES_PORT=3001'
        - 'DUMMY_DEVICES_API_KEY=4jggokgpepnvsb2uv4s40d59ov'
        - 'DUMMY_DEVICES_TRANSPORT=HTTP'
```

The `tutorial` container is listening on two ports:

-   Port `3000` is exposed so we can see the web page displaying the Dummy IoT devices.
-   Port `3001` is exposed purely for tutorial access - so that cUrl or Postman can make UltraLight commands without
    being part of the same network.

The `tutorial` container is driven by environment variables as shown:

| Key                     | Value                        | Description                                                                                                                               |
| ----------------------- | ---------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| DEBUG                   | `tutorial:*`                 | Debug flag used for logging                                                                                                               |
| WEB_APP_PORT            | `3000`                       | Port used by web-app which displays the dummy device data                                                                                 |
| IOTA_HTTP_HOST          | `iot-agent`                  | The hostname of the IoT Agent for UltraLight 2.0 - see below                                                                              |
| IOTA_HTTP_PORT          | `7896`                       | The port that the IoT Agent for UltraLight 2.0 will be listening on. `7896` is a common default for UltraLight over HTTP                  |
| DUMMY_DEVICES_PORT      | `3001`                       | Port used by the dummy IoT devices to receive commands                                                                                    |
| DUMMY_DEVICES_API_KEY   | `4jggokgpepnvsb2uv4s40d59ov` | Random security key used for UltraLight interactions - used to ensure the integrity of interactions between the devices and the IoT Agent |
| DUMMY_DEVICES_TRANSPORT | `HTTP`                       | The transport protocol used by the dummy IoT devices                                                                                      |

The other `tutorial` container configuration values described in the YAML file are not used in this tutorial.

## IoT Agent for UltraLight 2.0 Configuration

The [IoT Agent for UltraLight 2.0](https://fiware-iotagent-ul.readthedocs.io/en/latest/) can be instantiated within a
Docker container. An official Docker image is available from [Docker Hub](https://hub.docker.com/r/fiware/iotagent-ul/)
tagged `fiware/iotagent-ul`. The necessary configuration can be seen below:

```yaml
iot-agent:
    image: quay.io/fiware/iotagent-ul:latest
    hostname: iot-agent
    container_name: fiware-iot-agent
    depends_on:
        - mongo-db
    networks:
        - default
    expose:
        - '4041'
        - '7896'
    ports:
        - '4041:4041'
        - '7896:7896'
    environment:
        - IOTA_CB_HOST=orion
        - IOTA_CB_PORT=1026
        - IOTA_NORTH_PORT=4041
        - IOTA_REGISTRY_TYPE=mongodb
        - IOTA_LOG_LEVEL=DEBUG
        - IOTA_TIMESTAMP=true
        - IOTA_CB_NGSI_VERSION=v2
        - IOTA_AUTOCAST=true
        - IOTA_MONGO_HOST=mongo-db
        - IOTA_MONGO_PORT=27017
        - IOTA_MONGO_DB=iotagentul
        - IOTA_HTTP_PORT=7896
        - IOTA_PROVIDER_URL=http://iot-agent:4041
```

The `iot-agent` container relies on the precence of the Orion Context Broker and uses a MongoDB database to hold device
information such as device URLs and Keys. The container is listening on two ports:

-   Port `7896` is exposed to receive Ultralight measurements over HTTP from the Dummy IoT devices
-   Port `4041` is exposed purely for tutorial access - so that cUrl or Postman can make provisioning commands without
    being part of the same network.

The `iot-agent` container is driven by environment variables as shown:

| Key                  | Value                   | Description                                                                                                                                           |
| -------------------- | ----------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| IOTA_CB_HOST         | `orion`                 | Hostname of the context broker to update context                                                                                                      |
| IOTA_CB_PORT         | `1026`                  | Port that context broker listens on to update context                                                                                                 |
| IOTA_NORTH_PORT      | `4041`                  | Port used for Configuring the IoT Agent and receiving context updates from the context broker                                                         |
| IOTA_REGISTRY_TYPE   | `mongodb`               | Whether to hold IoT device info in memory or in a database                                                                                            |
| IOTA_LOG_LEVEL       | `DEBUG`                 | The log level of the IoT Agent                                                                                                                        |
| IOTA_TIMESTAMP       | `true`                  | Whether to supply timestamp information with each measurement received from attached devices                                                          |
| IOTA_CB_NGSI_VERSION | `v2`                    | Whether to supply use NGSI v2 when sending updates for active attributes                                                                              |
| IOTA_AUTOCAST        | `true`                  | Ensure Ultralight number values are read as numbers not strings                                                                                       |
| IOTA_MONGO_HOST      | `context-db`            | The hostname of mongoDB - used for holding device information                                                                                         |
| IOTA_MONGO_PORT      | `27017`                 | The port mongoDB is listening on                                                                                                                      |
| IOTA_MONGO_DB        | `iotagentul`            | The name of the database used in mongoDB                                                                                                              |
| IOTA_HTTP_PORT       | `7896`                  | The port where the IoT Agent listens for IoT device traffic over HTTP                                                                                 |
| IOTA_PROVIDER_URL    | `http://iot-agent:4041` | URL passed to the Context Broker when commands are registered, used as a forwarding URL location when the Context Broker issues a command to a device |

# Prerequisites

## Docker

To keep things simple all components will be run using [Docker](https://www.docker.com). **Docker** is a container
technology which allows to different components isolated into their respective environments.

-   To install Docker on Windows follow the instructions [here](https://docs.docker.com/docker-for-windows/)
-   To install Docker on Mac follow the instructions [here](https://docs.docker.com/docker-for-mac/)
-   To install Docker on Linux follow the instructions [here](https://docs.docker.com/install/)

**Docker Compose** is a tool for defining and running multi-container Docker applications. A
[YAML file](https://raw.githubusercontent.com/Fiware/tutorials.Entity-Relationships/master/docker-compose.yml) is used
configure the required services for the application. This means all container services can be brought up in a single
command. Docker Compose is installed by default as part of Docker for Windows and Docker for Mac, however Linux users
will need to follow the instructions found [here](https://docs.docker.com/compose/install/)

You can check your current **Docker** and **Docker Compose** versions using the following commands:

```console
docker-compose -v
docker version
```

Please ensure that you are using Docker version 20.10 or higher and Docker Compose 1.29 or higher and upgrade if
necessary.

## Cygwin

We will start up our services using a simple bash script. Windows users should download [cygwin](http://www.cygwin.com/)
to provide a command-line functionality similar to a Linux distribution on Windows.

# Start Up

Before you start you should ensure that you have obtained or built the necessary Docker images locally. Please clone the
repository and create the necessary images by running the commands as shown:

```console
git clone https://github.com/mohamadsadeque/Projeto_Fiware_IoT
cd Projeto_Fiware_IoT

./services create
```

```console
./services start
```

> [!NOTE]
> If you want to clean up and start over again you can do so with the following command:
>
> ```console
> ./services stop
> ```

# Provisioning an IoT Agent

To follow the tutorial correctly please ensure you have the device monitor page available in your browser and click on
the page to enable audio before you enter any cUrl commands. The device monitor displays the current state of an array
of dummy devices using Ultralight 2.0 syntax

#### Device Monitor

The device monitor can be found at: `http://localhost:3000/device/monitor`

## Checking the IoT Agent Service Health

You can check if the IoT Agent is running by making an HTTP request to the exposed port:

#### 1️⃣ Request:

```console
curl -X GET \
  'http://localhost:4041/iot/about'
```

The response will look similar to the following:

```json
{
    "libVersion": "3.4.0",
    "port": "4041",
    "baseRoot": "/",
    "version": "2.4.0"
}
```

> **What if I get a `Failed to connect to localhost port 4041: Connection refused` Response?**
>
> If you get a `Connection refused` response, the IoT Agent cannot be found where expected for this tutorial - you will
> need to substitute the URL and port in each cUrl command with the corrected IP address. All the cUrl commands tutorial
> assume that the IoT Agent is available on `localhost:4041`.
>
> Try the following remedies:
>
> -   To check that the docker containers are running try the following:
>
> ```console
> docker ps
> ```
>
> You should see four containers running. If the IoT Agent is not running, you can restart the containers as necessary.
> This command will also display open port information.
>
> -   If you have installed [`docker-machine`](https://docs.docker.com/machine/) and
>     [Virtual Box](https://www.virtualbox.org/), the context broker, IoT Agent and Dummy Device docker containers may
>     be running from another IP address - you will need to retrieve the virtual host IP as shown:
>
> ```console
> curl -X GET \
>  'http://$(docker-machine ip default):4041/version'
> ```
>
> Alternatively run all your curl commands from within the container network:
>
> ```console
> docker run --network fiware_default --rm appropriate/curl -s \
>  -X GET 'http://iot-agent:4041/iot/about'
> ```



### Criando um Service Group


#### Request:

```console
curl -iX POST \
  'http://localhost:4041/iot/services' \
  -H 'Content-Type: application/json' \
  -H 'fiware-service: openiot' \
  -H 'fiware-servicepath: /' \
  -d '{
 "services": [
   {
     "apikey":      "4jggokgpepnvsb2uv4s40d59ov",
     "cbroker":     "http://orion:1026",
     "entity_type": "Thing",
     "resource":    "/iot/d"
   }
 ]
}'
```


### Provisioning a Sensor


####  Request para criar o dispositivo:

```console

curl -iX POST \
  'http://localhost:4041/iot/devices' \
  -H 'Content-Type: application/json' \
  -H 'fiware-service: openiot' \
  -H 'fiware-servicepath: /' \
  -d '{
 "devices": [
   {
     "device_id":   "paciente001",
     "entity_name": "urn:ngsi-ld:Paciente:001",
     "entity_type": "Paciente",
     "timezone":    "America/Fortaleza",
     "attributes": [
       { "object_id": "b", "name": "BPM", "type": "Integer" },
       { "object_id": "s", "name": "SO2", "type": "Integer" }
     ],
     "static_attributes": [
       { "name":"refStore", "type": "Relationship", "value": "urn:ngsi-ld:Store:001"}
     ]
   }
 ]
}
'
```

#### Request para alterar valor do sensor:

```console
curl -iX POST \
  'http://localhost:7896/iot/d?k=4jggokgpepnvsb2uv4s40d59ov&i=paciente001' \
  -H 'Content-Type: text/plain' \
  -d 'b|83|s|98'
```
Nesse caso o 'b' se trata da chave para BPM e 's' para SO2

#### Request para ler valor do sensor:

```console
curl -X GET \
  'http://localhost:1026/v2/entities/urn:ngsi-ld:Paciente:001?type=Paciente' \
  -H 'fiware-service: openiot' \
  -H 'fiware-servicepath: /'
```

#### Request para ver dispositiovs:

```console
curl -X GET \
  'http://localhost:4041/iot/devices' \
  -H 'fiware-service: openiot' \
  -H 'fiware-servicepath: /'
```

### Creating a Service Group

This example provisions an anonymous group of devices. It tells the IoT Agent that a series of devices will be sending
messages to the `IOTA_HTTP_PORT` (where the IoT Agent is listening for **Northbound** communications)

####  Request:

```console
curl -iX POST \
  'http://localhost:4041/iot/services' \
  -H 'Content-Type: application/json' \
  -H 'fiware-service: openiot' \
  -H 'fiware-servicepath: /' \
  -d '{
 "services": [
   {
     "apikey":      "12345",
     "cbroker":     "http://orion:1026",
     "entity_type": "Thing",
     "resource":    "/iot/d"
   }
 ]
}'
```

### Read Service Group Details

This example obtains the full details of a provisioned service with a given `resource` path.

Service group details can be read by making a GET request to the `/iot/services` endpoint and providing a `resource`
parameter.

#### 1️⃣7️⃣  Request:

```console
curl -X GET \
  'http://localhost:4041/iot/services?resource=/iot/d' \
  -H 'fiware-service: openiot' \
  -H 'fiware-servicepath: /'
```

#### Response:

```json
{
    "_id": "5b07b2c3d7eec57836ecfed4",
    "subservice": "/",
    "service": "openiot",
    "apikey": "4jggokgpepnvsb2uv4s40d59ov",
    "resource": "/iot/d",
    "attributes": [],
    "lazy": [],
    "commands": [],
    "entity_type": "Thing",
    "internal_attributes": [],
    "static_attributes": []
}
```

The response includes all the defaults associated with each service group such as the `entity_type` and any default
commands or attribute mappings.

### List all Service Groups

This example lists all provisioned services by making a GET request to the `/iot/services` endpoint.

####  Request:

```console
curl -X GET \
  'http://localhost:4041/iot/services' \
  -H 'fiware-service: openiot' \
  -H 'fiware-servicepath: /'
```

#### Response:

```json
{
    "_id": "5b07b2c3d7eec57836ecfed4",
    "subservice": "/",
    "service": "openiot",
    "apikey": "4jggokgpepnvsb2uv4s40d59ov",
    "resource": "/iot/d",
    "attributes": [],
    "lazy": [],
    "commands": [],
    "entity_type": "Thing",
    "internal_attributes": [],
    "static_attributes": []
}
```

The response includes all the defaults associated with each service group such as the `entity_type` and any default
commands or attribute mappings.

### Update a Service Group

####  Request:

```console
curl -iX PUT \
  'http://localhost:4041/iot/services?resource=/iot/d&apikey=4jggokgpepnvsb2uv4s40d59ov' \
  -H 'Content-Type: application/json' \
  -H 'fiware-service: openiot' \
  -H 'fiware-servicepath: /' \
  -d '{
  "entity_type": "IoT-Device"
}'
```

### Deletar um Service Group

####  Request:

```console
curl -iX DELETE \
  'http://localhost:4041/iot/services/?resource=/iot/d&apikey=4jggokgpepnvsb2uv4s40d59ov' \
  -H 'fiware-service: openiot' \
  -H 'fiware-servicepath: /'
```

### Deletar dispositivo

####  Request:

```console
curl -iX DELETE \
  'http://localhost:4041/iot/devices/paciente001' \
  -H 'fiware-service: openiot' \
  -H 'fiware-servicepath: /'
```

