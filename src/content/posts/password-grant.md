---
title: 'OAuth Grant Types'
published: 2025-12-10
draft: false
description: 'Discussing various OAuth 2.0 grant types'
tags: ['OAuth']
series: 'OAuth Simplified'
---

## What is an OAuth 2.0 Grant Type?

## Password Grant Type

The is the most insecure grant type of OAuth 2.0 and is now seen as legacy. During this flow the client takes the user's password and carries the password over to the authorization server. This means that the user's password is used by the client in exchange for an access token. This flow is quite limiting and insecure. It does not allow for multi-factor authentication, for example. Another downside is that the authorization server can't differentiate between the user and client since both are using the same password to login. Also there is no real way to get user consent since there is no guarantee that the user is actually at the computer and is willing to authorize the client.

This is an example of the authorization server notifying the user to consent. This is not possible to do with the password grant.
TODO:
Show a picture of a user consent screen

This flow is no different than a third party application asking for your Gmail password. Instead the client is just asking for your password to the authorization server.

## Authorization Code Flow Grant Type

For this flow, the user only enters their password in the authorization server and nowhere else. With this way, the authorization server is certain that the user is present and can confidently show the consent screen.

:::note
Most of the time, the user consent screen is skipped for first party, confidential clients. This is because if the user already has an account on their own service, then it does not make much sense to ask the user to delegate authorization to the client. Also there is no app impersonation risk either since a confidential client is being used.
:::
