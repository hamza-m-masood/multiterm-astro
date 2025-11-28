---
title: "OAuth Grant Types"
published: 2025-12-10
draft: true
description: "Discussing various OAuth 2.0 grant types"
tags: ["OAuth"]
series: "OAuth Simplified"
---

## What is an OAuth 2.0 Grant Type?

## Password Grant Type

The is the most insecure grant type of OAuth 2.0 and is now seen as legacy.
During this flow the client takes the user's password and carries the
password over to the authorization server. This means that the user's
password is used by the client in exchange for an access token. This flow
is quite limiting and insecure. It does not allow for multi-factor
authentication, for example. Another downside is that the authorization
server can't differentiate between the user and client since both are using
the same password to login. Also there is no real way to get user consent
since there is no guarantee that the user is actually at the computer and
is willing to authorize the client.

This is an example of the authorization server notifying the user to
consent. This is not possible to do with the password grant. TODO: Show a
picture of a user consent screen

This flow is no different than a third party application asking for your
Gmail password. Instead the client is just asking for your password to the
authorization server.

## Authorization Code Flow Grant Type

For this flow, the user only enters their password in the authorization
server and nowhere else. With this way, the authorization server is certain
that the user is present and can confidently show the consent screen.

:::note Most of the time, the user consent screen is skipped for first
party, confidential clients. This is because if the user already has an
account on their own service, then it does not make much sense to ask the
user to delegate authorization to the client. Also there is no app
impersonation risk either since a confidential client is being used. :::

## PKCSE

If a public client does not have a client secret then how does the public
client get an access token from the authorization server?

This is where the Proof Key for Code Exchange (PKCSE) extension comes into
play for public clients. The public client generates a code challenge. The
code challenge is simply an SHA256 encrypted randomly generated string.
Once generated, the code challenge is added to the initial redirect when
the authorization code is requested.

Here is an example of what the authorization URL would look like:

```
https://authorization-server.com/authorize?
  response_type=code
  &client_id=73NbzDrSNDeXM4-aIfCJnHte
  &redirect_uri=https://www.oauth.com/playground/authorization-code-with-pkce.html
  &scope=photo+offline_access
  &code_challenge=NLMnmQNiZnKI_J9eEQIZLT1cZpZA-TbxuGMm3Te-54g
  &code_challenge_method=S256
```

When the public client then requests the access token through the back
channel from the authorization server after user approval, the code
challenge is verified on the authorization server.

The PKCSE extension makes sure that the authorization code is given the
same app that started the flow. However, when using only the PKCSE
extension, the authorization server can't identify the app. This leaves the
app open to being impersonated.

Since a client secret can't be used to identify a public client, the only
real way is to make sure the redirect URI is unique to the public client.
This is why it is so important to register the correct redirect URIs to the
authorization server. Especially for public clients. By using a combination
of PKCSE and unique redirect URIs, public clients can be correctly
identified and the authorization server can make sure that the access token
is sent to the same client that started the flow.

<!-- markdownlint-disable -->
<!-- prettier-ignore-start -->
:::note
PKCSE was initially created for mobile or single page applications
which are treated as public clients. But recently, the OAuth 2.0 spec
recommends to use PKCSE even for private clients to safeguard applications
from attacks such as Authorization Code Injection. Even if your
authorization server does not support PKCSE, you can still provide it in
the URL, since authorization servers are designed to ignore parameters that
they do not recognize.
:::
<!-- markdownlint-restore -->
<!-- prettier-ignore-end -->

In the next blog post, we will dive deep into the OAuth client and discuss
its inner workings.
