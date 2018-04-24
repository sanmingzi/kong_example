# Kong tourist with docker

## Environment

- Linux x86_64
- Docker version 17.09.1-ce
- docker-compose version 1.11.1

## Run kong with docker

- command

```
docker-compose up -d
```

- validate containers running

```
docker ps -a
# there will be three up containers
# kong-dashboard
# kong
# kong-database
```

- validate the kong-dashboard

```
http://your-host:8080
```

## Add api

- command

```
curl -d "name=cats" \
     -d "uris=/cats" \
     -d "upstream_url=http://mockbin.org/" \
     http://0.0.0.0:8001/apis/
```

- validate the api

```
http://your-host:8000/cats
```

## Add global oauth plugin

- command

```
curl -X POST http://0.0.0.0:8001/plugins \
     -d 'name=oauth2' \
     -d 'config.enable_client_credentials=true' \
     -d 'config.enable_authorization_code=true' \
     -d 'config.enable_password_grant=true' \
     -d 'config.scopes=profile' \
     -d 'config.global_credentials=true'
```

```
{
  "created_at": 1524539680000,
  "config": {
    "refresh_token_ttl": 1209600,
    "token_expiration": 7200,
    "mandatory_scope": false,
    "scopes": [
      "profile"
    ],
    "hide_credentials": false,
    "enable_client_credentials": true,
    "enable_implicit_grant": false,
    "global_credentials": true,
    "anonymous": "",
    "enable_password_grant": true,
    "accept_http_if_already_terminated": false,
    "enable_authorization_code": true,
    "provision_key": "Pg46iWFzNMG1xlobUUVNevsdqLqCImDA",
    "auth_header_name": "authorization"
  },
  "id": "c6a4425b-c2e0-42e0-8824-7f1a9ca19c4b",
  "enabled": true,
  "name": "oauth2"
}
```

When we create the oauth plugin, the response include the provision_key, this will be used in the next.

- validate the plugin is working

After we create a plugin oauth, the api 'cats' we create before can not be visited.

```
http://your-host:8000/cats
# {"error_description":"The access token is missing","error":"invalid_request"}
```

## Add consumer

```
curl -d "username=thefosk" \
     http://0.0.0.0:8001/consumers/
```

## Add application for consumer

- command

```
curl -d "name=Hello World App" \
     -d "redirect_uri=http://getkong.org/" \
     http://0.0.0.0:8001/consumers/thefosk/oauth2/
```

```
{
  "client_id": "3f1NkHXMFWTJezH9o3osXHxAFKzYM78j",
  "created_at": 1524539069000,
  "id": "ddd15614-060e-43cb-801b-c9c9992bcd5c",
  "redirect_uri": [
    "http://getkong.org/"
  ],
  "name": "Hello World App",
  "client_secret": "nc7ATBB9KqXsyMyng90pYMMKAHZC9NgC",
  "consumer_id": "5b91dd08-3c2c-4076-9fc7-21726fbb6b45"
}
```

When we create an application, the response include client_id/client_secret, it is useful.

## Authenticate

We use two way to authenticate with oauth.

- [Client Credentials](##Client-Credentials)
- [Authorization Code](##Authorization-Code)

## Client Credentials

- retrieve access_token

```
curl -X POST https://0.0.0.0:8443/cats/oauth2/token \
     -d 'grant_type=client_credentials' \
     -d 'client_id=3f1NkHXMFWTJezH9o3osXHxAFKzYM78j' \
     -d 'client_secret=nc7ATBB9KqXsyMyng90pYMMKAHZC9NgC' --insecure
```

```
{
  "token_type": "bearer",
  "access_token": "t5xDwOiA56a8HhNdDAOQ7FwiNKLyZp1q",
  "expires_in": 7200
}
```

- validate the access_token

```
http://your-host:8000/cats?access_token=t5xDwOiA56a8HhNdDAOQ7FwiNKLyZp1q
```

## Authorization Code

- retrieve code

```
curl -X POST https://0.0.0.0:8443/cats/oauth2/authorize \
    --data "response_type=code" \
    --data "client_id=3f1NkHXMFWTJezH9o3osXHxAFKzYM78j" \
    --data "provision_key=Pg46iWFzNMG1xlobUUVNevsdqLqCImDA" \
    --data "authenticated_userid=1" \
    --insecure
```

```
{
  "redirect_uri": "http://getkong.org/?code=BNLHpBrYVfutiss31jWdFixvnh6KH3PW"
}
```

- retrieve access_token

```
curl -X POST https://0.0.0.0:8443/cats/oauth2/token \
    --data client_id=3f1NkHXMFWTJezH9o3osXHxAFKzYM78j \
    --data client_secret=nc7ATBB9KqXsyMyng90pYMMKAHZC9NgC \
    --data provision_key=Pg46iWFzNMG1xlobUUVNevsdqLqCImDA \
    --data code=BNLHpBrYVfutiss31jWdFixvnh6KH3PW \
    --data grant_type=authorization_code \
    --insecure
```

```
{
  "refresh_token": "bfGWcHdy8afGw7E6X8rkE6zYYWTro4MM",
  "token_type": "bearer",
  "access_token": "jmt9mN2VOCpiHRfLIt1wHvjQfiDROouw",
  "expires_in": 7200
}
```

- validate the access_token

```
http://your-host:8000/cats?access_token=jmt9mN2VOCpiHRfLIt1wHvjQfiDROouw
```
