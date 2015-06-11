---
title: Exist OAuth2 reference

language_tabs:
  - shell
  - python

toc_footers:
  - <a href="https://exist.io">Sign up for an account</a>
  - <a href="https://confirmsubscription.com/h/t/2E3A4057D66FD499">Developer mailing list</a>
  - <a href="http://github.com/tripit/slate">Docs powered by Slate</a>

includes:
  

search: true
---

# Introduction

Welcome! These docs are private and document the OAuth2 client flow for Exist client applications. Please do not share these docs with anyone.

# Getting started

The OAuth2 endpoints live at `https://exist.io/oauth2/`. All requests must use HTTPS.

POST bodies must be sent as `application/x-www-form-urlencoded`. The API will only return JSON.

There is no rate-limiting on the OAuth2 endpoints, but updating attributes is limited to 300 requests/hr.

Terminology note: we use **client** to refer to the OAuth2 client. A client application which writes data to attributes is termed a **service**. 

# Authentication

## The OAuth2 authorisation flow

> Send your user to the authorisation page at `/oauth2/authorize`

```shell
# We can't really do this from the shell, but your URL would look like this:

curl https://exist.io/oauth2/authorize?response_type=code&client_id=[your_id]&redirect_uri=[your_uri]&scope=[your_scope]
```

```python
# in django, we would do something like this
return redirect('https://exist.io/oauth2/authorize?response_type=code&client_id=%s&redirect_uri=%s&scope=%s' % (CLIENT_ID, REDIRECT_URI,"read+write"))
```

> User authorises your client by hitting 'Allow', and
> Exist returns the user to your `redirect_uri` with `code=[some_code]` in the query string.
> Exchange your code for an access token: 

```shell
curl -X POST https://exist.io/oauth2/access_token -d "grant_type=authorization_code" -d "code=[some_code]" -d "client_id=[your_id]" -d "client_secret=[your_secret]" -d "redirect_uri=[your_uri]"
```

```python
import requests

url = 'https://exist.io/oauth2/access_token'

response = requests.post(url,
           {'grant_type':'authorization_code',
            'code':code,
            'client_id':CLIENT_ID,
            'client_secret':CLIENT_SECRET,
            'redirect_uri':REDIRECT_URI })
```

> Returns JSON if your request was successful:

```json
{ "access_token": "122bb8707b6aee134e7746a40feca41868ddd578", "token_type": "Bearer", "expires_in": 31535999, "refresh_token": "ac45027ad037f53b3ce91be272b163f55a4a87e9", "scope": "read write read+write" }
```

The OAuth2 authorisation flow is vastly simpler than the original OAuth 1.0:

1. Send your user to the "request authorisation" page at `/oauth2/authorize` with these parameters:
  *  `response_type=code` to request an auth code in return   
  *  `redirect_uri` with the URI to which Exist returns the user
  *  `scope=read` or `scope=read+write` to request read or read/write permissions
  *  `client_id` which is your OAuth2 client ID
2. User authorises your application within the requested scopes (by hitting 'Allow' in the browser)
3. Exist returns the user to your `redirect_uri` (GET request) with the following:
  *  `code` parameter upon success
  *  `error` parameter if the user didn't authorise your client, or any other error with your request
4. Exchange this code for an access token by POSTing to `/oauth2/access_token` these parameters:
  *  `grant_type=authorization_code`
  *  `code` with the code you just received
  *  `client_id` with your OAuth2 client ID
  *  `client_secret` with your OAuth2 client secret
  *  `redirect_uri` with the URI you used earlier
5. If successful you will receive a JSON object with an `access_token`, `refresh_token`, `token_type`, `scope`, and `expires_in` time in seconds. 

## Refreshing an access token

```shell
curl -X POST https://exist.io/oauth2/access_token -d "grant_type=refresh_token" -d "refresh_token=[token]" -d "client_id=[your_id]" -d "client_secret=[your_secret]"
```

```python
import requests

url = 'https://exist.io/oauth2/access_token'

response = requests.post(url,
           {'grant_type':'refresh_token',
            'refresh_token':token,
            'client_id':CLIENT_ID,
            'client_secret':CLIENT_SECRET 
           })
```

> Returns JSON if your request was successful:

