# fiware Tutorial-Step By Step

<aside>
❗ Docker Version 20 이상
Docker Compose Version 2.5 이상
환경에서 진행됨
</aside>

# Step01

## Architecture

![Untitled](/assets/architecture.png)

fiware의 구성요소인 [Orion Context Broker](https://fiware-orion.readthedocs.io/en/latest/)를 사용

Orion Context Broker가 보유하고 있는 Context 데이터의 지속성을 유지하기 위해 MongoDB를 사용

### Orion Context Broker

NGSI를 사용하여 요청을 수신할 Orion Context Broker 설치

```yaml
# docker-compose.yml
version: '3.3'
services:
  orion:
    container_name: fiware-orion
    network_mode: fiware_default
    ports:
      - '1026:1026'
    image: fiware/orion
```

### MongoDB Database

Orion Context Broker에서 데이터 엔티티, 구독 및 등록과 같은 컨텍스트 데이터 정보를 보유하는데 사용

```yaml
# docker-compose.yml
version: '3.3'
services:
  mongo:
    container_name: mongo-db
    network_mode: fiware_default
    expose:
      - '27017'
    image: 'mongo:3.6'
```

## Combine

### .env

```bash
ORION_VERSION=latest
ORION_PORT=1026
MONGO_DB_VERSION=3.6
MONGO_DB_PORT=27017
```

### docker-compose.yml

```yaml
version: '3.8'
services:
  # Orion is the context broker
  orion:
    labels:
      org.fiware: 'tutorial'
    image: fiware/orion:${ORION_VERSION}
    hostname: orion
    container_name: fiware-orion
    depends_on:
      - mongo-db
    networks:
      - default
    ports:
      - '${ORION_PORT}:${ORION_PORT}' # localhost:1026
    command: -dbhost mongo-db -logLevel DEBUG -noCache
    healthcheck:
      test: curl --fail -s http://orion:${ORION_PORT}/version || exit 1
      interval: 5s

  # Databases
  mongo-db:
    labels:
      org.fiware: 'tutorial'
    image: mongo:${MONGO_DB_VERSION}
    hostname: mongo-db
    container_name: db-mongo
    expose:
      - '${MONGO_DB_PORT}'
    ports:
      - '${MONGO_DB_PORT}:${MONGO_DB_PORT}' # localhost:27017 # localhost:27017
    networks:
      - default
    volumes:
      - mongo-db:/data
    healthcheck:
      test: |
        host=`hostname --ip-address || echo '127.0.0.1'`; 
        mongo --quiet $host/test --eval 'quit(db.runCommand({ ping: 1 }).ok ? 0 : 2)' && echo 0 || echo 1
      interval: 5s

networks:
  default:
    labels:
      org.fiware: 'tutorial'
    ipam:
      config:
        - subnet: 172.18.1.0/24

volumes:
  mongo-db: ~
```

## 01. Service Status Check

### Request

```bash
# 01ServiceStatus.http
curl -X GET \
  'http://localhost:1026/version'
```

### Response

```bash
# 01ServiceStatus.http
HTTP/1.1 200 OK
Connection: close
Content-Length: 735
Content-Type: application/json
Fiware-Correlator: 92818f0c-9710-11ed-94da-0242ac120103
Date: Wed, 18 Jan 2023 09:14:55 GMT

{
  "orion": {
    "version": "3.7.0-next",
    "uptime": "0 d, 0 h, 8 m, 22 s",
    "git_hash": "c92ba18320ba692d72a8333dc86fa62bb1cd7ec8",
    "compile_time": "Thu Dec 22 17:48:15 UTC 2022",
    "compiled_by": "root",
    "compiled_in": "70921c94dd4e",
    "release_date": "Thu Dec 22 17:48:15 UTC 2022",
    "machine": "x86_64",
    "doc": "https://fiware-orion.rtfd.io/",
    "libversions": {
      "boost": "1_74",
      "libcurl": "libcurl/7.74.0 OpenSSL/1.1.1n zlib/1.2.11 brotli/1.0.9 libidn2/2.3.0 libpsl/0.21.0 (+libidn2/2.3.0) libssh2/1.9.0 nghttp2/1.43.0 librtmp/2.3",
      "libmosquitto": "2.0.12",
      "libmicrohttpd": "0.9.70",
      "openssl": "1.1",
      "rapidjson": "1.1.0",
      "mongoc": "1.17.4",
      "bson": "1.17.4"
    }
  }
}
```

## 02.Create Context Data01

### Request

```bash
# 02CreateContextData01.http
curl -iX POST \
  'http://localhost:1026/v2/entities' \
  -H 'Content-Type: application/json' \
  -d '
{
    "id": "urn:ngsi-ld:Store:001",
    "type": "Store",
    "address": {
        "type": "PostalAddress",
        "value": {
            "streetAddress": "Bornholmer Straße 65",
            "addressRegion": "Berlin",
            "addressLocality": "Prenzlauer Berg",
            "postalCode": "10439"
        }
    },
    "location": {
        "type": "geo:json",
        "value": {
             "type": "Point",
             "coordinates": [13.3986, 52.5547]
        }
    },
    "name": {
        "type": "Text",
        "value": "Bösebrücke Einkauf"
    }
}'
```

### Response

```bash
# 02CreateContextData01.http
HTTP/1.1 201 Created
Connection: close
Content-Length: 0
Location: /v2/entities/urn:ngsi-ld:Store:001?type=Store
Fiware-Correlator: f0225a60-9710-11ed-8cb4-0242ac120103
Date: Wed, 18 Jan 2023 09:17:32 GMT
```

## 03.Create Context Data02

### Request

```bash
# 03CreateContextData02.http
curl -iX POST \
  'http://localhost:1026/v2/entities' \
  -H 'Content-Type: application/json' \
  -d '
{
    "type": "Store",
    "id": "urn:ngsi-ld:Store:002",
    "address": {
        "type": "PostalAddress",
        "value": {
            "streetAddress": "Friedrichstraße 44",
            "addressRegion": "Berlin",
            "addressLocality": "Kreuzberg",
            "postalCode": "10969"
        }
    },
    "location": {
        "type": "geo:json",
        "value": {
             "type": "Point",
             "coordinates": [13.3903, 52.5075]
        }
    },
    "name": {
        "type": "Text",
        "value": "Checkpoint Markt"
    }
}'
```

### Response

```bash
#03CreateContextData02.http
HTTP/1.1 201 Created
Connection: close
Content-Length: 0
Location: /v2/entities/urn:ngsi-ld:Store:002?type=Store
Fiware-Correlator: 0b28e8f6-9711-11ed-8ae3-0242ac120103
Date: Wed, 18 Jan 2023 09:18:17 GMT
```

## 04. Get Entity By ID

### Request

```bash
# 04GetEntityById
curl -G -X GET \
   'http://localhost:1026/v2/entities/urn:ngsi-ld:Store:001?options=keyValues'
```

### Response

```bash
# 04GetEntityById
HTTP/1.1 200 OK
Connection: close
Content-Length: 269
Content-Type: application/json
Fiware-Correlator: bc30dc6a-9713-11ed-976c-0242ac120103
Date: Wed, 18 Jan 2023 09:37:33 GMT

{
  "id": "urn:ngsi-ld:Store:001",
  "type": "Store",
  "address": {
    "streetAddress": "Bornholmer Straße 65",
    "addressRegion": "Berlin",
    "addressLocality": "Prenzlauer Berg",
    "postalCode": "10439"
  },
  "location": {
    "type": "Point",
    "coordinates": [
      13.3986,
      52.5547
    ]
  },
  "name": "Bösebrücke Einkauf"
}
```

## 05. Get Entity By Type

### Request

```bash
#05GetEntityByType
curl -G -X GET \
    'http://localhost:1026/v2/entities?type=Store&options=keyValues'
```

### Response

```bash
#05GetEntityByType
HTTP/1.1 200 OK
Connection: close
Content-Length: 529
Content-Type: application/json
Fiware-Correlator: 832ef9ae-9715-11ed-8ff1-0242ac120103
Date: Wed, 18 Jan 2023 09:50:16 GMT

[
  {
    "id": "urn:ngsi-ld:Store:001",
    "type": "Store",
    "address": {
      "streetAddress": "Bornholmer Straße 65",
      "addressRegion": "Berlin",
      "addressLocality": "Prenzlauer Berg",
      "postalCode": "10439"
    },
    "location": {
      "type": "Point",
      "coordinates": [
        13.3986,
        52.5547
      ]
    },
    "name": "Bösebrücke Einkauf"
  },
  {
    "id": "urn:ngsi-ld:Store:002",
    "type": "Store",
    "address": {
      "streetAddress": "Friedrichstraße 44",
      "addressRegion": "Berlin",
      "addressLocality": "Kreuzberg",
      "postalCode": "10969"
    },
    "location": {
      "type": "Point",
      "coordinates": [
        13.3903,
        52.5075
      ]
    },
    "name": "Checkpoint Markt"
  }
]
```

## 06. Filter Data Comparing01

### Request

```bash
curl -G -X GET \
    'http://localhost:1026/v2/entities?type=Store&q=address.addressLocality==Kreuzberg&options=keyValues'
```

### Response

```bash
HTTP/1.1 200 OK
Connection: close
Content-Length: 259
Content-Type: application/json
Fiware-Correlator: 3870a5e2-9716-11ed-a0b1-0242ac120103
Date: Wed, 18 Jan 2023 09:55:21 GMT

[
  {
    "id": "urn:ngsi-ld:Store:002",
    "type": "Store",
    "address": {
      "streetAddress": "Friedrichstraße 44",
      "addressRegion": "Berlin",
      "addressLocality": "Kreuzberg",
      "postalCode": "10969"
    },
    "location": {
      "type": "Point",
      "coordinates": [
        13.3903,
        52.5075
      ]
    },
    "name": "Checkpoint Markt"
  }
]
```

## 07. Filter Data Comparing02

### Request

```bash
# 07FilterDataComparing02.http
curl -G -X GET \
  'http://localhost:1026/v2/entities?type=Store&georel=near;maxDistance:1500&geometry=point&coords=52.5162,13.3777&options=keyValues'
```

### Response

```bash
# 07FilterDataComparing02.http
HTTP/1.1 200 OK
Connection: close
Content-Length: 259
Content-Type: application/json
Fiware-Correlator: a0d12ab6-9717-11ed-ba2c-0242ac120103
Date: Wed, 18 Jan 2023 10:05:25 GMT

[
  {
    "id": "urn:ngsi-ld:Store:002",
    "type": "Store",
    "address": {
      "streetAddress": "Friedrichstraße 44",
      "addressRegion": "Berlin",
      "addressLocality": "Kreuzberg",
      "postalCode": "10969"
    },
    "location": {
      "type": "Point",
      "coordinates": [
        13.3903,
        52.5075
      ]
    },
    "name": "Checkpoint Markt"
  }
]
```
