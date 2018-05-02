# Kong access control with acl plugin

This article will show you how to use acl and auth plugin to control accesses.

In the next, I will create two consumer/group(test/dev) and two apis(api1/api2).
The '1' mean can access the api, '0' mean can not access the api.
The consumer test can only access the api1 and the consumer dev can access the two apis.

| consumer | group | api1 | api2 |
| :---: | :---: | :---: | :---: |
| test | test | 1 | 0 |
| dev | dev | 1 | 1 |

## Create two api

```
curl -i -X POST \
  --url http://0.0.0.0:8001/apis/ \
  --data 'name=api1' \
  --data 'uris=/api1' \
  --data 'upstream_url=http://www.google.com/'
```

visit http://your-host:8000/api1 to validate the api1.

```
curl -i -X POST \
  --url http://0.0.0.0:8001/apis/ \
  --data 'name=api2' \
  --data 'uris=/api2' \
  --data 'upstream_url=http://www.bing.com/'
```

visit http://your-host:8000/api2 to validate the api2.

## Add acl plugin to api

```
curl -X POST http://0.0.0.0:8001/apis/api1/plugins \
    --data "name=acl" \
    --data "config.whitelist=test, dev"
```

```
curl -X POST http://0.0.0.0:8001/apis/api2/plugins \
    --data "name=acl" \
    --data "config.whitelist=dev"
```

There is no consumer now, so when you visit api1/api2, it should return:

```
{"message":"You cannot consume this service"}
```

## Add jwt plugin to apis

The acl plugin can be work with the other auth plugin, I choose the jwt plugin.

```
curl -X POST http://0.0.0.0:8001/plugins \
    --data 'name=jwt'
```

## Add consumers

```
curl -X POST http://0.0.0.0:8001/consumers \
    --data "username=test"
```

```
curl -X POST http://0.0.0.0:8001/consumers \
    --data "username=dev"
```

## Assign group to consumer

```
curl -X POST http://0.0.0.0:8001/consumers/test/acls \
    --data "group=test"

curl -X POST http://0.0.0.0:8001/consumers/dev/acls \
    --data "group=dev"
```

## Add jwt plugin to consumer

```
curl -X POST http://0.0.0.0:8001/consumers/test/jwt

{
  "created_at": 1524820302000,
  "id": "0677db90-f221-4b3b-b3e2-127557ceba95",
  "algorithm": "HS256",
  "key": "2ySARDGStXpRmy5bAalEqqWb5jWspLZx",
  "secret": "9AEnySWTftLHieJKnlzYJjd67kNKSWQ5",
  "consumer_id": "8ef71aef-d531-4d5e-bddc-5b3297780312"
}

eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiIyeVNBUkRHU3RYcFJteTViQWFsRXFxV2I1aldzcExaeCJ9.Q_Nj6blBYlgwGXTg2AUYvTlLtdU0DGrZxWOLarjOjpw
```

We can use [https://jwt.io/](https://jwt.io/) to generate a jwt. The iss is the key in front.

```
curl -X POST http://0.0.0.0:8001/consumers/dev/jwt

{
  "created_at": 1524820706000,
  "id": "4341d4bb-7fe1-4f8b-80b9-628a28c3ec71",
  "algorithm": "HS256",
  "key": "cwrnjHKGQtjMiGbzwAMHrqceTv9CYOQy",
  "secret": "tooX2ou5NH1WjZhFljV23tDrLBpGKINd",
  "consumer_id": "ea28013b-b737-4068-93d2-82bed4e9bd12"
}

eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJjd3JuakhLR1F0ak1pR2J6d0FNSHJxY2VUdjlDWU9ReSJ9.eO88GKjBIRncry4I5m9Zmcp7G14gNDoI_3GVoAA9gnI
```

## Validate the access

```
http://your-host:8000/api1?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiIyeVNBUkRHU3RYcFJteTViQWFsRXFxV2I1aldzcExaeCJ9.Q_Nj6blBYlgwGXTg2AUYvTlLtdU0DGrZxWOLarjOjpw

=> google.com

http://your-host:8000/api2?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiIyeVNBUkRHU3RYcFJteTViQWFsRXFxV2I1aldzcExaeCJ9.Q_Nj6blBYlgwGXTg2AUYvTlLtdU0DGrZxWOLarjOjpw

=> {"message":"You cannot consume this service"}

http://your-host:8000/api1?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJjd3JuakhLR1F0ak1pR2J6d0FNSHJxY2VUdjlDWU9ReSJ9.eO88GKjBIRncry4I5m9Zmcp7G14gNDoI_3GVoAA9gnI

=> google.com

http://your-host:8000/api2?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJjd3JuakhLR1F0ak1pR2J6d0FNSHJxY2VUdjlDWU9ReSJ9.eO88GKjBIRncry4I5m9Zmcp7G14gNDoI_3GVoAA9gnI

=> bing.com
```

When we use the jwt of the consumer 'test', it can only visit the api1.
When we use the jwt of the consumer 'dev', it can visit both the two apis.

After I add the group 'dev' to consumer 'test', it also can visit the two apis.

## Expire

The JWT plugin support the expire feature.

```
curl -X PATCH http://kong:8001//plugins/{jwt plugin id} \
    --data "config.claims_to_verify=exp"
```

After we update the config of jwt plugin, we should add expire time into the json-web-token.
The 'exp' should be a unix timestamp integer(second). The jwt plugin will check the exp column.

```
{
  "alg": "HS256",
  "typ": "JWT"
}

{
  "iss": "2ySARDGStXpRmy5bAalEqqWb5jWspLZx",
  "exp": 1525241832
}

HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  your-256-bit-secret
)
```

## Conclusion

We can user acl plugin and one auth plugin to control the access in kong.

## Advance

We can specified the secret of jwt.
Maybe we can set all consumers have the same secret, the kong can decode the jwt by the secret, the iss(key) is different, so kong know which consumer it is.
If all the secret is same, we can put more info in jwt.