```json
{ "access_token": "122bb8707b6aee134e7746a40feca41868ddd578", "token_type": "Bearer", "expires_in": 31535999, "refresh_token": "ac45027ad037f53b3ce91be272b163f55a4a87e9", "scope": "read write read+write" }
```

Tokens expire in a year and can be refreshed at any time, invalidating the original access and refresh tokens.


### Request

`POST /oauth2/access_token`

### Parameters

Name  | Description
------|--------
`refresh_token` | The refresh token previously received in the auth flow
`grant_type` | `refresh_token`
`client_id` | Your OAuth2 client ID
`client_secret` | Your OAuth2 client secret

### Response

The same as your original access token response, a JSON object with an `access_token`, `refresh_token`, `token_type`, `scope`, and `expires_in` time in seconds. 

## Signing requests

```python
import requests

requests.post(url,
    headers={'Authorization':'Bearer 96524c5ca126d87eb18ee7eff408ca0e71e94737'})
```

```shell
# With curl, you can just pass the correct header with each request
curl "api_endpoint_here"
  -H "Authorization: Bearer 96524c5ca126d87eb18ee7eff408ca0e71e94737"
```

Sign all authenticated requests by adding the Authorization header, `Authorization: Bearer [access_token]`. Note that this differs from the token-based authentication by using `Bearer`, *not* `Token`.


# Attribute ownership

Only one client service can have ownership of any user attribute at any given time. Services must acquire ownership of an attribute to be able to write data for this attribute, and can release ownership if needed, for example if the user closes their account with this service or chooses to turn off certain attributes.

## Acquire attributes

```shell
curl https://exist.io/api/1/attributes/acquire/ -H "Content-Type: application/json" -H "Authorization: Bearer 96524c5ca126d87eb18ee7eff408ca0e71e94737" -X POST -d '[{"name":"mood", "active":true}, {"name":"mood_note", "active":true}]'
```

```python
import requests, json

url = 'https://exist.io/api/1/attributes/acquire/'

attributes = [{"name":"mood", "active":True}, {"name":"mood_note", "active":True}]

response = requests.post(url, headers={'Authorization':'Bearer 96524c5ca126d87eb18ee7eff408ca0e71e94737'},
    data=json.dumps(attributes))
```

> Returns JSON and a status code of `202 Accepted` if some attributes failed (just for example, the above is correct)

```json
{ "success": [ 
    { "name":"mood_note",
      "active":"true"
    }
  ],
  "failed": [
    { "name":"mood",
      "error_code":"missing_field",
      "error":"Object at index 0 missing field(s) 'active'"
    }
  ]
}
```

Allows a service to update attribute data for these attributes. Users do not have to approve this (mostly because this would be cumbersome) so please explain/confirm this behaviour with users within your own application.

### Request

`POST /api/1/attributes/acquire/`

### Parameters

Clients must send a JSON-encoded array of objects, where each object contains a `name` string and an `active` boolean. Setting `active` to `false` indicates you'd like to deactivate this attribute without giving up ownership.

Name  | Description
------|--------
`name` | The attribute name, eg. `mood_note`
`active` | `true` or `false` to set this attribute to active or inactive
`private` | Optional `true` or `false` to change the privacy status of this attribute. Please notify users if you are makeing previously private attributes public and only do this with good reason.

### Response

Returns `200 OK` if all attributes were processed successfully, or `202 Accepted` if some attributes failed. The content is a JSON object containing `success` and `failed` arrays, where each item in the array is an attribute sent in the prior request. Failed attributes get `error` and `error_code` fields added. 

## Release attributes

```shell
curl https://exist.io/api/1/attributes/release/ -H "Content-Type: application/json" -H "Authorization: Bearer 96524c5ca126d87eb18ee7eff408ca0e71e94737" -X POST -d '[{"name":"mood"}, {"name":"mood_note"}]'
```

```python
import requests, json

url = 'https://exist.io/api/1/attributes/release/'

attributes = [{"name":"mood"}, {"name":"mood_note"}]

response = requests.post(url, headers={'Authorization':'Bearer 96524c5ca126d87eb18ee7eff408ca0e71e94737'}, 
    data=json.dumps(attributes))
```

> Returns JSON and a status code of `202 Accepted` if some attributes failed (just for example, the above is correct)

```json
{ "success": [ 
    { "name":"mood_note" }
  ],
  "failed": [
    { "name":"mood",
      "error_code":"unauthorised",
      "error":"Attribute 'mood' does not belong to this service"
    }
  ]
}
```

