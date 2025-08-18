---
title: 'Part 3 - The OAuth Authorization Server'
published: 2025-08-01
draft: false
description: 'Learning about the OAuth Authorization Server from the authorization grant type'
tags: ['OAuth']
series: 'OAuth Series'
---

## 1 Introduction

<video src="https://github.com/user-attachments/assets/0caebe87-8d06-48d8-ac72-71fc5e3ad76c" controls autoplay loop muted></video>

**1.1** In this section, there will be a deep dive into the Authorization server.

The authorization server that will be discussed supports the authorization code grant type has the following features:

- Registers clients.
- Issues tokens to clients.
- Assigns a scope to an issued token.
- Issues refresh tokens to clients.
- Authenticates users.

Because of the vast amount of features the authorization server must support, it is arguably the most complex component in the OAuth ecosystem.

**1.2** In the OAuth protocol, most complexity is pushed to the authorization server because authorization servers are the fewest in number. This means there are many more clients than protected resources, and there are many more protected resources than authorization servers. As can be seen in figure 1.3.

![showing number of OAuth entities](../images/size.png 'Figure 1.3 - Shows the difference in number between the OAuth entities.')

## 2 Registering a Client

**2.1** The simplest form of managing clients on the authorization server is static registration. This means the authorization server predefines all the unique client identifiers (client IDs) needed for all clients that connect to the authorization server. The clients would be stored in a database on a production-level authorization server.

**2.2** Once the client IDs are defined, the authorization server generates a client secret for each client. This secret is also stored in the database for production-level authorization servers.

**2.3** The final step in registering a client is to define a redirect URI. Unlike the client ID and the client secret, the authorization server does not generate the redirect URI. The redirect URI is defined by the client and manually entered into the authorization server.

At the end of the client registration process, the authorization server would have a client object like this:

```JSON
{
  "client_id": "oauth-client-1",
  "client_secret": "oauth-client-secret-1",
  "redirect_uris": ["http://client-server:9000/callback"],
}
```

## 3 Authorizing a Client

**3.1** As discussed, the authorization server needs to authorize the client on behalf of the user so the client can receive an authorization code. This communication is done over the front channel, which needs to be reachable by the user's browser. Since most authorization servers are web servers, the authorization endpoint typically has the `/authorize` path and is always a `GET` request.

**3.2** Firstly, when the `/authorize` endpoint is called, the authorization server finds out which client made the request. Typically, the client passes its identifier in the `client_id` parameter and its redirect URI in the `redirect_uri` parameter.

**3.3** Once the client ID has been found, the authorization server must determine if the client exists in its database of predefined clients. If the client does not, an error is emitted such as `{error: 'Unknown client'}`. A classic check is to see if the `client_id` and `redirect_uri` match what is already stored in the database. Since only checking the `client_id` could lead to security gaps, all the current communication is being done on the public front channel. This can be seen in figure 3.6.

According to the OAuth specification, a client can register multiple redirect URIs for itself, allowing the client to be served from different URLs in different circumstances. This can complicate the client authorization process.

**3.4** When the client is authorized, a `request ID` number is randomly generated to keep track of the client's initial authorization request. As we will see in the next step, this request ID will protect the server from cross-site request forgery. Also, this `request ID` will get stored in the database alongside the specific client's information that sent the initial authorization request to receive the authorization code.

**3.5** The client is now authorized, with its request ID safely stored in the authorization server's database. The next stage prompts the user to authorize the client on the user's behalf. This is done so the request ID from the form can be validated with the request ID from the initial request by the client.

![showing the client authorization](../images/client-authorization.drawio.png 'Figure 3.6 - Shows the client authorization flow.')

## 4 User Decision

**4.1** After the client is authorized, the user is prompted to give the client the relevant permissions to access the protected resource and act on the user's behalf. This can be done through a form using a UI. Here is an example:</br>

<fieldset>
Approve this Client?</br>
ID: `test-client`
    <select>
        <option value="1">Approve</option>
        <option value="2">Reject</option>
    </select><br>
    <button type="submi">Submit</button>
</fieldset>

**4.2** The user can approve or reject the client who requested to be authorized and act on behalf of the user. After clicking the `Submit` button, an API request is typically sent to the endpoint path `/approve` on the authorization server.

