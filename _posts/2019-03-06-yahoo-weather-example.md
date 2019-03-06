---
title:  "Fetch Yahoo weather data using Oauth1 authentication in Python 3"
categories: example
mathjax: true
---

Recently Yahoo updated how you can access their weather API. Instead of the
previous free-for-all you now need to have a whitelisted app and authenticate
with Oauth1 when requesting data.

As per the warning in the [documentation](https://developer.yahoo.com/weather/):

> Important EOL Notice: As of Thursday, Jan. 3, 2019, the weather.yahooapis.com
> and query.yahooapis.com for Yahoo Weather API will be retired. 
>
> To continue using our free Yahoo Weather APIs, use 
> https://weather-ydn-yql.media.yahoo.com/forecastrss.
>
> Follow below instructions to get credentials and onboard to this free Yahoo
> Weather API service.

Yahoo have been kind enough to [provide a Python example](https://developer.yahoo.com/weather/documentation.html#oauth-python)
showing how to request this data however it's aimed at Python 2.7

Below I provide an example for Python 3.6+ (might work on previous versions, I've not tested)

## Usage

Functionality is provided via the function `get_yahoo_weather` which takes the parameters:
 - `location`
 - `app_id`
 - `consumer_key`
 - `consumer_secret`

## The code

{% highlight python %}
import time, uuid, urllib
from json import loads
import hmac, hashlib
from base64 import b64encode


def _generate_signature(key, data):
    key_bytes= bytes(key , 'utf-8')
    data_bytes = bytes(data, 'utf-8') 
    signature =  hmac.new(
        key_bytes,
        data_bytes,
        hashlib.sha1
    ).digest()
    return b64encode(signature).decode()


def get_yahoo_weather(
    location,
    app_id,
    consumer_key,
    consumer_secret,
    url='https://weather-ydn-yql.media.yahoo.com/forecastrss',
    format='json'
):
    # Basic info
    method = 'GET'
    app_id = app_id
    consumer_key = consumer_key
    consumer_secret = consumer_secret
    concat = '&'
    query = {
        'location': location
        'format': format
    }
    oauth = {
        'oauth_consumer_key': consumer_key,
        'oauth_nonce': uuid.uuid4().hex,
        'oauth_signature_method': 'HMAC-SHA1',
        'oauth_timestamp': str(int(time.time())),
        'oauth_version': '1.0'
    }

    # Prepare signature string (merge all params and SORT them)
    merged_params = query.copy()
    merged_params.update(oauth)
    sorted_params = [
        k + '=' + urllib.parse.quote(merged_params[k], safe='')
        for k in sorted(merged_params.keys())
    ]
    signature_base_str = (
        method + 
        concat + 
        urllib.parse.quote(
            url,
            safe=''
        ) +
        concat + 
        urllib.parse.quote(concat.join(sorted_params), safe='')
    )

    # Generate signature
    composite_key = urllib.parse.quote(
        consumer_secret,
        safe=''
    ) + concat
    oauth_signature = _generate_signature(
        composite_key,
        signature_base_str
    )

    # Prepare Authorization header
    oauth['oauth_signature'] = oauth_signature
    auth_header = (
        'OAuth ' + 
        ', '.join(
            [
                '{}="{}"'.format(k,v) 
                for k,v in oauth.items()
            ]
        )
    )

    # Send request
    url = url + '?' + urllib.parse.urlencode(query)
    request = urllib.request.Request(url)
    request.add_header('Authorization', auth_header)
    request.add_header('X-Yahoo-App-Id', app_id)
    response = urllib.request.urlopen(request).read()
    return loads(response)

{% endhighlight %}
