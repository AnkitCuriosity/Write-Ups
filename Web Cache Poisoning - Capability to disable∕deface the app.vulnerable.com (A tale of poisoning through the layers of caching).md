# Web Cache Poisoning - Capability to disable/deface the app.██████████.com (A tale of poisoning through the layers of caching)


## Description -:

**NOTE** to respect the private policy of the program, the actual vulnerable asset is not disclosed and the same has been referenced as `app.vulnerable.com` wherever necessary.

Earlier I went to `https://app.vulnerable.com` and realized that it is vulnerable to Web Cache Poisoning attack. It simply means, if I send a crafted request to the website, then it's response gets cached by the Cloudfront operating at the front end and the same is served to all the approaching legit users.

## Challenges -:

I've been working around "Web Cache poisoning" issues for quite some time and have submitted tons of such issues to many organizations to help them secure their cache related disasters. On the basis of my understanding of these attacks, I can say that the problem triagers/team would face here is to determine the vulnerability within the testing scope.

This case is the one, where it might be difficult to determine the vulnerability within the testing scope. But on the opposite side it would be easy to implement the attack on a real scope in real time. The reason for this problem is -:

### Challenge #1

The cache-key transformation at `https://app.vulnerable.com` is excluding the entire query string from the cache key. Means, GET parameters are NOT serving as the cache keys, or in more simpler words you cannot use GET parameters (such as `DontPoisonEveryone=true`) as the cache buster.

Means, requesting `https://app.vulnerable.com/?DontPoisonEveryone=true` will always retrieve the cache stored for `https://app.vulnerable.com/` and as a result a potential cache poisoning issue will remain hidden under the surface. On the opposite side, the moment the current cache has expired against `https://app.vulnerable.com/` and if you try to poison against the `https://app.vulnerable.com/?DontPoisonEveryone=true` then it will also poison the `https://app.vulnerable.com/` affecting the genuine users because GET parameters are NOT serving as the cache keys.

### Challenge #2

Not only query string, but this one is among those rare cases where even the path has been excluded from the cache key. Means requesting `https://app.vulnerable.com/anything` (where `anything` is not a static file endpoint) will only result in a `HTTP/1.1 200 OK` with a cache HIT where the response would fetch the cache stored for the `https://app.vulnerable.com/` endpoint.

And so because of this reason path normalization using the characters such as `//` or `%2F` won't help you to acquire a cache MISS. (As documented in "Unkeyed Query Detection" section at `https://portswigger.net/research/web-cache-entanglement`) 

### Challenge #3

No other request header including the `Origin` header is included in the cache key with the exception that `Accept-Encoding` header against a single character length value would help you to hit the back end web server to get a cache MISS but that also only for "once". Means, requesting with `Accept-Encoding: g` (where `g` can be any single character alphabetical or numerical value) would act as the cache buster but afterwards requesting `Accept-Encoding: a` or `Accept-Encoding: bbb` will NOT and would only get you the cache HIT against the `Accept-Encoding: g` response. 

Means, within the testing scope if you only for once initiate the request with `Accept-Encoding: g` cache buster but without the unkeyed header (one that is used for actual poisoning), then you'd end up losing the opportunity because you cannot use `Accept-Encoding: x` any further as long as the cache against `Accept-Encoding: g` expires.

## Reproduction Steps (For testing scenario) -:

Relying on the limited possibility through the Challenge #3, it is possible to poison the cache by initiating a request with `Accept-Encoding: g` header (along with the unkeyed header to trigger the poisoning in the form of a `HTTP/1.1 400 Bad Request` response). Once done, the victim end needs to request the asset with the `Accept-Encoding: g` header at their end too.

Conclusion here is that we poisoned the cache for any user with `Accept-Encoding: g` header value. This small test proved that the cache server is allowed to cache even the 400 responses which is the actual vulnerability.

But obviously this way of testing with "Accept-Encoding" didn't look appealing as far as targeting the real users was intended and this is the exact reason why this report was closed as `N/A` initially -:

![E](https://user-images.githubusercontent.com/58471667/230042836-5e47f0a0-4758-498b-93d7-b5f9ca2cea5e.png)

Now to workaround this, the only thing I was left with was to actually poison the `https://app.vulnerable.com` (which is a "staging" environment not production) using the steps in my next section.

## Reproduction Steps (For real scenario. CAUTION - this will affect the realtime staging users) -:

1) Based on the values of `Age` and `max-age` response headers, an attacker can identify that the life of current cache is for 21600 seconds (6 hours) out of which `Age` seconds are already passed. Wait for the value of `Age` to complete the `max-age` value and instantly initiate the following -:

```
curl --header "Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8" --compressed --header "Accept-Language: en-US,en;q=0.5" --header "Connection: close" --header "DNT: 1" --header "Sec-Fetch-Dest: document" --header "Sec-Fetch-Mode: navigate" --header "Sec-Fetch-Site: none" --header "Sec-Fetch-User: ?1" --header "Upgrade-Insecure-Requests: 1" --user-agent "Mozilla/5.0 (Windows NT 6.2; Win64; x64; rv:95.0) Gecko/20100101 Firefox/95.0" --header "x-amz-server-side-encryption: PoC-by-AnkitSingh-PoC-by-AnkitSingh-PoC-by-AnkitSingh-PoC-by-AnkitSingh-PoC-by-AnkitSingh-PoC-by-AnkitSingh-PoC-by-AnkitSingh-PoC-by-AnkitSingh-PoC-by-AnkitSingh" -v https://app.vulnerable.com/ 
```

2) Now any genuine user visiting at `https://app.vulnerable.com` would see the message `PoC-by-AnkitSingh` instead of the intended `HTTP/1.1 200 OK` response.

After acquiring the permission to actually poison the `https://app.vulnerable.com` in real time and providing a PoC video for the same along with some further guidance to triage team, the state of the report was changed to -:

![5](https://user-images.githubusercontent.com/58471667/222517116-a4726b8a-b38d-489f-895a-ba6206cc9b55.png)

## Supporting Material/Screenshots -:

### This is how the application portal looked before poisoning -:

![before](https://user-images.githubusercontent.com/58471667/222510715-ea8180f7-a759-4405-84d3-ae0b815ba5ca.png)

### After poisoning -:

![after](https://user-images.githubusercontent.com/58471667/222510741-04dd263e-4716-4942-8e42-dd76567ed47e.png)

It's important to note here that it would have been extremely difficult to convince the program/triage team about the potential threat and impact, if the target would have been an application at production.

## References -:

`https://portswigger.net/research/web-cache-entanglement`

**Reward** -:

Severity assigned : P3  
Bounty assigned : $1000

![3](https://user-images.githubusercontent.com/58471667/222513648-fd186d68-bb20-48a4-bd2c-1d5a8abb3b20.png)

*Find me on [@AnkitCuriosity](https://twitter.com/AnkitCuriosity) for more updates!*
