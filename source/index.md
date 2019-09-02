---
title: Exist API reference

language_tabs:
  - shell
  - python

toc_footers:
  - <a href="https://exist.io">Sign up for an account</a>
  - <a href="https://confirmsubscription.com/h/t/2E3A4057D66FD499">Developer mailing list</a>
  - <a href="https://github.com/hellocodeco/exist-api-docs">Contribute on Github</a>

includes:
  

search: true
---

# Introduction


Hello! So you want to take your Exist data and do interesting things with it. Great!
First things first, though: the API can only be accessed by user accounts. [Sign up](https://exist.io) if you don't yet have an account.
Once you do, great! You can access all features of the API including writing data to attributes.

If you're building something on the Exist API we encourage you to [sign up to the API mailing list](https://confirmsubscription.com/h/t/2E3A4057D66FD499)
so we can update you on progress (typically we only email a few times a year).

## Getting started

The API lives at `https://exist.io/api/1/`. All requests must use HTTPS.

POST bodies can be sent as `application/json`, `application/x-www-form-urlencoded`, or `multipart/form-data`.
However the API will only return JSON. XML is so last decade.

Requests are currently rate-limited at 300/hr per user token. Given user data does not change frequently within
an hourly period (if at all) this should be more than adequate.

Wherever you see the `username` argument in a URL you may substitute the special keyword `$self` to request the authenticated user.

OAuth2 is the main means of authorising requests to the API, but if you're building a personal read-only client you might like to use simple token auth.
[Read the authentication overview](#authentication-overview) to see which one you need.

To get stuck in and retrieve some personal data, you should start by [creating a new client app](https://exist.io/account/apps/),
then [get today's overview](#get-current-overview-for-user).

## Important values

Name | Value
-----------------| ------
**API base URL**     | `https://exist.io/api/1/`
**OAuth2 base URL**  | `https://exist.io/oauth2/`
**Response type**| `application/json`
**Rate-limiting**| 300 requests/hr per user token
**POSTing data** | `application/x-www-form-urlencoded` (the usual) or send the body as `application/json`
**OAuth2 auth header** | `Authorization: Bearer [tokenxyz]`
**Simple token auth header** | `Authorization: Token [tokenabc]`
**Getting a simple token** | `POST  'username' and 'password' to https://exist.io/api/1/auth/simple-token/`
**Testing your token** | `GET https://exist.io/api/1/users/$self/today/`
**See some JSON in your browser** | [https://exist.io/api/1/users/$self/today/](https://exist.io/api/1/users/$self/today/)

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

## Clients and services

We use **client** to refer an application with OAuth2 client credentials. A client which writes data to attributes is termed a **service**. 


## List of attributes

All attributes we currently support. The group an attribute belongs to may change in future, but attribute names should be considered stable.

Remember all data should be sent, and is stored and returned, in metric units. Imperial conversion should occur when rendering as required.

See [attribute definition](#attributes).

Name                | Group        | Value type description         | Value type 
--------------------| ------------ | -------------------------------|-----------
`steps`             | Activity     | Integer                        | `0`
`steps_active_min`  | Activity     | Period (minutes as integer)    | `3`
`steps_elevation`   | Activity     | Float (km)                     | `1`
`floors`            | Activity     | Integer                        | `0`
`steps_distance`    | Activity     | Float (km)                     | `1`
`cycle_min`         | Activity     | Period (minutes as integer)    | `3`
`cycle_distance`    | Activity     | Float (km)                     | `1`
`active_energy` | Activity | Float (kJ) | `1`
`workouts`          | Workouts     | Integer                        | `0`
`workouts_min`      | Workouts     | Period (minutes as integer)    | `3`
`workouts_distance` | Workouts     | Float (km)                     | `1`
`productive_min`    | Productivity | Period (minutes as integer)    | `3`
`neutral_min`       | Productivity | Period (minutes as integer)    | `3`
`distracting_min`   | Productivity | Period (minutes as integer)    | `3`
`commits`           | Productivity | Integer                        | `0`
`tasks_completed`   | Productivity | Integer                        | `0`
`words_written`     | Productivity | Integer                        | `0`
`emails_sent`       | Productivity | Integer                        | `0`
`emails_received`   | Productivity | Integer                        | `0`
`pomodoros_min` | Productivity | Period (minutes as integer) | `3`
`keystrokes` | Productivity | Period (minutes as integer) | `3`
`custom`            | Custom tracking | String                      | `2`
`coffees`           | Food and drink | Integer                      | `0`
`alcoholic_drinks`  | Food and drink | Integer                      | `0`
`energy`            | Food and drink | Float (kJ)                   | `1`
`water`             | Food and drink | Integer (ml)                 | `0`
`carbohydrates`     | Food and drink | Float (g) | `1`
`fat`               | Food and drink | Float (g) | `1`
`fibre`             | Food and drink | Float (g) | `1`
`protein`           | Food and drink | Float (g) | `1`
`sugar`             | Food and drink | Float (g) | `1`
`sodium`            | Food and drink | Float (mg) | `1`
`cholesterol`       | Food and drink | Float (mg) | `1`
`caffeine`          | Food and drink | Float (mg) | `1`
`money_spent`       | Finance      | Float (user's local currency unit) | `1`
`mood`              | Mood         | Integer (between 1 and 5 inclusive)    | `0`
`mood_note`         | Mood         | String (max 255 characters)    | `2`
`sleep`             | Sleep        | Period (minutes as integer)    | `3`
`time_in_bed`       | Sleep        | Period (minutes as integer)    | `3`
`sleep_start`       | Sleep        | Time of day (minutes from midday as integer)   | `6`
`sleep_end`         | Sleep        | Time of day (minutes from midnight as integer) | `4`
`sleep_awakenings`  | Sleep        | Integer                        | `0`
`events`            | Events       | Integer                        | `0`
`events_duration`   | Events       | Period (minutes as integer)    | `3`
`weight`            | Health       | Float (kg)                     | `1`
`body_fat`          | Health       | Float (percentage, 0.0 to 1.0) | `5`
`lean_mass` | Health | Float (kg) | `1`
`heartrate`         | Health       | Integer                        | `0`
`heartrate_max` | Health | Integer | `0`
`heartrate_resting` | Health | Integer | `0`
`meditation_min`    | Health       | Period (minutes as integer)    | `3`
`menstrual_flow` | Health | Integer (`0`=none, `1`=spotting, `2`=light, `3`=medium, `4`=heavy) | `0`
`sexual_activity` | Health | Integer | `0`
`checkins`          | Location     | Integer                        | `0`
`location`          | Location     | String (`"lat,lng"` format where `lat` and `lng` are floats) | `2`
`tracks`            | Media        | Integer                        | `0`
`articles_read`     | Media        | Integer                        | `0`
`pages_read`     | Media        | Integer                        | `0`
`mobile_screen_min`     | Media        | Period (minutes as integer)                        | `3`
`gaming_min`     | Media        | Period (minutes as integer)                        | `3`
`tv_min`     | Media        | Period (minutes as integer)                        | `3`
`instagram_posts`   | Social       | Integer                        | `0`
`instagram_comments`| Social       | Integer                        | `0`
`instagram_likes`   | Social       | Integer                        | `0`
`instagram_username`| Social       | String                         | `2`
`facebook_posts`    | Social       | Integer                        | `0`
`facebook_comments` | Social       | Integer                        | `0`
`facebook_reactions`| Social       | Integer                        | `0`
`tweets`            | Twitter      | Integer                        | `0`
`twitter_mentions`  | Twitter      | Integer                        | `0`
`twitter_username`  | Twitter      | String                         | `2`
`weather_temp_max`  | Weather      | Float (degrees Celsius)        | `1`
`weather_temp_min`  | Weather      | Float (degrees Celsius)         | `1`
`weather_precipitation` | Weather  | Float (inches of water per hour) | `1`
`weather_cloud_cover`   | Weather  | Float (percentage of sky covered, 0.0 to 1.0) | `5`
`weather_wind_speed`    | Weather  | Float (km/hr)                  | `1`
`weather_summary`       | Weather  | String                         | `2`
`weather_icon`          | Weather  | String (name of icon best representing weather values) | `2`
`sunrise`               | Weather  | Time of day (minutes from midnight as integer) | `4`
`sunset`                | Weather  | Time of day (minutes from midday as integer) | `6`

## Custom tags

> An example of the today endpoint's output for the custom group.

```json

{

    "group": "custom",
    "label": "Custom tracking",
    "priority": 3,
    "items": [
        {
            "attribute": "custom",
            "label": "Custom tracking",
            "value": "accounting,",
            "service": "exist_for_android",
            "priority": 1,
            "private": true,
            "value_type": 2,
            "value_type_description": "String"
        },
        {
            "attribute": "accounting",
            "label": "Accounting",
            "value": 1,
            "service": "builtin",
            "priority": 2,
            "private": true,
            "value_type": 0,
            "value_type_description": "Integer"
        },
        {
            "attribute": "tired",
            "label": "Tired",
            "value": 0,
            "service": "builtin",
            "priority": 2,
            "private": true,
            "value_type": 0,
            "value_type_description": "Integer"
        },
        
    ]
}

```

Custom tags are represented as integer type attributes within the `custom` group, and are user-defined so unable to be listed here. Examples include tags like `meditation`, `tired`, and `sex`. 

The `custom` attribute is a string representation of the tags a user has sent, but may be truncated to the last tag to fit within 250 characters. Using this string might be helpful to you as a quick and easy way to display tags, so feel free to use it if it suits your purposes, but is not the source of truth. To correctly get all tags for a day, collect all attribute names for attributes with a priority of 2 or higher, within the `custom` group, with a value of `1`. A value of `1` represents the "tagged" state, `0` meaning the tag was not used for that day.  

Tag names are stored with spaces converted to underscores and should always be rendered with underscores converted back to spaces, to be more user-friendly.


# Authentication overview

There are two authentication methods — simple tokens, and OAuth2 clients. So which one do you need?

**Simple token authentication** is read-only and exists as a basic means for users to access their own data from Exist. This method is available to everyone
by exchanging a username and password for a token that doesn't expire. This is only recommended for quickly building a single-user, read-only client,
and may be deprecated in future.

**OAuth2 clients** are superior to simple-token authentication as they can acquire control of attributes and write values for attributes.
One client application can create and use access tokens for many users. One user may have many clients authorised to access their Exist account, each with a separate token that can be revoked.

OAuth2 clients are now available to all users. You can create one from your [app management page](https://exist.io/account/apps/) within your Exist account.

# Simple token authentication

All endpoints require authentication. We use a simple token-based scheme which allows a single token per user.
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

All endpoints require authentication, except those that are part of the OAuth2 authorisation flow.
Make sure your access token is included in your requests by including the `Authorization: Bearer` header with every request.

You may mix and match OAuth2 authentication with simple token or even session-based authentication as you test the API. API endpoints will respond with JSON within your browser
if you are logged in to the site.

## Authorisation flow

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

## Get profile stub for user

```shell
curl -H "Authorization: Token [YOUR_TOKEN]" https://exist.io/api/1/users/\$self/profile/
```

```python
import requests

requests.get("https://exist.io/api/1/users/$self/profile/",
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
    "imperial_distance": false,
    "imperial_weight": false,
    "imperial_energy": false,
    "imperial_liquid": false,
    "imperial_temperature": false,
    "attributes": [ ]
        }
    ]
}
```

Returns an overview of the user's personal details.

### Request

`GET /api/1/users/:username/profile/`


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
    "imperial_distance": false,
    "imperial_weight": false,
    "imperial_energy": false,
    "imperial_liquid": false,
    "imperial_temperature": false,
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

When requesting the currently authenticated user, you will receive all attributes. For other users,
this will return a user stub.

### Request

`GET /api/1/users/:username/today/`

# Attributes

## Get multiple attributes

```shell
curl -H "Authorization: Token [YOUR_TOKEN]" https://exist.io/api/1/users/\$self/attributes/
```

```python
import requests

requests.get("https://exist.io/api/1/users/$self/attributes/",
    headers={'Authorization':'Token [YOUR_TOKEN]'})
```

> Returns a list of attribute objects each with a list of values, by default all attributes:

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

Return the user's attributes, all attributes with the last week's values by default. Currently this method is only allowed for the authenticated user.

If you need to get a specific date range or combination of attribute values, you may combine the optional parameters to do so.

### Request

`GET /api/1/users/:username/attributes/`

### Parameters

Name  | Description
------|--------
`limit` | Number of values to return, starting with today. Optional, max is 31.
`attributes` | Optional comma-separated list of attributes, eg. `mood,mood_note`
`date_min`   | Optional date specifying the oldest value that can be returned, in format `YYYY-mm-dd`.
`date_max`   | Optional date specifying the newest value that can be returned, in format `YYYY-mm-dd`.

## Get a specific attribute

```shell
curl -H "Authorization: Token [YOUR_TOKEN]" https://exist.io/api/1/users/\$self/attributes/steps/
```

```python
import requests

requests.get("https://exist.io/api/1/users/$self/attributes/steps/",
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
            "text": "Friday night: Shortest sleep for 3 days\r\n", 
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
            "text": "09:38 average time asleep...",
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
            "text": "Friday night: Shortest sleep for 3 days...",
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

`GET /api/1/users/:username/insights/attribute/:attribute/`

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

## Get the strongest correlations for all attributes

```shell
curl -H "Authorization: Token [YOUR_TOKEN]" https://exist.io/api/1/users/\$self/correlations/strongest/
```

```python
import requests

requests.get("https://exist.io/api/1/users/$self/correlations/strongest/",
    headers={'Authorization':'Token [YOUR_TOKEN]'})
```

> Returns a JSON array:

```json
[
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



Returns a list of the user's strongest correlations across all attributes.


### Request

`GET /api/1/users/:username/correlations/strongest/`


# Attribute ownership

**This section only applies for OAuth2 clients.**

Only one service may have ownership of any user attribute at any given time. Services must acquire ownership of an attribute to be able to write data for this attribute, and can release ownership if needed, for example if the user closes their account with this service or chooses to turn off certain attributes.

The process of acquiring an attribute only needs to happen once, not each time the attribute is updated, although you may get an error if you attempt to update an attribute you don't own.

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

## Overview

If you jumped straight here looking for how to send data for your own attributes, here's a recap of the steps you may have missed:

1. Make sure you have an [OAuth2 client](#authentication-overview) set up
2. Make sure you've listed all attributes you'd like to be able to write to (set this from your [app management page](https://exist.io/account/apps/))
3. Make sure you have [acquired ownership](#attribute-ownership) of the attribute before updating (just once, not each time)
4. Now you can update.

## Updating attribute values

```shell
curl https://exist.io/api/1/attributes/update/ -H "Content-Type: application/json" -H "Authorization: Bearer 96524c5ca126d87eb18ee7eff408ca0e71e94737" -X POST -d '[{"name":"mood", "date":"2015-05-20", "value":5}, {"name":"mood_note", "date":"2015-05-20", "value":"Great day playing with the Exist API"}]'
```

```python
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

Valid values are described by the attribute's `value_type` and `value_type_description` fields. However, values are only validated broadly by type and so care must be taken to send correct data. Do not rely on Exist to validate your values beyond
enforcing the correct type.

Check value types for each attribute in [list of supported attributes](#list-of-attributes).

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


# Updating custom tags

There are two valid approaches here for sending tags:
 
- managing all tags by acquiring the `custom` attribute,
- using the `append` scope to write individual tags for a day, without being able to modify other tags.

Both are documented below.

## Validating tag names

```python
import re

def validate_tag(raw_tag):
    # for python 3: remove the 'u'
    regex = re.compile(ur'[\W]', re.UNICODE)
    
    tag = raw_tag.replace(' ', '_')
    tag = regex.sub('', tag)
    tag = tag.strip('_')

    if len(tag) == 0:
        raise Exception("Tag '%s' contains too many invalid characters" % raw_tag)

    return tag
```

Tags must be "slugs" that can contain only valid unicode letters, numbers, and underscores. Convert spaces to underscores, remove invalid characters, and make sure to trim leading and trailing whitespace. An example validation method is provided here in Python.   

## Acquiring and managing all tags

If you wish to manage all tags for a user, which you might do if you were writing an alternative "tag client" app, for example, you can handle this similarly to updating any other attribute. In this case, you acquire and update the `custom` attribute with a string of comma-separated tags. 

This is a short-hand means for updating each tag attribute to have a value of `1` for the day. For example, sending a string value of `tired, sex` means these tags will have a value of `1` set, and all others will be set to `0`. The zeroing of other tags gives users an easy way to remove tags for a day, but this means care must be taken to not overwrite tags sent via the [append](#appending-specific-tags) method.

### To get existing tags for a day

If you are managing all tags and don't want to overwrite existing tags, you should:

1. Get a list of all custom tag attributes
2. Get the names of all attributes with a value of `1` for the day
3. Concatenate these into one string, joined by commas
4. Use this string as the starting value for your updates


### To update tags

1. Make sure your client app has permission to acquire the `custom` attribute (set this from your [app management page](https://exist.io/account/apps/))
2. Make sure you have [acquired ownership](#attribute-ownership) of the `custom` attribute before updating (just once, not each time)
3. Send a string value for `custom` containing a list of tags.  Tags must be sent as a comma-separated list of [validated tag names](#validating-tag-names). 

To read and present tags, you can read about [custom tags values](#custom-tags).


```shell
curl https://exist.io/api/1/attributes/update/ -H "Content-Type: application/json" -H "Authorization: Bearer 96524c5ca126d87eb18ee7eff408ca0e71e94737" -X POST -d '[{"name":"custom", "date":"2015-05-20", "value":"tired, bike_ride, vitamin_d"}]'
```

```python
import requests, json

url = 'https://exist.io/api/1/attributes/update/'

attributes = [{"name":"custom", "date":"2015-05-20", "value":"tired, bike_ride, vitamin_d"}]

response = requests.post(url, headers={'Authorization':'Bearer 96524c5ca126d87eb18ee7eff408ca0e71e94737'},
    data=json.dumps(attributes))
```

> Returns a JSON object containing successful and failed updates:

```json
{ "success": [ 
    { "name":"custom",
      "date":"2015-05-20",
      "value":"tired, bike_ride, vitamin_d"
    }
  ],
  "failed": []
}
```

### Request

`POST /api/1/attributes/update/`

### Parameters

Clients must send a JSON-encoded array of objects containing a `name`, `date`, and `value`.

Name  | Description
------|--------
`name` | `custom`
`date` | String of format `YYYY-mm-dd`
`value` | A string of comma- (and optional space-) separated tags in the form `bike_ride, meditation, sex` 


### Response

Returns `200 OK` if all attributes were processed successfully, or `202 Accepted` if some attributes failed. The content is a JSON object containing `success` and `failed` arrays, where each item in the array is an attribute sent in the prior request. Failed attributes get `error` and `error_code` fields added. 


## Appending specific tags

This alternative method allows you to append tags to a day without ownership of the `custom` attribute. By using the `append` endpoint, you can send a list of tags, with optional dates, and have these tags added to the user's other tag selections for that day. This works well for event-based use cases, where a particular event might trigger a particular tag. You can easily send a single tag name, without date, to add it to the current day in the user's time zone.

To use this endpoint, your client needs to have asked for the `append` permission with the current user. When following [the OAuth2 authentication process](#authorisation-flow), substitute `scope=append`, or `scope=read+write+append` if you also need write access.  


```shell
curl https://exist.io/api/1/attributes/custom/append/ -H "Content-Type: application/json" -H "Authorization: Bearer 96524c5ca126d87eb18ee7eff408ca0e71e94737" -X POST -d '[{"value":"bike_ride", "date":"2017-05-20"}]'
```

```python
import requests, json

url = 'https://exist.io/api/1/attributes/custom/append/'

tags = [{"value":"bike_ride", "date":"2017-05-20"}]
# alternatively you could send a single value for today
tags = {"value":"bike_ride"}

response = requests.post(url, headers={'Authorization':'Bearer 96524c5ca126d87eb18ee7eff408ca0e71e94737'},
    data=json.dumps(tags))
```

> Returns a JSON object containing successful and failed updates:

```json
{ "success": [ 
    { "date":"2015-05-20",
      "value":"bike_ride"
    }
  ],
  "failed": []
}
```

### Request

`POST /api/1/attributes/custom/append/`

### Parameters

Clients must send either:

- a JSON-encoded array of objects containing a `value` and optional `date`
- a single JSON object with a `value` and optional `date`

Name  | Description
------|--------
`value` | The [validated name](#validating-tags) of the tag to append
`date` | Optional string of format `YYYY-mm-dd`. If omitted, current day is assumed
 

### Response

Returns `200 OK` if all tags were processed successfully, or `202 Accepted` if some attributes failed. The content is a JSON object containing `success` and `failed` arrays, where each item in the array is an attribute sent in the prior request. Failed attributes get `error` and `error_code` fields added. 


# API roadmap

## Changelog

* **2017-09-27:** Custom tags now available from the API
* **2015-12-01:** OAuth2 clients are available for all users
* **2015-10-05:** Removed separate value fields in insights, added rendered text field
* **2015-07-09:** Introduced attribute validation and started validating `mood` values
* **2015-05-13:** Added date filtering for insights, correlations, averages, and attribute data.


# Questions and feedback

You can always send us an [email](mailto:hello@exist.io) with questions.

Please, give the API a thorough workout and tell us what's inconsistent or missing. Your feedback is going to help us shape a sensible, robust API v1 that we'll collectively be stuck with for a fair while.

### Sign up to the developer list

The easiest way to be kept up to date about API changes and announcements is to join the list.

[Subscribe](https://confirmsubscription.com/h/t/2E3A4057D66FD499)
