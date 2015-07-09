---
title: Exist API reference

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

Welcome! So you want to take your Exist data and do interesting things with it. Great!
For now, the API can only be accessed by user accounts. [Sign up](https://exist.io) to give it a try.

This is draft documentation for the official Exist API. Both the API and
the docs are liable to change at any time, though we'll do our best to document changes here and keep the docs in sync with the API itself.

**Everybody is welcome to use read-only endpoints and token-based authentication, but you must apply for access to an OAuth2 client for the ability to write data.**
If you'd like to build a client that others can use, [get in touch](mailto:hello@exist.io?Subject=Exist OAuth2 API access) with details of your plans.
We'd love for you to help us get more into and out of Exist.

# Getting started

The API lives at `https://exist.io/api/`. All requests must use HTTPS.

POST bodies can be sent as `application/json`, `application/x-www-form-urlencoded`, or `multipart/form-data`.
However the API will only return JSON. XML is so last decade.

Requests are currently rate-limited at 300/hr per user token. Given user data does not change frequently within
an hourly period (if at all) this should be more than adequate.

Wherever you see the `username` argument in a URL you may substitute the special keyword `$self` to request the authenticated user.

# Object types and terminology

## Clients and services

We use **client** to refer to the OAuth2 client. A client application which writes data to attributes is termed a **service**. 

## Users

```json
{
    "id": 1,
    "username": "josh",
    "first_name": "Josh",
    "last_name": "Sharp",
    "bio": "I made this thing you're using.",
    "url": "http://hellocode.co/",
    "avatar": "https://exist.io/static/media/avatars/josh_2.png",
    "timezone": "Australia/Melbourne",
    "local_time": "2020-05-08T20:56:20.417+10:00",
    "private": false,
    "imperial_units": false
}
```
Users are pretty self-explanatory.

Most of the time you will only be concerned with the currently authenticated user, so
wherever you see the `username` argument in a URL you may substitute the special keyword `$self` to request the
authenticated user.

## Attributes

```json
{
    "attribute": "steps",
    "label": "Steps",
    "group": 
        {
            "name": "activity",
            "priority": 1
        },
    "priority": 1,
    "service": "Fitbit",
    "value_type": 0,
    "value_type_description": "Integer",
    "private": false,
    "values": [
        {
            "value": 3725,
            "date": "2015-05-08"
        },
        {
            "value": 6177,
            "date": "2015-05-07"
        },
        {
            "value": 7811,
            "date": "2015-05-06"
        },
        {
            "value": 6632,
            "date": "2015-05-05"
        },
        {
            "value": 6014,
            "date": "2015-05-04"
        },
        {
            "value": 4376,
            "date": "2015-05-03"
        },
        {
            "value": 8671,
            "date": "2015-05-02"
        },
        {
            "value": 4658,
            "date": "2015-05-01"
        }
    ]
}

```

This is our name for data points or individual numbers a user can track about themselves.
These are tracked at a single day granularity — there's no per-hour or -minute data. Attributes can be
strings but are usually quantifiable, ie. integers or floats.

An attribute has many values, one for each day that it has been tracked. In some responses, you'll see the `value` property
reflected within an attribute object. At other times, particularly if you are requesting multiple days of data, the `values` property will
contain an array of `date`/`value` pairs.

If there is no data for a particular date, this will be reflected with a null value — you should expect to
receive a list of results containing every single day, rather than days without data being left out.

All datetimes are in UTC unless otherwise specified, and should have the user's timezone applied to create a local TZ-aware datetime. All dates are local to the user.

Values are always stored internally and returned in metric units. Each user object contains an `imperial_units` boolean which must be respected when formatting values for the user,
ie. if this is `true`, a `steps_distance` value must be converted from km to miles.

In situations where multiple attributes are requested, for example the "today" overview, these attributes will be returned **grouped**.
A group represents attributes that belong together, for example the "activity" group contains the attributes `steps`, `active_min`, etc.
You can see an example of this in the ['overview' endpoint](#get-current-overview-for-user).

Clients should display attributes in these groups when displaying multiple attributes.
Groups are currently fairly broad and may change as we add more supported attributes.

See [list of supported attributes](#list-of-attributes).

## Correlations

```json
{
    "date": "2015-05-03",
    "period": 90,
    "attribute": "steps",
    "attribute2": "floors",
    "value": 0.697409107405056,
    "p": 2.22989517679296e-14,
    "first_person": "I am much more likely (70%) to climb floors on days I take more steps.",
    "second_person": "You are much more likely (70%) to climb floors on days you take more steps."
}
```

Correlations are a measure of the relationship between two variables. A positive
correlation implies that as one variable increases (its values get higher), so
does the other. A negative correlation implies that as one variable increases,
the other decreases. We present these to users as a way to explain past trends,
and use them to predict future behaviour.

Correlation values vary between -1 and +1 with 0 implying no correlation.
Correlations of -1 or +1 imply an exact linear relationship.

P-values are a way of determining confidence in the result. A value lower than
0.05 is generally considered "statistically significant".

We create simple English sentences to represent each possible correlation as a combination of attributes.

We're also careful to represent these as correlations only, not as one attribute directly causing a change in
the other, and we ask that you do the same. **Correlation is not causation**.
There may be many hidden factors which underlie the relationship between two attributes. It is up to the user to
determine the cause.

Correlations are generated weekly, so past values are also available in order to chart their changing strength.

### Examples

**"You are moderately more likely (55%) to be productive when you listen to music."**

In this case the value is 0.55, and this is a positive correlation — when one value (*tracks played*) increases,
so does the other (*time spent productively*).

**"You are somewhat less likely (25%) to be active when it's warmer in the day."**

In this case the value is -0.25, and this is a negative correlation — when one value (*max temp*) increases,
the other decreases (*time active*).

## Insights

```json
{
    "created": "2015-05-08T05:05:46Z",
    "target_date": "2015-05-07",
    "html": "<div class=\"secondary\">Thursday marks a new productivity streak!</div>",
    "value": "3 days",
    "value2": "3",
    "comment": null,
    "type": {
        "name": "productive_min_goal_best_streak",
        "period": 1,
        "priority": 2,
        "attribute": {
            "name": "productive_min",
            "label": "Productive time",
            "group": {
                "name": "productivity",
                "priority": 2
            },
            "priority": 1
        }
    }
}
```

Insights are interesting events found within the user's data. These are not triggered by the user but generated automatically if the user's data fits an insight type's criteria. Typically these fall into a few categories: day-level events, for example, yesterday was the highest or lowest steps value in however many days; and week and month-level events, like summaries of total steps walked for the month. If an insight is relevant to a specific day it will contain a `target_date` value.

Insights have a priority where `1` is highest and means real-time, `2` is day-level, `3` is week, and `4` is month.

HTML output is provided, however each insight could be assembled from the value fields and knowledge of the insight type, if required.

### Examples

**Thursday marks a new productivity streak!** Beat your goal every day for 3 days

## Averages
```json
{
    "attribute": "floors",
    "date": "2015-05-03",
    "overall": 13,
    "monday": 13,
    "tuesday": 16,
    "wednesday": 13,
    "thursday": 12,
    "friday": 14,
    "saturday": 13,
    "sunday": 13
}
```

Averages are generated weekly and are the basis of our goal system. For attributes that don't warrant a specific related `attribute_name_goal` attribute, the average is used to create the "end value" of the attribute's progress bar in our dashboard — meaning each day, users are being shown their progress relative to their usual behaviour. We break down averages by day of the week but also record the overall average. As we keep historical data this allows us to plot "rolling averages" showing changes in attribute values.

Note: these are actually medians, but we use "average" as it's simpler to explain to users. Please also use this terminology.

# List of attributes

All attributes we currently support. The group an attribute belongs to may change in future, but attribute names should be considered stable.

See [attribute definition](#attributes).

Name                | Group        | Value type
--------------------| ------------ | ----------
`steps`             | Activity     | Integer
`steps_active_min`  | Activity     | Integer
`steps_elevation`   | Activity     | Float (km)
`floors`            | Activity     | Integer
`steps_distance`    | Activity     | Integer (km)
`productive_min`    | Productivity | Duration (minutes as integer)
`neutral_min`       | Productivity | Duration (minutes as integer)
`distracting_min`   | Productivity | Duration (minutes as integer)
`mood`              | Mood         | Integer (between 1 and 5 inclusive)
`mood_note`         | Mood         | String (max 255 characters)
`sleep`             | Sleep        | Duration (minutes as integer)
`time_in_bed`       | Sleep        | Duration (minutes as integer)
`sleep_start`       | Sleep        | Time of day (minutes from midday as integer)
`sleep_end`         | Sleep        | Time of day (minutes from midnight as integer)
`sleep_awakenings`  | Sleep        | Integer
`events`            | Events       | Integer
`events_duration`   | Events       | Duration (minutes as integer)
`weight`            | Health       | Float (kg)
`checkins`          | Location     | Integer
`location`          | Location     | String (`"lat,lng"` format where `lat` and `lng` are floats)
`tracks`            | Media        | Integer
`instagram_posts`   | Social       | Integer
`instagram_comments`| Social       | Integer
`instagram_likes`   | Social       | Integer
`instagram_username`| Social       | String
`tweets`            | Twitter      | Integer
`twitter_mentions`  | Twitter      | Integer
`twitter_username`  | Twitter      | String
`weather_temp_max`  | Weather      | Float (degrees Celsius)
`weather_temp_min`  | Weather      | Float (degrees Celsius)
`weather_precipitation` | Weather  | Float (inches of water per hour)
`weather_cloud_cover`   | Weather  | Float (percentage of sky covered, 0.0 to 1.0)
`weather_wind_speed`    | Weather  | Float (km/hr)
`weather_summary`       | Weather  | String
`weather_icon`          | Weather  | String (name of icon best representing weather values)


# Simple token authentication


All endpoints require authentication. We use a simple token-based scheme which allows a single token per user.
This is available so users can take advantage of their own data — if you're building a client for multiple users,
you want to apply for [OAuth2 client](#oauth2-authentication) credentials.
Make sure this token is included in your requests by including the `Authorization` header with every request.

If you are logged in to Exist in the browser your session-based authentication will also work. This is handy for browsing the API
(assuming you've set up your browser to accept JSON) but shouldn't be relied on for programmatic access.

## Requesting a token 


```shell

curl https://exist.io/api/1/auth/simple-token/ -d username=bobby_tables -d password=existrulz123
```

```python
import requests

requests.post('https://exist.io/api/1/auth/simple-token/',
    {'username':'bobby_tables','password':'existrulz123'})
```

> Returns a token object in JSON:

```json
{ "token": "96524c5ca126d87eb18ee7eff408ca0e71e94737" }
```

Exchange your user credentials for a token. This token will not change or expire by design but may be deprecated as we move to OAuth2 in the future.

### Request

`POST /api/1/auth/simple-token/`

### Parameters

Key      | Example value
-------- | --------
`username` | bobby_tables
`password` | existrulz123

### Response

A JSON object containing a token key.

## Signing requests

Include the `Authorization: Token [your_token]` header in all requests.

> Include the "Authorization: Token xyz" header in all requests.

```python
import requests

requests.post(url,
    headers={'Authorization':'Token 96524c5ca126d87eb18ee7eff408ca0e71e94737'})
```

```shell
# With curl, you can just pass the correct header with each request
curl "api_endpoint_here"
  -H "Authorization: Token 96524c5ca126d87eb18ee7eff408ca0e71e94737"
```

# OAuth2 authentication

**Everybody is welcome to use read-only endpoints and token-based authentication, but you must apply for access to an OAuth2 client for the ability to write data.**
If you'd like to build a client that others can use, [get in touch](mailto:hello@exist.io?Subject=Exist OAuth2 API access) with details of your plans.
We'd love for you to help us get more into and out of Exist.

All endpoints require authentication, except those that are part of the OAuth2 authorisation flow.

OAuth2 clients are superior to the simple-token authentication scheme as they can acquire control of attributes and write values for attributes.

## The OAuth2 authorisation flow

> Send your user to the authorisation page at `https://exist.io/oauth2/authorize`

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
  *  `redirect_uri` with the URI to which Exist returns the user (**must be HTTPS**)
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

Sign all authenticated requests by adding the Authorization header, `Authorization: Bearer [access_token]`. Note that this differs from the simple token-based authentication by using `Bearer`, *not* `Token`.



# Users

## Get current overview for user

```shell
curl -H "Authorization: Token [YOUR_TOKEN]" https://exist.io/api/1/users/\$self/today/
```

```python
import requests

requests.get("https://exist.io/api/1/users/$self/today/",
    headers={'Authorization':'Token [YOUR_TOKEN]'})
```

> Returns a user object in JSON:

```json
{
    "id": 1,
    "username": "josh",
    "first_name": "Josh",
    "last_name": "Sharp",
    "bio": "I made this thing you're using.",
    "url": "http://hellocode.co/",
    "avatar": "https://exist.io/static/media/avatars/josh_2.png",
    "timezone": "Australia/Melbourne",
    "local_time": "2020-07-31T22:33:49.359+10:00",
    "private": false,
    "imperial_units": false,
    "attributes": [
        {
            "group": "activity",
            "label": "Activity",
            "priority": 1, 
            "items": [
                {
                    "attribute": "steps", 
                    "label": "Steps", 
                    "value": 258, 
                    "service": "Fitbit", 
                    "priority": 1, 
                    "private": false,
                    "value_type": 0,
                    "value_type_description": "Integer"
                }, 
                {
                    "attribute": "floors", 
                    "label": "Floors", 
                    "value": 2, 
                    "service": "Fitbit", 
                    "priority": 2, 
                    "private": false,
                    "value_type": 0,
                    "value_type_description": "Integer"
                }
            ]
        }
    ]
}
```

Returns an overview of the user's personal details, and their grouped attributes containing current values.
This is analogous to the "Today" tiles in the dashboard.

When requesting the currently authenticated user, you will receive all attributes.

For **other** **public** users, private attributes will not be returned. For other **private** users,
this will return a user stub.

### Request

`GET /api/1/users/:username/today/`

# Attributes

## Get all attributes

```shell
curl -H "Authorization: Token [YOUR_TOKEN]" https://exist.io/api/1/users/\$self/attributes/
```

```python
import requests

requests.get("https://exist.io/api/1/users/$self/attributes/",
    headers={'Authorization':'Token [YOUR_TOKEN]'})
```

> Returns a list of attribute objects each with a list of values:

```json
[
    {
        "attribute": "steps",
        "label": "Steps",
        "group": "activity",
        "service": "Fitbit",
        "private": false,
        "values": [
            {
                "value": 331,
                "date": "2014-08-01"
            },
            {
                "value": 5872,
                "date": "2014-07-31"
            },
            {
                "value": 2832,
                "date": "2014-07-30"
            },
            {
                "value": 5153,
                "date": "2014-07-29"
            },
            {
                "value": 4354,
                "date": "2014-07-28"
            },
            {
                "value": 7132,
                "date": "2014-07-27"
            },
            {
                "value": 4144,
                "date": "2014-07-26"
            }
        ]
    },
    {
        "attribute": "floors",
        "label": "Floors",
        "group": "activity",
        "service": "Fitbit",
        "private": false,
        "values": [ ]
    }
]
```

Return all of the user's attributes, with the last week's values by default. Currently this method is only allowed for the authenticated user.

### Request

`GET /api/1/users/:username/attributes/`

### Parameters

Name  | Description
------|--------
`limit` | Number of values to return, starting with today. Optional, max is 31.

## Get a specific attribute

```shell
curl -H "Authorization: Token [YOUR_TOKEN]" https://exist.io/api/1/users/\$self/attributes/steps/
```

```python
import requests

requests.get("https://exist.io/api/1/users/$self/attributes/attribute/steps/",
    headers={'Authorization':'Token [YOUR_TOKEN]'})
```

> Returns a list of attribute values, omitting the attribute object itself:

```json
{
    "count": 655,
    "next": "https://exist.io/api/1/users/josh/attributes/steps/?page=2",
    "previous": null,
    "results": [
        {
            "value": 3783,
            "date": "2015-05-08"
        }, {
            "value": 6177,
            "date": "2015-05-07"
        }, {
            "value": 7811,
            "date": "2015-05-06"
        }, {
            "value": 6632,
            "date": "2015-05-05"
        }, {
            "value": 6014,
            "date": "2015-05-04"
        }, {
            "value": 4376,
            "date": "2015-05-03"
        }, {
            "value": 8671,
            "date": "2015-05-02"
        }
    ]
}

```

Returns a paged list of all values from an attribute.

### Request

`GET /api/1/users/:username/attributes/:attribute/`

### Parameters

Name  | Description
------|--------
`limit` | Number of values to return per page. Optional, max is 100.
`page`  | Page index. Optional, default is 1.
`date_min` | Oldest date (inclusive) of results to be returned, in format `YYYY-mm-dd`. Optional.
`date_max` | Most recent date (inclusive) of results to be returned, in format `YYYY-mm-dd`. Optional.


# Insights

## Get all insights

```shell
curl -H "Authorization: Token [YOUR_TOKEN]" https://exist.io/api/1/users/\$self/insights/
```

```python
import requests

requests.get("https://exist.io/api/1/users/$self/insights/",
    headers={'Authorization':'Token [YOUR_TOKEN]'})
```

> Returns a JSON object containing a paged array:

```json
{
    "count": 740, 
    "next": "https://exist.io/api/1/users/josh/insights/?page=2", 
    "previous": null, 
    "results": [
        {
            "created": "2015-05-09T01:00:02Z", 
            "target_date": "2015-05-08", 
            "html": "<div class=\"secondary\">Friday night: Shortest sleep for 3 days</div>...", 
            "value": "303", 
            "value2": "3 days", 
            "comment": null, 
            "type": {
                "name": "sleep_worst_since_x", 
                "period": 1, 
                "priority": 2, 
                "attribute": {
                    "name": "sleep", 
                    "label": "Time asleep", 
                    "group": {
                        "name": "sleep", 
                        "priority": 3
                    }, 
                    "priority": 2
                }
            }
        }, 
        {
            "created": "2015-05-08T21:00:03Z", 
            "target_date": null, 
            "html": "<div class=\"number\">09:38</div>...", 
            "value": "578", 
            "value2": "65 min earlier", 
            "comment": null, 
            "type": {
                "name": "sleep_end_average_week", 
                "period": 7, 
                "priority": 3, 
                "attribute": {
                    "name": "sleep_end", 
                    "label": "Wake time", 
                    "group": {
                        "name": "sleep", 
                        "priority": 3
                    }, 
                    "priority": 4
                }
            }
        }
    ]
}
```

Returns a paged list of user's insights. Only available for the currently authenticated user.

### Request

`GET /api/1/users/:username/insights/`

### Parameters

Name  | Description
------|--------
`limit` | Number of values to return per page, starting with today. Optional, max is 100.
`page`  | Page index. Optional, default is 1.
`date_min` | Oldest date (inclusive) of results to be returned, in format `YYYY-mm-dd`. Optional.
`date_max` | Most recent date (inclusive) of results to be returned, in format `YYYY-mm-dd`. Optional.

## Get all insights for attribute

```shell
curl -H "Authorization: Token [YOUR_TOKEN]" https://exist.io/api/1/users/\$self/insights/attribute/sleep/
```

```python
import requests

requests.get("https://exist.io/api/1/users/$self/insights/attribute/sleep/",
    headers={'Authorization':'Token [YOUR_TOKEN]'})
```

> Returns a JSON object containing a paged array:

```json
{
    "count": 220, 
    "next": "https://exist.io/api/1/users/josh/insights/attribute/sleep/?page=2", 
    "previous": null, 
    "results": [
        {
            "created": "2015-05-09T01:00:02Z", 
            "target_date": "2015-05-08", 
            "html": "<div class=\"secondary\">Friday night: Shortest sleep for 3 days</div>...", 
            "value": "303", 
            "value2": "3 days", 
            "comment": null, 
            "type": {
                "name": "sleep_worst_since_x", 
                "period": 1, 
                "priority": 2, 
                "attribute": {
                    "name": "sleep", 
                    "label": "Time asleep", 
                    "group": {
                        "name": "sleep", 
                        "priority": 3
                    }, 
                    "priority": 2
                }
            }
        } 
    ]
}
```

Returns a paged list of user's insights for a specific attribute. Only available for the currently authenticated user.

### Request

`GET /api/1/users/:username/insights/`

### Parameters

Name  | Description
------|--------
`limit` | Number of values to return per page, starting with today. Optional, max is 100.
`page`  | Page index. Optional, default is 1.
`date_min` | Oldest date (inclusive) of results to be returned, in format `YYYY-mm-dd`. Optional.
`date_max` | Most recent date (inclusive) of results to be returned, in format `YYYY-mm-dd`. Optional.

# Averages

## Get current averages

```shell
curl -H "Authorization: Token [YOUR_TOKEN]" https://exist.io/api/1/users/\$self/averages/
```

```python
import requests

requests.get("https://exist.io/api/1/users/$self/averages/",
    headers={'Authorization':'Token [YOUR_TOKEN]'})
```

> Returns a JSON array:

```json
[
    {
        "attribute": "steps", 
        "date": "2020-04-29", 
        "overall": 4174.0, 
        "monday": 4057.0, 
        "tuesday": 6614.0, 
        "wednesday": 4001.0, 
        "thursday": 3923.0, 
        "friday": 4528.0, 
        "saturday": 3649.0, 
        "sunday": 3904.0
    }, 
    {
        "attribute": "floors", 
        "date": "2020-04-29", 
        "overall": 13.0, 
        "monday": 13.0, 
        "tuesday": 16.0, 
        "wednesday": 14.0, 
        "thursday": 12.0, 
        "friday": 14.0, 
        "saturday": 13.0, 
        "sunday": 12.0
    }
]
```

Returns the most recent average values for each attribute. Only available for the currently authenticated user.

### Request

`GET /api/1/users/:username/averages/`

## Get all averages for attribute

```shell
curl -H "Authorization: Token [YOUR_TOKEN]" https://exist.io/api/1/users/\$self/averages/attribute/steps/
```

```python
import requests

requests.get("https://exist.io/api/1/users/$self/averages/attribute/steps/",
    headers={'Authorization':'Token [YOUR_TOKEN]'})
```

> Returns a JSON object containing a paged array of results:

```json
{
    "count": 11, 
    "next": null, 
    "previous": null, 
    "results": [
        {
            "attribute": "steps", 
            "date": "2020-04-29", 
            "overall": 4174.0, 
            "monday": 4057.0, 
            "tuesday": 6614.0, 
            "wednesday": 4001.0, 
            "thursday": 3923.0, 
            "friday": 4528.0, 
            "saturday": 3649.0, 
            "sunday": 3904.0
        }, 
        {
            "attribute": "steps", 
            "date": "2020-03-30", 
            "overall": 4062.0, 
            "monday": 4057.0, 
            "tuesday": 6618.0, 
            "wednesday": 3610.0, 
            "thursday": 3923.0, 
            "friday": 4063.0, 
            "saturday": 3636.0, 
            "sunday": 3904.0
        }
    ]
}
```

Returns a paged list of average values for an attribute.

### Request

`GET /api/1/users/:username/averages/attribute/:attribute/`

### Parameters

Name  | Description
------|--------
`limit` | Number of values to return per page, starting with today. Optional, max is 100.
`page`  | Page index. Optional, default is 1.
`date_min` | Oldest date (inclusive) of results to be returned, in format `YYYY-mm-dd`. Optional.
`date_max` | Most recent date (inclusive) of results to be returned, in format `YYYY-mm-dd`. Optional.


# Correlations

## Get all correlations for attribute

```shell
curl -H "Authorization: Token [YOUR_TOKEN]" https://exist.io/api/1/users/\$self/correlations/attribute/steps/
```

```python
import requests

requests.get("https://exist.io/api/1/users/$self/correlations/attribute/steps/",
    headers={'Authorization':'Token [YOUR_TOKEN]'})
```

> Returns a JSON object containing an array of results:

```json
{
    "count": 479, 
    "next": "https://exist.io/api/1/users/josh/correlations/attribute/steps/?page=2", 
    "previous": null, 
    "results": [
        {
            "date": "2015-05-11", 
            "period": 90, 
            "attribute": "steps", 
            "attribute2": "steps_distance", 
            "value": 0.999735821732415, 
            "p": 5.43055953485446e-146, 
            "first_person": "I am almost certain (100%) to travel a further distance on days I take more steps.", 
            "second_person": "You are almost certain (100%) to travel a further distance on days you take more steps."
        }, 
        {
            "date": "2015-05-11", 
            "period": 90, 
            "attribute": "steps", 
            "attribute2": "floors", 
            "value": 0.693332839316818, 
            "p": 3.63477333939521e-14, 
            "first_person": "I am much more likely (69%) to climb floors on days I take more steps.", 
            "second_person": "You are much more likely (69%) to climb floors on days you take more steps."
        }
    ]
}
```



Returns a paged list of all correlations generated relating to this attribute, ordered by date.
Correlations may appear more than once, with different results, as this relationship changes over time.


### Request

`GET /api/1/users/:username/correlations/attribute/:attribute/`

### Parameters

Name  | Description
------|--------
`limit` | Number of values to return per page, starting with today. Optional, max is 100.
`page`  | Page index. Optional, default is 1.
`date_min` | Oldest date (inclusive) of results to be returned, in format `YYYY-mm-dd`. Optional.
`date_max` | Most recent date (inclusive) of results to be returned, in format `YYYY-mm-dd`. Optional.
`latest` | Set this to `true` to return only the most recently generated batch of correlations. Use this on its own without `date_min` and `date_max`.

# Attribute ownership

**This section only applies for OAuth2 clients.**

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
`private` | Optional `true` or `false` to change the privacy status of this attribute. Please notify users if you are making previously private attributes public and only do this with good reason.

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

**This section only applies for OAuth2 clients.**

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




# API roadmap

## Upcoming

The following are our short-term priorities:

1. Private beta for OAuth2 full read/write clients with the ability to update attribute data (underway)
2. Public access to OAuth2 clients and the standard authentication flow

Once these are complete, our main ongoing priority will be
extending the list of supported attributes so users can track a wider range of things.

If you'd like to use the full write API, please [email us](mailto:hello@exist.io).


## Changelog

* **2015-07-09:** Introduced attribute validation and started validating `mood` values
* **2015-05-13:** Added date filtering for insights, correlations, averages, and attribute data.


# Questions and feedback

You can always send us an [email](mailto:hello@exist.io) with questions.

Please, give the API a thorough workout and tell us what's inconsistent or missing. Your feedback is going to help us shape a sensible, robust API v1 that we'll collectively be stuck with for a fair while.

### Sign up to the developer list

The easiest way to be kept up to date about API changes and announcements is to join the list.

[Subscribe](https://confirmsubscription.com/h/t/2E3A4057D66FD499)