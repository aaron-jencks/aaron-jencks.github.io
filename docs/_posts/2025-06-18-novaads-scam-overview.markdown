---
layout: post
title:  "NovaAds Scam Overview"
date:   2025-06-18 18:01:00 -0400
categories: jekyll update
---
<img src="/assets/images/novaads-logo.png" alt="NovaAds Logo" style="height: 200px">

# Scam Overview

Recently I was contacted by a person via WhatsApp about a job opportunity. They indicated that it was an online job opportunity that pays between $100 and $400 a day and offers flexible hours. They claimed that they represent a digital marketing company called NovaAds. They make money by boosting the sales of small businesses by falsifying ratings and reviews (this is a red flag). They explain that last part in much more vague detail, but at the end of the day, it's what they're doing.

The salary structure is a bit confusing, but basically the more days in a row that you complete:

<img src="/assets/images/novaads-salary-structure.jpeg" alt="NovaAds Salary Structure" style="float: left; margin: 0 10px 10px 0; width:300px;">

In addition to this you would also make a commission based on the value of the product that you review. Typically it's only a couple of dollars. The way that the job works is that they have a website that you go to along with a page that you click on to receive products to review. The next red flag comes in the fact that you have to pony up your own money in order to review a product. What this means, is that say you have a radio that would cost $25 on Amazon, to review it, you'd have to pay $25 from your account on the website and then when the review goes through you'll get your money back along with a commission (this is another red flag).

Typically they offer you tasks in batches of 45, you do two batches a day. In each batch there can be 0-3 of these special "combination" tasks. This is the hook of the scam. Whenever you click on the button to do a task, the amount of the product is deducted from your account without checking the balance first and if you don't have the money to pay it, then your account is frozen until you deposit more money into a crypo wallet (another red flag) to replenish it (another red flag). A "combination" task basically guarantees that your account will be frozen by selecting an item that costs hundreds of dollars and then automatically assigns multiple tasks to you at once (hence the term combination). These tasks then come with the added bonus of having an increased commission.

