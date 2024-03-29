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

## Step 5 - Authentication

kong.yml (only showing new route and Consumer with credentials)
```
_format_version: "2.1"
_transform: true

services:
  ...
  - name: v0.2.0
    headers:
      "Accept-Version": ["0.2.0"]
    plugins:
    - name: key-auth

consumers:
  - username: pepe
    keyauth_credentials:
    - key: secret
```

```
KONG_DATABASE=off KONG_DECLARATIVE_CONFIG=kong.yml KONG_LOG_LEVEL=debug kong restart
```

This new api will require authentication

```
http :8000 Accept-Version:0.2.0 # fail with 401
```

But will accept a request with the secret key:

```
http :8000 Accept-Version:0.2.0 apikey:secret
```

## Step 6 - Ratelimiting

kong.yml (modify v0.2.0)
```
_format_version: "2.1"
_transform: true

services:
  ...
  - name: v0.2.0
    headers:
      "Accept-Version": ["0.2.0"]
    plugins:
    - name: key-auth
    - name: rate-limiting
      config:
        minute: 5
        limit_by: consumer

consumers:
- username: pepe
  keyauth_credentials:
  - key: secret
- username: maria
  keyauth_credentials:
  - key: secret2
```

```
KONG_DATABASE=off KONG_DECLARATIVE_CONFIG=kong.yml KONG_LOG_LEVEL=debug kong restart
```

The 6th request in 1 minute will get ratelimited for the first consumer:

```
http :8000 Accept-Version:0.2.0 apikey:secret
http :8000 Accept-Version:0.2.0 apikey:secret
http :8000 Accept-Version:0.2.0 apikey:secret
http :8000 Accept-Version:0.2.0 apikey:secret
http :8000 Accept-Version:0.2.0 apikey:secret
http :8000 Accept-Version:0.2.0 apikey:secret # ratelimited
```

Other consumers can still use the ratelimited API:
```
http :8000 Accept-Version:0.2.0 apikey:secret2 # works
```

## Step 7 - Caching

kong.yml (keep modifying v0.2.0)
```
_format_version: "2.1"
_transform: true

services:
  ...
  - name: v0.2.0
    ...
    plugins:
    ...
    - name: proxy-cache
      config:
        strategy: memory
        cache_ttl: 3600
```

```
KONG_DATABASE=off KONG_DECLARATIVE_CONFIG=kong.yml KONG_LOG_LEVEL=debug kong restart
```

Test that the response still arrives:

```
http :8000 Accept-Version:0.2.0 apikey:secret
```

Kill the upstream server:
```
killall python
```

Check that v0.0.0 and v0.1.0 don't work any more:
```
http :8000 Accept-Version:0.0.0 apikey:secret
http :8000 Accept-Version:0.1.0 apikey:secret
```

But v2.0.0 still works (and can be rate-limited by consumer):
```
http :8000 Accept-Version:0.2.0 apikey:secret
```

## Step 8 - Mock Servers

Add service-less route:

kong.yml
```
routes:
- name: mock-server
  headers:
    "Mock": ["true"]
  plugins:
  - name: post-function
    config:
      access:
      - |
          local req_path = kong.request.get_path()
          if req_path == "/foo" then
            kong.response.exit(200, { message = "Bar" })
          end
          kong.response.exit(200, { message = "Baz" })
```


```
KONG_DATABASE=off KONG_DECLARATIVE_CONFIG=kong.yml KONG_LOG_LEVEL=debug kong restart
```

Test that the new mockserver responds with "Baz" by default

```
http :8000 Mock:true
```

Test that the new mockserver responds with "Bar" with the correct path
```
http :8000/foo Mock:true
```
