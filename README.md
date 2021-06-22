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

## Step 3 - Versioning via headers

kong.yml:
```
_format_version: "2.1"
_transform: true

services:
- name: weather-service
  url: http://localhost:10000/temperatures.json
  routes:
  - name: v0.0.0
    headers:
      "Accept-Version": ["0.0.0"]
```
Note that we replaced the "raw" route with "v0.0.0"

```
KONG_DATABASE=off KONG_DECLARATIVE_CONFIG=kong.yml KONG_LOG_LEVEL=debug kong restart
```

Try to access the upstream server without header version won't work

```
http :8000
```

But passing `Accept-Version` will work

```
http :8000 Accept-Version:0.0.0
```


## Step 4 - Modify the response, releasing a new version

kong.yml:
```
_format_version: "2.1"
_transform: true

services:
- name: weather-service
  url: http://localhost:10000/temperatures.json
  routes:
  - name: v0.0.0
    headers:
      "Accept-Version": ["0.0.0"]
  - name: v0.1.0
    headers:
      "Accept-Version": ["0.1.0"]
    plugins:
    - name: response-transformer
      config:
        remove:
          json:
            - London
```

```
KONG_DATABASE=off KONG_DECLARATIVE_CONFIG=kong.yml KONG_LOG_LEVEL=debug kong restart
```

Now the API is versioned:

```
http :8000 Accept-Version:0.0.0
http :8000 Accept-Version:0.1.0
```

Version 0.1.0 should not have London


