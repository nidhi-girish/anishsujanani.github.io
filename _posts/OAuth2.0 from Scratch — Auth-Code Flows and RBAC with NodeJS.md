# What is OAuth2.0? What is an Authorization-Code Flow? Why do I care?
The OAuth2.0 framework defines a set of protocols that allow an application to obtain authorization grants (permissions) for certain resources/actions of a user by delegating authentication and consent to a centralized server.  
  
_For example, a web application offering a ‘Sign in with Google’ feature and requesting for permissions to ‘Read GMail Contacts’ and ‘Read Drive Information’._

**The centralized interface between the user and the application allows for the following:**
- Better authentication and authorization practices — developers on different projects need not build their own user-store, authentication mechanisms and user roles/permissions.
- Third-party applications need not implement their own identity systems. They can instead rely on other providers that their users trust.
- Centralized systems can ensure that applications access data that the user explicitly consented to and nothing more.

# Terminology
- **Resource-Owner:** the end-user who will consent to permissions and authenticate.
- **Client:** the application accessed by the Resource-Owner that would like to access a service that the Resource-Owner uses (typically via API).
- *A*uthorization-Server:** the centralized server that carries out the actual authentication and scopes the access of the Client to the Resource-Owner’s resources.
- **Resource-Server:** are application servers that provide API services. This server will issue services based on the access token provided to it by a Client. It fully trusts that access token as it has been issued by the centralized Authorization-Server that this server is also integrated with.

OAuth2.0 provides multiple auth-flows, of which the `Authorization Code Flow` is primarily used for Resource-Owners communicating with Client web applications and goes as follows:
- **Precursor:** The **Client** and the **Authorization-Server** go through a ‘registration’ process where parameters such as `client_id, client_secret, and redirect_uri` among others are decided. The technical usage of these parameters is explained in the implementation section.
- *Step 1:* The **Resource-Owner** contacts the **Client** (a web-app) through a browser, and chooses to grant certain permissions that the application requests.
- *Step 2:* The **Client** redirects the **Resource-Owner** to the **Authorization-Server**’s `/auth` endpoint, where the **Resource-Owner** actually authenticates with credentials and consents to the **Client** getting stated permissions.
- *Step 3:* The **Authorization-Server** redirects the **Resource-Owner** back to the **Client** with a `code`.
- *Step 4:* The **Client** receives the `code`, and once again reaches out directly to the **Authorization-Server** and exchanges it for an access token through the **Authorization-Server**’s `/token` endpoint. (This seems redundant, why exchange a code for a token? Explained in the Step 4 implementation)  

The *Client* will now have an access token that has been created by a trusted source — the **Authorization-Server**, to which the **Resource-Owner** authenticated to and consented the grant of permissions. This token may now be passed to **Resource-Servers** that trust the **Authorization-Server **which can now securely provide services to the **Client**.
Below is an implementation of the `Authorization Code Flow` from scratch. Extending this for RBAC is covered after.

# Step 1: Client App — Requesting a Code
Setting up code on the **‘Client’** ie. the web application that the **Resource-Owner** will access. Remember that the registration step has already occurred. The **Authorization-Server** has the details of the **Client** application including unique identifiers such as the `client_id` and `client_secret`, which `Referer` it will be sending requests from, and which `redirect-uri` information is to be sent to as a callback.

The page served by the Client web application is as follows:
[oauth_1](https://raw.githubusercontent.com/anishsujanani/anishsujanani.github.io/master/assets/img/oauth_1.png)