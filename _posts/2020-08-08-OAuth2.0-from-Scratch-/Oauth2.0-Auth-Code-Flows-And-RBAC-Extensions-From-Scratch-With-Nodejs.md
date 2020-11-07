---
published: false
---
# What is OAuth2.0? What is an Authorization-Code Flow? Why do I care?
The OAuth2.0 framework defines a set of protocols that allow an application to obtain authorization grants (permissions) for certain resources/actions of a user by delegating authentication and consent to a centralized server.  
  
_For example, a web application offering a ‘Sign in with Google’ feature and requesting for permissions to ‘Read GMail Contacts’ and ‘Read Drive Information’._

**The centralized interface between the user and the application allows for the following:**
- Better authentication and authorization practices — developers on different projects need not build their own user-store, authentication mechanisms and user roles/permissions.
- Third-party applications need not implement their own identity systems. They can instead rely on other providers that their users trust.
- Centralized systems can ensure that applications access data that the user explicitly consented to and nothing more.

# Terminology
- **Resource-Owner**:** the end-user who will consent to permissions and authenticate.
- **Client:** the application accessed by the **Resource-Owner** that would like to access a service that the **Resource-Owner** uses (typically via API).
- **Authorization-Server:** the centralized server that carries out the actual authentication and scopes the access of the **Client** to the **Resource-Owner**’s resources.
- **Resource-Server:** are application servers that provide API services. This server will issue services based on the access token provided to it by a Client. It fully trusts that access token as it has been issued by the centralized **Authorization-Server** that this server is also integrated with.

OAuth2.0 provides multiple auth-flows, of which the `Authorization Code Flow` is primarily used for **Resource-Owner**s communicating with **Client** web applications and goes as follows:
- **Precursor:** The **Client** and the **Authorization-Server** go through a ‘registration’ process where parameters such as `client_id, client_secret, and redirect_uri` among others are decided. The technical usage of these parameters is explained in the implementation section.
- **Step 1:** The **Resource-Owner** contacts the **Client** (a web-app) through a browser, and chooses to grant certain permissions that the application requests.
- **Step 2:** The **Client** redirects the **Resource-Owner** to the **Authorization-Server**’s `/auth` endpoint, where the **Resource-Owner** actually authenticates with credentials and consents to the **Client** getting stated permissions.
- **Step 3:** The **Authorization-Server** redirects the **Resource-Owner** back to the **Client** with a `code`.
- **Step 4:** The **Client** receives the `code`, and once again reaches out directly to the **Authorization-Server** and exchanges it for an access token through the **Authorization-Server**’s `/token` endpoint. (This seems redundant, why exchange a code for a token? Explained in the Step 4 implementation)  

The **Client** will now have an access token that has been created by a trusted source — the **Authorization-Server**, to which the **Resource-Owner** authenticated to and consented the grant of permissions. This token may now be passed to **Resource-Servers** that trust the **Authorization-Server **which can now securely provide services to the **Client**.
Below is an implementation of the `Authorization Code Flow` from scratch. Extending this for RBAC is covered after.

# Step 1: **Client** App — Requesting a Code
Setting up code on the **‘Client’** ie. the web application that the **Resource-Owner** will access. Remember that the registration step has already occurred. The **Authorization-Server** has the details of the **Client** application including unique identifiers such as the `client_id` and `client_secret`, which `Referer` it will be sending requests from, and which `redirect-uri` information is to be sent to as a callback.

The page served by the **Client** web application is as follows:
![oauth_1.png]({{site.baseurl}}/_posts/2020-08-08-OAuth2.0-from-Scratch-/oauth_1.png)


When the user (**Resource-Owner**) picks an option and submits, a POST route services the request. The **Client** redirects the **Resource-Owner** to the **Authorization-Server**’s /auth endpoint with pre-defined parameters such as the client_id and response_type as well as the selected scope and generated state.

The state parameter is of additional importance as it enables the **Client** and **Authorization-Server** to synchronize throughout the flow and also prevents CSRF/replay attacks by a bad-guy trying to get access tokens. (See CSRF in references)

# Step 2: Authorization-Server — Authenticating the **Resource-Owner** and Sending back a Code to the Client
The registration process has been completed prior and is stored here in the form of a JSON in the source. Details such as the issued client_id, client_secret, referer — the address that will make requests and redirect_uri — the address to callback to are recorded.

## Implementing the /auth Endpoint and validating the Client:
As seen earlier, the **Resource-Owner** submitting the form on the Client’s page will trigger a redirect to this server’s /auth endpoint with certain parameters about the **Client** itself. This implementation has a separate function to match the parameters in the request to the parameters recorded during registration.

## Performing **Resource-Owner** authentication after **Client** validation:
Once the Client’s request parameters have been verified, the **Authorization-Server** needs to authenticate the **Resource-Owner**. This may be done through a traditional email-password entry or through other factors and creation of a session with user details. For simplicity, this implementation assumes a single generic user and uses a button to simulate successful authentication.