Do this to release your ownership of any attributes. The attributes' ownership will pass to another service, if the user has another supplied that has indicated it can handle this attribute, or otherwise become inactive.

### Request

`POST /api/1/attributes/release/`

### Parameters

Clients must send a JSON-encoded array of objects, where each object contains a `name` string. The objects may seem superfluous but this is to be consistent with the `acquire` endpoint.

Name  | Description
------|--------
`name` | The attribute name, eg. `mood_note`

### Response

Returns `200 OK` if all attributes were processed successfully, or `202 Accepted` if some attributes failed. The content is a JSON object containing `success` and `failed` arrays, where each item in the array is an attribute sent in the prior request. Failed attributes get `error` and `error_code` fields added. 


## List owned attributes

```shell
curl https://exist.io/api/1/attributes/owned/ -H "Authorization: Bearer 96524c5ca126d87eb18ee7eff408ca0e71e94737"
```

```python
import requests

url = 'https://exist.io/api/1/attributes/owned/'

response = requests.get(url, headers={'Authorization':'Bearer 96524c5ca126d87eb18ee7eff408ca0e71e94737'})
```

> Returns a JSON array of attributes for the authenticated user and owned by this service:

```json
[
    {
        "attribute": "steps", 
        "label": "Steps", 
        "value": null, 
        "service": "fitbit", 
        "priority": 1, 
        "private": false, 
        "value_type": 0, 
        "value_type_description": "Integer"
    }, 
    {
        "attribute": "steps_active_min", 
        "label": "Active minutes", 
        "value": null, 
        "service": "fitbit", 
        "priority": 2, 
        "private": false, 
        "value_type": 0, 
        "value_type_description": "Integer"
    }
]
```

This is a convenience endpoint to list all attributes for the authenticated user currently owned by this service.

### Request

`GET /api/1/attributes/owned/`

# Updating attributes 

```shell
shell
curl https://exist.io/api/1/attributes/update/ -H "Content-Type: application/json" -H "Authorization: Bearer 96524c5ca126d87eb18ee7eff408ca0e71e94737" -X POST -d '[{"name":"mood", "date":"2015-05-20", "value":5}, {"name":"mood_note", "date":"2015-05-20", "value":"Great day playing with the Exist API"}]'
```

```python
python
import requests, json

url = 'https://exist.io/api/1/attributes/update/'

attributes = [{"name":"mood", "date":"2015-05-20", "value":5}, {"name":"mood_note", "date":"2015-05-20", "value":"Great day playing with the Exist API"}]

response = requests.post(url, headers={'Authorization':'Bearer 96524c5ca126d87eb18ee7eff408ca0e71e94737'},
    data=json.dumps(attributes))
```

> Returns a JSON object containing successful and failed updates:

```json
{ "success": [ 
    { "name":"mood_note",
      "date":"2015-05-20",
      "value":"Great day playing with the Exist API"
    }
  ],
  "failed": [
    { "name":"mood",
      "date":"2015-05-20",  
      "error_code":"missing_field",
      "error":"Object at index 0 missing field(s) 'value'"
    }
  ]
}
```

This endpoint allows services to update attribute data for the authenticated user. Data is stored on a single day granularity, so each update contains `name`, `date`, and `value`. Make sure the date is local to the user — though you do not have to worry about timezones directly, if you are using your local time instead of the user's local time, you may be a day ahead or behind!

Valid values are described by the attribute's `value_type` and `value_type_description` fields. However, values are only validated broadly by type and so care must be taken to send correct data. For example, `mood` is of type `integer` so any integer value would be accepted, but *actual* valid values are only within 1-5. 

### Request

`POST /api/1/attributes/update/`

### Parameters

Clients must send a JSON-encoded array of objects containing a `name`, `date`, and `value`.

Name  | Description
------|--------
`name` | The attribute name, eg. `mood_note`
`date` | String of format `YYYY-mm-dd`
`value` | A valid value for this attribute type: string, integer, or float


### Response

Returns `200 OK` if all attributes were processed successfully, or `202 Accepted` if some attributes failed. The content is a JSON object containing `success` and `failed` arrays, where each item in the array is an attribute sent in the prior request. Failed attributes get `error` and `error_code` fields added. 

