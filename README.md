# API-Addicts demo

## Prerequisites

* Python
* Kong
* httpie

## Step 1

```
python -m SimpleHTTPServer 10000 &
```

Should give us an upstream server in

```
http http://localhost:10000/temperatures.json
```

The server can be killed with

```
killall python
```

## Step 2 - Plain wrapper

kong.yml:

```
_format_version: "2.1"
_transform: true

services:
- name: weather-service
  url: http://localhost:10000/temperatures.json
  routes:
  - name: raw
    paths: ["/"]
```

```
KONG_DATABASE=off KONG_DECLARATIVE_CONFIG=kong.yml KONG_LOG_LEVEL=debug kong start
```

Use kong's Admin API to check that the config was loaded correctly

```
http :8001/services
http :8001/services/weather-service/routes
```

Access the upstream server through Kong


```
http :8001
```