The authentication page that the **Authorization-Server** presents after validating the incoming **Client** request:
[oauth_2](https://raw.githubusercontent.com/anishsujanani/anishsujanani.github.io/master/assets/img/oauth_2.png)

## Does this look familiar?
This is essentially what you see when you sign in with Google or install new applications that ask for permissions, eg. ‘this app would like access to make phone calls, read contacts, read messages’, etc. Each of these ‘permissions’ would be formally defined by **Resource-Servers** , integrated with the **Authorization-Server** and requested by the Client, typically with a naming convention such as ‘phone:write contacts:read messages:read’.
Once the user agrees to the scope requested by the **Client** by authenticating, this **Authorization-Server** will redirect back to the Client’s pre-registered redirect_uri with a one-time code and the state that it was sent originally.
It also maintains the code and its relevant token in a cache to ensure that it can only be exchanged once within a limited time period.
This implementation keeps it simple and maps a code to a token containing fields such as authorized, username, and scope. You may choose to issue additional information such as an intended audience, issuer details, timestamps, etc.

# Step 3: **Client** App — Exchanging the Code for a Token
The **Client** listens on a pre-registered callback endpoint for a code. This code is validated by ensuring that it is tied to a state, ie. it was expected and not unsolicited. The **Client** will now POST to the **Authorization-Server**’s /token endpoint with the code and its client_secret. The client_secret is sent to authenticate the **Client** and to ensure that a man-in-the-middle cannot exchange codes for tokens and get unauthorized access.
The token that is sent back is a signed JWT (JSON Web Token). The hash of its payload is verified with the **Authorization-Server**’s public key and can then be used to access services on **Resource-Servers** . (See references for JWT format and verification).

**But all of this seems redundant, right? Why do we go through all the redirection to get a code, only to send the code back immediately for a token?**
This is done because the code was obtained through redirecting and interacting with the **Resource-Owner**’s user-agent, typically a user’s browser. Even though we whitelist parameters such as client_id, client_secret, redirect_uri, and others, we ultimately should not trust what the end-user gives us. This code is exchanged over a secure channel between the **Client** and the **Authorization-Server** directly that does not have any interaction with the public users or their browsers. There are, however, some cases where this may be desired. Certain flows such as the ‘Implicit Flow’ correspond to this and are usually implemented with Single-Page-Applications — client-side only processing, see references.

# Step 4: Authorization Server — Sending back a Token for a Code
The Client, having received a code, will exchange it for an access token after authenticating itself to the **Authorization-Server** through a pre-defined client_secret. This implementation does not cover timeouts, but codes are usually valid for a very short period of time to prevent replay attacks.
The **Authorization-Server** then places the token corresponding to that code in the payload of a JWT, signs a hash of the token with its private key for proof of integrity and sends it back.

# The Result
Now that all 4 steps have been completed, here is what the **Client** sees:
[oauth_3](https://raw.githubusercontent.com/anishsujanani/anishsujanani.github.io/master/assets/img/oauth_3.png)

A token within a signed JWT from the trusted **Authorization-Server** that it registered with. This token is proof that the **Resource-Owner** did, in fact, authenticate successfully with the **Authorization-Server** and consented to the **Client** getting the ‘resource1_read resource1_write’ permissions.
The **Client** app may then use this token with other API services that require those permissions.

**Therefore, this flow has enabled a third party **Client** to gain the permissions it needs through delegating authentication and user consent to authorization to a centralized server that both the third-party **Client** AND the user trust.**

------

This works great when a third-party application requires permissions to access user resources from one or more services. Can we extend this framework for RBAC in our own trusted applications?

# Extending OAuth2.0 for Role Based Access Control
You might have multiple internal applications developed by different teams. Having each team maintain their own user-base, permission system and authentication procedure is not only inefficient but is usually insecure.
We can extend the OAuth Auth Code Flow for RBAC by implementing a new scope on the **Authorization-Server** that returns all the roles and attributes for the **Resource-Owner**.
This requires changes to Step 2: **Authorization-Server** — Authenticating the **Resource-Owner** and Sending back a Code to the **Client**.
When the **Client** requests for a ‘getallrolesandperms’ scope, carry out the same **Resource-Owner** authentication and consent process but generate a token that contains all the applications and roles that the **Resource-Owner** has access to.

Minor change to the client’s front-end page to allow a user to pick this scope:
[oauth_4](https://raw.githubusercontent.com/anishsujanani/anishsujanani.github.io/master/assets/img/oauth_4.png)

# The Results — Extending OAuth2.0 for RBAC

[oauth_5](https://raw.githubusercontent.com/anishsujanani/anishsujanani.github.io/master/assets/img/oauth_5.png)

The **Client** gets back a token within a signed JWT containing user attributes. This can then be used to implement RBAC within the application and user-specific content tailoring. You may also be interested in OpenID Connect (see references) — an additional Identity Layer on top of OAuth2.0 through defining a new scope.
**OAuth2.0 can not only be used for delegating consent and authentication to a trusted party , but also for centralizing RBAC for internal highly trusted applications.**

# Further reading
