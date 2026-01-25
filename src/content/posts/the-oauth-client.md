---
title: "Part 2- The OAuth Client"
published: 2025-07-25
draft: true
description:
  "Learning about the OAuth client from the authorization grant type"
tags: ["OAuth"]
series: "OAuth Simplified"
---

## Introduction

In the previous blog post, we walked through the
[Authorization Code grant type](/posts/introduction-to-oauth#enhancing-security).
We will continue using the example of Strava connecting to your Facebook
account on your behalf in order to make a post.

In this blog post, we will dive deeper into the OAuth Client component and
discuss various concepts around it. Once we have covered the foundations,
we will discuss why it is important for the client to use the state
parameter. Here is an animation of a simulated CSRF attack on an OAuth
client that does not take advantage of the state parameter:

<!-- markdownlint-disable -->
<!-- prettier-ignore-start -->
:::attackerDuck
I will be the attacker! heheee!
:::
<!-- markdownlint-restore -->
<!-- prettier-ignore-end -->
<iframe
  src="https://player.mux.com/8XUb76hIAdOqfWL1S61zVEH5a6n100TTaaYwhHMtQP4A?metadata-video-title=CSRF&video-title=CSRF"
  style="width: 100%; border: none; aspect-ratio: 1/1;"
  allow="accelerometer; gyroscope; autoplay; encrypted-media; picture-in-picture;"
  allowfullscreen
></iframe>

<!-- markdownlint-disable -->
<!-- prettier-ignore-start -->
:::me
Don't worry if the animation looks confusing. We will build ourselves up to
understand it just like the [previous blog post](/posts/introduction-to-oauth#oauth-20-definition), and also learn ways to stop this kind of attack using the state parameter!
:::
<!-- markdownlint-restore -->
<!-- prettier-ignore-end -->

Let's start discussing various concepts that are used in relation to the
OAuth Client.

## Redirection

Here is the definition of a
[URL redirection](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Redirections):

"URL redirection, also known as URL forwarding, is a technique to give more
than one URL address to a page, a form, a whole website, or a web
application. HTTP has a special kind of response, called an HTTP redirect,
for this operation."

<!-- markdownlint-disable -->
<!-- prettier-ignore-start -->
:::confusedDuck
What does it mean to give more than one URL?
:::
<!-- markdownlint-restore -->
<!-- prettier-ignore-end -->

In the previous blog post, we saw that the user is
[redirected to the Authorization Server](/posts/introduction-to-oauth#/#delegating-access)
when initially accessing the client. This is possible because the client
provided more than one URL. On top of providing the client's own URL, the
client also provides the URL of the authorization server to forward the
user to delegate authorization. So when the user initially clicks on the
link of the client, the user gets sent (redirected) to the Authorization
Server.

## Back Channel vs. Front Channel

Back Channel and Front Channel communication is not particular to OAuth. It
is more of a networking concept.

**Front Channel** communication is when network communication is done
through the browser. For example in the previous blog post, the user
accesses the client and gets sent (redirected) to the Authorization Server,
through their browser.

**Back Channel** communication is when a direct request is sent to a server
without any middle-man. This becomes highly secure especially when HTTPS is
used. An example of this is when the client directly calls the
Authorization Server to request an Access Token by providing a valid
Authorization Code. The client made a request directly to the Authorization
Server, and did not go through the user's browser.

<!-- markdownlint-disable -->
<!-- prettier-ignore-start -->
:::confusedDuck
How does the client know how to talk to the Authorization Server to send a
direct request for the Access Token?
:::

:::me
To answer that question, let's take a step back and look at the calls made
in the OAuth flow, with the perspective of the client.
:::
<!-- markdownlint-restore -->
<!-- prettier-ignore-end -->

## OAuth Flow - Client Perspective

### Client Initialization

The first step in the OAuth flow is for the client to get the registered
client ID and client secret from the authorization server.

<!-- markdownlint-disable -->
<!-- prettier-ignore-start -->
:::me
We’ll cover client registration when we explore the Authorization Server in upcoming posts in this series.  
:::
<!-- markdownlint-restore -->
<!-- prettier-ignore-end -->

Once the client has the client ID and the client secret, the client needs
to know how to talk to the Authorization Server. The client requires two
endpoints:

- The **authorization endpoint** to get the Authorization Code.
- The **token endpoint** to get an Access Token by providing a valid
  Authorization Code.

The client doesn't need to know anything about the Authorization Server
beyond that.

### User Redirection

Once the user accesses the client, the user is redirected to the
Authorization Server using the authorization endpoint `/authorize` to
initiate the authorization process. The client adds the following
parameters in the URL:

```bash
response_type: code
client_id: strava
redirect_uri: https://strava.com/callback
```

This would form an HTTP redirect like the following:

```bash
HTTP/1.1 302 Moved Temporarily
Location: https://auth-server:9001/authorize?response_type=code&scope=post&client_id=strava
-1&redirect_uri=http%3A%2F%2Fstrava%3A9000%
Vary: Accept
Content-Type: text/html; charset=utf-8
Content-Length: 444
Connection: keep-alive
```

And from the redirect, the client would send an HTTP request like the
following to the Authorization Server:

```bash
GET /authorize?response_type=code&scope=post&client_id=strava
-1&redirect_uri=http%3A%2F%2Fstrava%3A9000%
2Fcallback HTTP/1.1
Host: auth-server:9001
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
```

### Client Requests Access Token

The client receives an Authorization Code from the Authorization Server
when the user is redirected back to the client. The Authorization Code
represents as the result of the user's authorization decision. Here is what
the call back to the client from the Authorization Server looks like:

```bash
GET /callback?code=8V1pr0rJ
Host: strava.com
```

The client can parse the request and grab the Authorization Code. The
client is Now ready for the next step, which is requesting for the Access
Token. This is done by the client providing the following information in
the `POST` request to the `/token` endpoint on the Authorization Server:

- Client ID
- Client secret
- Authorization Code

This request is done directly between the client and the Authorization
Server. This request occurs on the **Back Channel**. Here is what the
request would look like:

```bash
POST /token
Host: auth-server:9001
Accept: application/json
Content-type: application/x-www-form-encoded
Authorization: Basic b2F1dGgtY2xpZW50LTE6b2F1dGgtY2xpZW50LXNlY3JldC0x
grant_type=authorization_code&
redirect_uri=https%3A%2F%2Fstrava%3A9000%2Fcallback&code=8V1pr0rJ
```

If the request is successful, then the client would get an Access Token
back from the Authorization Server. Here is what the call would look like:

```bash
HTTP 200 OK
Date: Fri, 31 Jul 2026 21:19:03 GMT
Content-type: application/json
{
“access_token”: “987tghjkiu6trfghjuytrghj”,
“token_type”: “Bearer”
}
```

### Client Accesses The Protected Resource

There are several methods for the client to present the Access Code to the
protected resource. In most cases the client presents a `bearer token`.
Here is what the call looks like:

```bash
GET /resource
Host: facebook.com
Accept: application/json
Connection: keep-alive
Authorization: Bearer 987tghjkiu6trfghjuytrghj
```

<!-- markdownlint-disable -->
<!-- prettier-ignore-start -->
  :::me
  Now that we have covered the basics of the OAuth flow in the perspective
  of the client, let's see how we can make the client more secure to shield
  the flow from potential attacks. 
  :::
  <!-- markdownlint-restore -->
  <!-- prettier-ignore-end -->

## The state parameter

The state parameter is generated by the client for every authorization
flow. It is stored in the client's local storage. The client then includes
this state parameter as a query to the Authorization Server in the initial
redirect of the user when requesting an Authorization Token.

When the user is redirected back to the client with the Authorization Token, the state parameter is also
included in the response. The state parameter is compared with what is
already stored in the client's local storage when it initially made the
request. If the client sees a mismatch, then the client rejects the
Authorization Token and ends the flow. Each state parameter needs to be
unique to every authorization flow.

## Simulating A CSRF Attack

Let's walk through a flow of how a CSRF attack can occur when no state
parameter is used.

1. The Attacker calls the Client.
2. The Client redirects the attacker to the Authorization Server.
3. The Attacker authenticates with the Authorization Server.
4. The Attacker approves the Client.
5. The Authorization Server redirects the Attacker with the Authorization
   Code. The Attacker stops the redirect from happening and grabs the
   Authorization Code.
6. The Attacker sends the user a phishing email with the redirect URI
   including the Attacker's Authorization Code.
7. The User's browser continues the OAuth flow by sending the Client the
   Attacker's Authorization Code.
8. The client requests an Access Token from the Authorization Server by
   using the attacker's Authorization Code.
9. The Authorization Server sends the client an Access Token paired to the
   Attacker's account
10. The Client makes a post to the attacker's Facebook account with the
    Access Token.

<!-- markdownlint-disable -->
<!-- prettier-ignore-start -->
:::magnifyingglassme
If the above steps looked confusing, here is a full video walkthrough, once
again.
:::
<!-- markdownlint-restore -->
<!-- prettier-ignore-end -->
<iframe
  src="https://player.mux.com/8XUb76hIAdOqfWL1S61zVEH5a6n100TTaaYwhHMtQP4A?metadata-video-title=CSRF&video-title=CSRF"
  style="width: 100%; border: none; aspect-ratio: 1/1;"
  allow="accelerometer; gyroscope; autoplay; encrypted-media; picture-in-picture;"
  allowfullscreen
></iframe>

![Diagram Legend](../images/legend-csrf.png)

## Simulating A CSRF Attack With State Parameter

Now let's walk through the same CSRF attack, but in this case, a state
parameter is used!

1. The Attacker calls the Client.
2. The Client redirects the attacker to the Authorization Server **with the
   state parameter**.
3. The Attacker authenticates with the Authorization Server.
4. The Attacker approves the Client.
5. The Authorization Server redirects the Attacker with the Authorization
   Code **and the state parameter**. The Attacker stops the redirect from
   happening, then grabs the Authorization Code and the state parameter.
6. The Attacker sends the user a phishing email with the redirect URI
   including the Attacker's Authorization Code and **state parameter**.
7. The User's browser continues the OAuth flow by sending the Client the
   Attacker's Authorization Code and **state parameter**.
8. The Client rejects the state parameter of the attacker and stops the
   flow.

<!-- markdownlint-disable -->
<!-- prettier-ignore-start -->
:::magnifyingglassme
If the above steps looked confusing, here is a full video walkthrough of a
simulated CSRF attack with the state paramter.
:::
<!-- markdownlint-restore -->
<!-- prettier-ignore-end -->

<iframe
  src="https://player.mux.com/g019hD002fSucWxu5e1tnW3uMN9PBzTxXLrbfzbLFMlSs?metadata-video-title=CSRF+state&video-title=CSRF+state"
  style="width: 100%; border: none; aspect-ratio: 1/1;"
  allow="accelerometer; gyroscope; autoplay; encrypted-media; picture-in-picture;"
  allowfullscreen
></iframe>

![Diagram Legend](../images/legend-csrf-state.png)

## Client Types

There are two client types in OAuth 2.0. Private clients, and public
clients.

A **private client** can also be known as a confidential client. An example
of a private client can be a web application with a backend. The client
secret can be stored in the backend of the web application and will not be
viewable to the public. This is because the client secret is not exposed to
the frontend. For this reason the private client can securely authenticate
with the authorization server using its own client secret. In the above
example, Strava is seen as a private client, since Strava has its own
backend and can store its client secret securely.

A **public client** is unable to store a client secret. An example of a
public client would be a single page application with no backend. If the
single page application were to store a client secret, then the client
secret would be exposed to the public making it a security risk.

## Refresh Tokn

<!-- markdownlint-disable -->
<!-- prettier-ignore-start -->
:::confusedDuck
What happens when lifetime of the Access Token runs out?
:::
<!-- markdownlint-restore -->
<!-- prettier-ignore-end -->

In that case, the OAuth flow, would start again and the user would be sent
to the Authorization Server again in order to delegate the user's
permissions to the client. This can get very cumbersome. Especially when
the lifetime of the Access Token is short. The user might have to grant the
client permissions mutliple times a day. To combat this the OAuth protocol
has a new token called the Refresh Token.

Adding a refresh token is possible in the Authorization Code Grant Type. If
Refresh Tokens are enabled on an Authorization Server, then the client
would receive a Refresh Token alongside the Access Token. The Refresh Token
generally has a longer lifespan than the Access Token. Once the client
receives a Refresh Token from the Authorization Server, it is then possible
to request an access token by sending but instead, changing the
`grant_type`:

```bash
{
 grant_type: 'refresh_token',
 refresh_token: "987tghjkiu6trfghjuytrghj"
}
```

**2.3** The authorization server might send back a `refresh_token` with the
`access_token`, if that is the case, then the client must use that
`refresh_token` going forward:

```bash
# RESPONSE FROM AUTHORIZATION SERVER
{

"access_token": "IqTnLQKcSY62klAuNTVevPdyEnbY82PB",
"token_type": "Bearer",
"refresh_token": "j2r3oj32r23rmasd98uhjrk2o3i"
}
```

The client cannot know if the Access Token has expired. The Authorization
Server can give a hint by providing an expiration date, but other than
that, the only way to know would be to use Access Code on the Protected
Resource. A well-behaved client would throw out the token before the
expiry. When the client requests a token a second time from the
authorization server, the user is not prompted; instead, a refresh token is
provided.

# References

- Token types, other than bearer token:
  <https://curity.medium.com/the-different-token-types-and-formats-explained-19dd8b947b2e>