:::warning
:warning:</br>
The OAuth 2.0 protocol does not care if the user is **authenticated** when the authorization server prompts the user to authorize the client. User authentication is entirely outside the scope of OAuth 2.0. This is why adequate care is required to supply user authentication at this stage.</br>
:warning:
:::

**4.3** The `request ID` from the request in the previous section is embedded into this form in the background. So when the user clicks on Submit, the `request ID` is embedded into the API call to`/approve` in the authorization server. Here is the request object that is sent to the `/approve` endpoint path:

```JSON
{
  "client_id": "oauth-client-1",
  "client_secret": "oauth-client-secret-1",
  "redirect_uris": ["http://client-server:9000/callback"],
}
```

## 5 Processing the User Decision

**5.1** If the user **rejects** the client, this means the user has denied access to an otherwise valid client. The authorization server now has the responsibility to tell the client that the user has rejected its authorization request. This can be done the same way as the client communicated with the authorization server. The authorization server will take a URL hosted by the client, add a few special query parameters, and redirect the user to that location. The URL that is hosted by the client, which the authorization server can use is known as the `redirect URI`. The authorization server sends back an error message to the `redirect URI`with the message `error: access_denied`.

**5.2** This is why the client's `redirect URI` is needed. Also, this is why the authorization server validated the client's `redirect URI` against the existing client information in the authorization server's database when the initial authorization request arrived by the client.</br>

**5.3** If the user has **approved** the client, then this means the user allows the client to act on their behalf.</br>
When the `/approve` is called with `approve`, the first step is to check what kind of response the client seeks. The HTTP `response_type` should be `code`. If not, error is sent to the `redirect URI`.</br>
The second step is to generate the authorization code and save it into the database because the authorization code will need to be referenced by the authorization server for later steps as we will see.</br>
Finally, the third step is to send back the authorization code, and the `scope` (if provided by the client in the initial authorization request) to the client through the client's `redirect URI` and hand back control to the client. The authorization server is now fully prepared for the next step in the OAuth 2.0 authorization code grant type flow.

## 6 Issuing a Token

**6.1** At this point the client is now in control and has the authorization code that was received from the front-channel by the authorization server. The next step is to request an authorization token by sending a `POST` request to the `/authorize` endpoint of authorization server. The client can seend the code either in the header or the form body. Well behaved authorization servers would accept either methods but not both at the same time.

**6.2** When the `/authorize` endpoint is called, the authorization server will validate the following in order by comparing the incoming data from the client to what is already stored in the authorization server's database:

- The client ID
- The client secret
- The authorization code

:::note
Once the authorization code is validated, it is saved inside a variable at runtime and is deleted from the database. This is to prevent the use of a stolen authorization code and err on the side of caution.
:::

**6.3** When all the relevant data is validated, the next step is to check if the `grant_type` is set to `authorization_code` in the body of the `POST` request sent by the client. Since the authorization server in this case only supports the authorization code flow grant type, it is important to check.

**6.4** If the authorization server does find the authorization code grant, the authorization server then needs to generate an **authorization token**. OAuth 2.0 is famously silent about what is inside an authorization token. It is up to you to add whatever content you want inside the authorization token. For example, you can create a JWT token. But for this case, we will keep things simple and generate a random string, then store it in the database.

**6.5** Now the authorization server can finally send the token back to the client in the form of a JSON object that includes the authorization token. The JSON object should also include how the protected resource can use the authorization token. The usage method of the authorization token can be communicated by specifying the type of the authorization token. In this case, the authorization server sends back a `Bearer` token:

```JSON
{
  "access_toke": "lRQUChwvWf",
  "token_type": "Bearer"
}
```

**6.7** At this point, we have stepped through a simple but fully functioning authorization server:

- Authenticating clients
- Prompting users for authorization
- Issuing randomized bearer tokens using the authorization code flow.

Nice! :rocket:

The next steps can be seen as optional added features to the authorization server.

## Scope

<fieldset>
Approve this Client?</br>
ID: `test-client`
    <select>
        <option value="1">Approve</option>
        <option value="2">Reject</option>
    </select><br>
    <label><input type="checkbox"> read<br></label>
    <label><input type="checkbox"> write<br></label>
    <label><input type="checkbox"> delete<br></label>
    <button type="submi">Submit</button>
</fieldset>

You can see that the user can give the client fine-grained permissions by providing the client with read, write, or delete permissions. These permissions are added in the scope section when the token is issued in later steps.
