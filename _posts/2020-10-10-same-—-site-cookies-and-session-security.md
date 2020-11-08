*The addition of SameSite Cookie Policies to Chrome opens up a discussion on improved browser-based security through its implications on sessions, cross-origin assets, user experience and service development.*

## Maintaining State in a Stateless Protocol
HTTP is a stateless protocol, i.e. servers are not natively aware of the actions that a client requested for prior to the current one it is servicing. Without state, it would not be possible to interact with web applications the way we do today — custom user profiles, shopping carts, preferences, analytics; at least not securely.

### Cookies
A cookie is a header placed in the server response containing key-value pairs, which after being set, is automatically sent on every subsequent request made by your browser. Though this enables state tracking on the client’s end, it becomes trivial for a client to manipulate their own cookies to change the way your application behaves. Encrypting the cookies provides some additional tamper-proofing, but disabling cookies and weak encryption can still cause unexpected behaviour and compromise, especially if you store sensitive information in the key-value strings. JWT: JSON Web Tokens might be worth looking into as a more modern client-side state mechanism.

### Sessions
Sessions are typically implemented for state-maintenance on the server. All information is stored on the server, typically in memory or filesystem or an external service (eg. Redis/Memcached) and is referenced by a session-ID. This session-ID alone is set on the cookie that goes out to the client. All subsequent requests will have this session-ID as a cookie and can be used for processing on the server. This is an improvement since we aren’t storing information in client-side cookies anymore, but what stops a bad guy from just guessing other user session-IDs? 16-character UUIDs are typically used to defend against this.

### The Case for More Secure Cookies

### Enter — CSRF
Let’s say you are logged in to an important service on tab 1 of your browser. Assume that this service has an endpoint/action that you’d like to securely interact with, eg. viewing the balance of your bank account or transferring funds to someone else. A bad actor sending you a seemingly benign unrelated link which actually redirects your browser to those sensitive endpoints when you click on it is what we call Cross-Site Request Forgery. As a toy example:
`<a href="https://yourservice.com/deleteaccount"> Cat Photos </a>`

The above would look like a link for cat photos but following it would invoke account deletion. Most web applications do not execute such actions with GET requests. They usually involve form submission via POST, but you get the idea. If the bad guy hosts his own site, he could set up a form exactly as the service expects it but with the fields hidden. You would not be able to see these fields in the browser window, but clicking a link or a button would submit it. Standard methods to defend against such practices include:
- The service using anti-CSRF tokens in their forms.
- Implementing strong CORS and CSP headers to prevent scraping of the tokens and injection.
- Implementing strong same-site cookie policies.

## Server Set-up
Host: yourcriticalservice

### Routes:
- User: `GET /` ................... Server: returns static page with a ‘get cookie’ button (which invokes POST /requestcookie)
- User: `POST /requestcookie` ..... Server: sets a cookie and redirects to /sensitivepage
- User: `GET /sensitivepage` ...... Server: checks for cookie and returns a ‘your profile’ page
- User: `GET /sensitiveform` ...... Server returns page for “transferring units to an email address”
- User: `POST /sensitiveform` ..... Server performs the transfer

Intended functionality: 
![samesite_1.png]({{site.baseurl}}/assets/img/samesite_1.png)

```javascript
var express = require('express');
var app = express();
var path = require('path');
var session = require('express-session');
var bodyparser = require('body-parser');

app.use(bodyparser.urlencoded({extended: true}));

app.use(session({
	secret: 'MySecretKey#1324',
	saveUninitialized: false,
	key: 'nodesesscookie',
	resave: false,
	cookie: { expires: 60000, sameSite: '!!!! ENTER CONFIG HERE !!!' } // none, lax, strict
}));

app.get('/', function(req, res) {
	res.sendFile(path.join(__dirname + '/static/landing.html'));
});

app.post('/requestcookie', function(req, res) {
	// authenticate user here thrrough DB cred checks ...
	// set session vars, may include perm matrix?
	req.session.key_a = 'val_a';
	req.session.key_b = 'val_b';
	req.session.user = 'admin';
	res.redirect('/sensitivepage');
});

app.get('/sensitivepage', function(req, res){
	// typically would check for valid account here and permissions
	if (req.session.user != 'admin')
		res.send('Not logged in, redirecting to /login');
	else
		res.sendFile(path.join(__dirname + '/static/profile.html'));
});

app.get('/sensitiveform', function(req, res) {
	if (req.session.user != 'admin')
		res.send('Not logged in, redirecting to /login');
	else
		res.sendFile(path.join(__dirname + '/static/sensitiveform.html'));
});

app.post('/sensitiveform', function(req, res) {
	// business logic here.
	if (req.session.user != 'admin')
		res.send('Not logged in, redirecting to /login');
	else {
		var m = ' ! Transferred: ' + req.body.units + ' to ' + req.body.recipient;
		console.log(m);
		res.send(m);
	}
});

app.listen(5000, function() {
	console.log('Server is live and listening on 5000');
});
```

