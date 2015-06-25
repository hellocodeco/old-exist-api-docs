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
For now, the API is private and can only be accessed by users. [Sign up](https://exist.io) to give it a try.

This is draft documentation for the official Exist API, currently also in beta. Both the API and
the docs are liable to change at any time, though we'll do our best to document changes here and keep the docs in sync with the API itself.

**This beta, read-only API release is intended as a way for developers to make use of their own data.
Developing and releasing apps which ask other users for their credentials will be frowned upon and tsk-tsked.** However, open source and self-hosted is okay — just be mindful that users should never be asked to trust you with their credentials. 

# Getting started

The API lives at `https://exist.io/api/`. All requests must use HTTPS.

POST bodies can be sent as `application/json`, `application/x-www-form-urlencoded`, or `multipart/form-data`.
However the API will only return JSON. XML is so last decade.

Requests are currently rate-limited at 300/hr per user token. Given user data does not change frequently within
an hourly period (if at all) this should be more than adequate.

Wherever you see the `username` argument in a URL you may substitute the special keyword `$self` to request the authenticated user.

# Object types and terminology

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

In situations where multiple attributes are requested, for example the "today" overview, these attributes will be returned **grouped**.
A group represents attributes that belong together, for example the "activity" group contains the attributes `steps`, `active_min`, etc.
You can see an example of this in the ['overview' endpoint](#get-current-overview-for-user).

Clients should display attributes in these groups when displaying multiple attributes.
Groups are currently fairly broad and may change as we add more supported attributes.

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

# Authentication

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

All endpoints require authentication. We use a simple token-based scheme for now which allows a single token per user.
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

# API roadmap

## Upcoming

The following are our short-term priorities:

1. Private beta for OAuth2 full read/write clients with the ability to update attribute data
2. Public access to OAuth2 clients and the standard authentication flow

Once these are complete, our main ongoing priority will be
extending the list of supported attributes so users can track a wider range of things.

If you'd like to beta-test the full write API, please email us at the address below.


## Changelog

* **2015-05-13:** Added date filtering for insights, correlations, averages, and attribute data.


# Questions and feedback

You can always send us an [email](mailto:hello@exist.io) with questions.

Please, give the API a thorough workout and tell us what's inconsistent or missing. Your feedback is going to help us shape a sensible, robust API v1 that we'll collectively be stuck with for a fair while.

### Sign up to the developer list

The easiest way to be kept up to date about API changes and announcements is to join the list.

[Subscribe](https://confirmsubscription.com/h/t/2E3A4057D66FD499)