It can be assumed that once you get a comination tasks that you'll never see that money again. Also there's a little niche in the terms of service that indicates that to withdrawal more than $100k from your account you need to first deposit 10% of the value. I'm sure that there's also no guarantee that if you withdraw any money that you'll actually receive it. Anyway, if you want, here's a full description of the scam in their [Terms and Conditions](https://www.novaads.life/#/tac)

# API

Since I was fairly certain this was a scam and had no intention of actually doing any of the work, I was curious to see how their website worked in the backend. I managed to automate user generation and logging in before they caught on to me and revoked my invite codes and deleted all of my users. Here's what I learned during my tinkering:

The base URL for all of their API calls is `https://api.bloomingdales1.cfd`. All of the requests take the form of `{base}/api/{path}?lang={lang_code}`. `lang_code` is almost always `en-us`. 

### Headers

Each `POST` request requires certain headers. Namely an `Authorization` and a `Sign` field. I'll cover how to retrieve those now:

#### Authorization

The authorization field is a unique key assigned to each account, it's generated upon registration, when registering a new user, and when logging in, it's a hardcoded value:
```
eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJvcmRlckFkbWluIiwiYXVkIjoiYXNlIiwiaWF0IjoxNzUwMDA5NDA2LCJleHAiOjE3NTI2MDE0MDYsImRhdGEiOnsidXNlcl9pZCI6MTA1Miwib25seUNvZGUiOiI2ODRmMDYzZWFiMzM3In19.dDhVCf8TdTyKqBxNGACjCH_AgOPhg_93Zp4g9s3Fu94
```
Otherwise you can find it by pinging the `index` endpoint and extracting it.

#### Sign

This is the interesting one, this is an MD5 hash of a certain struct this struct looks like the following:
```json
{
  "domain": "https://api.bloomingdales1.cfd",
  "user_id": "your user id, or 1052 when registering/logging in",
  "timestamp": "epoch timestamp",
  "lang": "en-us",
  "secret": "f2qRGEOps0gms562N7V60B1SVJyrmNVDCsaddGS",
  // the rest of the payload
}
```
Where:
* `user_id`: is the database id assigned to each user, for registering a user, or logging in, you use 1052
* `timestamp`: the seconds since epoch of the request
* `// the rest of the payload`: the rest of the payload contents

To compute the `Sign` field you then generate a query string, in sorted alphabetical order, using all of the keys and values, excluding any with empty values. It should fit something using this: `k_1=v_1&k_2=v_2&...&k_n=v_n`. In python the code for this looks something like this:
```python
def generate_sign(data: dict, user_id: str, timestamp: int) -> str:
    payload = {
        "domain": DOMAIN,
        "user_id": user_id,
        "timestamp": timestamp,
        "lang": LANG,
        "secret": SECRET,
        **data
    }
    sorted_str = "&".join(
        f"{k}={v}" for k, v in sorted(payload.items()) if v is not None and v != ""
    )
    return hashlib.md5(sorted_str.encode()).hexdigest()
```

#### All Together

This is the code I used to post to an arbitrary endpoint in the novaads website:

```python
API_BASE = "https://api.bloomingdales1.cfd"
SECRET = "f2qRGEOps0gms562N7V60B1SVJyrmNVDCsaddGS"
LANG = "en-us"
DOMAIN = API_BASE
HEADERS = {
    "Content-Type": "application/json;charset=utf-8",
    "User-Agent": "Mozilla/5.0"
}


def generate_sign(data: dict, user_id: str, timestamp: int) -> str:
    payload = {
        "domain": DOMAIN,
        "user_id": user_id,
        "timestamp": timestamp,
        "lang": LANG,
        "secret": SECRET,
        **data
    }
    sorted_str = "&".join(
        f"{k}={v}" for k, v in sorted(payload.items()) if v is not None and v != ""
    )
    return hashlib.md5(sorted_str.encode()).hexdigest()


def post_payload(path: str, payload: dict):
    url = f"{API_BASE}/api/{path}?lang={LANG}"
    timestamp = int(time.time() // 10)
    # print(payload)
    sign = generate_sign(payload, user_id=1052, timestamp=timestamp)

    headers = {
        **HEADERS,
        "Authorization": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJvcmRlckFkbWluIiwiYXVkIjoiYXNlIiwiaWF0IjoxNzUwMDA5NDA2LCJleHAiOjE3NTI2MDE0MDYsImRhdGEiOnsidXNlcl9pZCI6MTA1Miwib25seUNvZGUiOiI2ODRmMDYzZWFiMzM3In19.dDhVCf8TdTyKqBxNGACjCH_AgOPhg_93Zp4g9s3Fu94",
        "Sign": sign,
        "Timestamp": str(timestamp)
    }

    return requests.post(url, json=payload, headers=headers)
```

#### Handling Responses

This is interesting. The site uses 2 different hosting services, the site goes through a relay, and also hosts on itself. This means something interesting. The relay actually responds with its own HTTP codes. So, even if the response code is non erroneous, the post may not have been accepted. There is a `code` in the json of the response as well. That said sometimes the response is garbled up with additional information. Sometimes I would see responses that looked something like `Error: operation timed out after 2000ms...{"code": 200, ...}`. So to recap, you have to be able to check for errors from both the relay, and the server. Sometimes the response will be json and sometimes it won't, sometimes the outer response code will be 200 and the actual request failed, the error handling is a bit complex.

#### Rate Limits

As far as I can tell the rate limit is 1.12-1.20 seconds/request. I've found this experimentally. The system doesn't use request quotas or anything. The server initially let me send requests in the 10s-100s of requests/second, but once they detected that I was going too fast they throttled the requests to 1.12 seconds/request. Your experience might vary.

### Registering a User

To register a user is pretty simple, but I want to cover the constraints (or lack thereof) of the field lengths and contents. To register a user you must construct a dictionary of the following:
```json
{
  "username": "...",
  "password": "...",
  "phone": "...",
  "pay_password": "...",
  "repeat_password": "...",
  "parent_invite_code": "...",
  "email": "...",
  "email_code": "...",
  "sex": "..."
}
```
**There's a special note here:** They modified the data rules while I was "tinkering" with their site, and there is a total length rule on the size of a single user object. I don't have an exact number, but it appears to be somewhere around 640,363 bytes. There are a couple of fields that clearly don't have length constraints on them. I found that if you restrict them to 160,000 characters then you can maximize the rest of the fields and won't go over the total length limit. For convention's sake I'm going to mark the fields as not having a limit, but you can refer back to this note later.

Lets go over what each field must contain:
* `username`: no character restrictions, but cannot be longer than 25 characters
* `password`: no character restrictions, no length restrictions. On the website they restrict the length of this field to 18 characters, but that constraint is only enforced by the client and not the server.
* `phone`: suprisingly no character restrictions, and can also be a length of up to 255 characters
* `pay_password`: no character restrictions, no length restrictions
* `repeat_password`: I assumed it must be equal to `password`, but it might not, further analysis is needed. Has same restrictions as `password`
* `parent_invite_code`: **THIS ONE IS IMPORTANT** It needs to exist in the system, you can't fake it. But unlike the Terms of Service you can use any user, even one that you just registered.
* `email`: no cannot contain punctuation, I left out numbers too as an extra measure, number of characters before the `@` cannot exceed 64 characters. I just used `@example.com` so you might be able to expand this part to be longer. This field is `""` by default because you don't have to supply an email to sign up.
* `email_code`: no character restrictions, no length restrictions. This field is `""` by default because you don't have to supply an email when you sign up.
* `sex`: must be either `1` or `2`

To register the user you simply `POST` to the `register` endpoint and follow the guide above.

### Logging In

Logging in is actually super simple. I used this endpoint to test and find the length constraints of the fields in the registration process. It contains a payload of the following:
```json
{
  "username": "...",
  "password": "..."
}
```
You just `POST` this payload to the `login` endpoint and follow the guide above.

# Conclusions

This is highly suspicious as a scam, there are several red flags in the overview of the structure of the system. Also several typos throughout the website (another red flag). But I'd avoid putting any money into this and keep your distance. If you do end up getting an invite code, go ahead and send it to me so that I can continue playing around :)