## Simulating a Bad-actor's server (running on localhost)
Below is the server that a bad actor might run. Note that it is running on a different origin compared to yourcriticalservice:
```python
import flask
import os
import requests

app = flask.Flask(__name__)

@app.route('/bget', methods=['GET'])
def servebget():
    return flask.render_template('bget.html')

@app.route('/bpost', methods=['GET'])
def servebpost():
    return flask.render_template('bpost.html')

app.run(host='localhost', port=8000, debug=True)
```


Below are the links that you, the user of yourcriticalservice would be directed to interact with:

`bget.html`:
```
<html>
	<a href="http://yourcriticalservice.com:5000/sensitivepage">Click me</a>
</html>
```

`bpost.html`:
```
<html>
	<body>
		<form action="http://yourcriticalservice:5000/sensitiveform", method="POST">
			<input name="units" type="hidden" value="1000"/>
			<input name="recipient" type="hidden" value="badguy@badguy.com">
			<button type="submit" value="submit">Special Offer!</button>
		</form>
	</body>
</html>
```

## Testing Same-Site Cookies — None/Lax/Strict

### Same-Site = None:

![samesite_2.png]({{site.baseurl}}/assets/img/samesite_2.png)

Setting the same-site cookie policy to none basically strips away all protection and allows cross-origin cookie sharing. Upon a user being signed into yourcriticalservice a bad actor can GET /sensitivepage by just redirecting the URL ie. TOP-LEVEL NAVIGATION.

Upon clicking on the ‘Special Offer!’ button in bpost.html, the form provided by yourcriticalservice would be submitted and if there is no anti-CSRF protection, it is executed. FORM SUBMISSION VIA POST IS ALLOWED.

**Top-level redirection transmits the set cookiec cross-origin POST transmits set cookie:**
![samesite_3.png]({{site.baseurl}}/assets/img/samesite_3.png)

*This is usually a bad idea.*

### Same-Site = Lax:

![samesite_4.png]({{site.baseurl}}/assets/img/samesite_4.png)
This is the default implementation of cookies in Chrome. It allows for cookies to be sent via link clicks/redirects (top-level navigation) but prevents form submissions via POST.

**Top-level redirection transmits the set cookie, cross-origin POST does not transmit the set cookie:**
![samesite_5.png]({{site.baseurl}}/assets/img/samesite_5.png)

This policy provides browser-level protection against vectors such as CSRF while providing the flexibility for other domains to request for assets via embedding (image/CSS/javascript tags) and top-level navigation (direct link-clicks).

### Same-Site = Strict:
![samesite_6.png]({{site.baseurl}}/assets/img/samesite_6.png)

This configuration does not allow cookies to be sent at all, regardless of submission, embedding or top-level navigation. This cookie policy should be used when you want to prohibit all other services from redirecting into your application via top level redirection, asset embedding and form submission.

Top-level redirection does not transmit the set cookie, Cross-origin POST does not transmit the set cookie:**
![samesite_7.png]({{site.baseurl}}/assets/img/samesite_7.png)

---------------

It is important to understand that SOP, CORS and Same-Site cookie policies are all browser protections. It is trivial to alter headers or send requests out directly via script. You still need to emphasize strong anti-CSRF protection on your forms and implement input sanitization+CSP to prevent injection.

With that said, the ‘strict’ setting is worth implementing in critical applications where zero external service interaction is a requirement.
