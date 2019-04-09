# VS-JWT
## Token. JSON Web Token.

JSON Web Token, or **JWT**, is a relatively simple way to securely exchanging data between authorized client  and server. Once User had sent his login and password to **Authentification Server**, it generates signed access token as JSON object and sends it back to **client**. When client issues request to **Resourse Server** it attaches JWT to header using a Bearer scheme. After validation check server accepts and processes request. Also JWT is a great technology for server-to-server authorization.

### What is it, exactly

![alt text](https://raw.githubusercontent.com/vonsavoyen/VS-JWT/master/JWT%20Diagram.jpg)

JWT consists of three components: Header, Payload and Signature. Header and Payload contains useful information and encoded with **Base64URL** algorithm. Signature is creating by taking encoded Header and Payload, secret key and an algorithm specified in Header. All three parts **serialize** into one string and separate with “.” symbols (for example *AiLCJhbGciOiJIUzI1NiJ9.J1c2VySWQiOiJiMDhmODZhZi0zNWRhLTQ4ZjItOGZhYi1jZWYz.zN_h82PHVTCMA9vdoHrcZxH-x5mb11y1537t3rGzcM*). At the server’s side JWT de-serialize using known secret key and then provide access to requested information.

The Header component contains two fields:
*{
	“typ” : “JWT”,
	“alg” : “HS256”
}*

The “typ” field specifies JWT object, the “alg” is preferred hashing algorithm to use to create JWT signature component. Dozen of algorithms are supported by different libraries, please visit https://jwt.io/#libraries for details.

The Payload contains various data about the user and few standard fields. **None of them are mandatory**, though.

For instance, Payload can store User’s ID, name and company’s position
```
{
	“UserID” : “16060131”,
	“UserName” : “Booker DeWitt”,
	“CompanyPos” : “admin”
}
```
and also reserved fields:

*iss (issuer)* : Identifies principal that issued the JWT.
*sub (subject)* : Identifies the subject of the JWT.
*aud (audience)* : Identifies the recipients that the JWT is intended for.
*exp (expiration time)* : Identifies the expiration time on and after which the JWT must not be accepted for processing.
*nbf (not before)* : Identifies the time on which the JWT will start to be accepted for processing.
*iat (issued at)* : Identifies the time at which the JWT was issued.
*Jti (JWT ID)* : Case sensitive unique identifier of the token.

To create **Signature** you need encoded Header and Payload, and also a secret key – only servers should have it. Then it has to be converted into bite array by using selected hashing algorithm.
```
HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  secret)
```
Usually JWT’s have an **expiration time** (*“exp”* property of the Payload), after which token fails to pass validation check on server’s side. When it happens, client generates so-called **Refresh Token**, which requests new, fresh and shiny Access Token.

It is important to store JWT in safe place inside an **HttpOnly cookie**. Also, keep in mind that JWT isn't encrypted and doesn’t designed as protected container for sensitive information.

### How to use it

First, you need to choose your perfect **JWT library** from those that available at https://jwt.io/#libraries. In general, the more greenlighted options it has, the better. Then, you have to create an **authorization server**. Create an empty solution and add a new ASP.NET Web application.

There are two basic functions responsible for JWT implementation: *jwt.sign* and *jwt.verify*. The first function will generate a JWT Token, the second verify the user’s token when a protected route is accessed.

*jwt.sign (payload, secretkey, [options, callback])* has two additional parameters: *“options”* allows to set values of payload’s reserved fields including expiration time; *“callback”* is where we handle sending our token. *jwt.verify(token, secretkey, [options, callback])* takes the whole token as single parameter and the secret key is defined in the *“jwt.sign”* function. Please note that the secret key should be stored in an environment variable, like all sensitive information.

We start from importing JWT and user dummy model:
```
const jwt = require('jsonwebtoken');
const user = require('../../models/dummyUser');
```
Then we have a POST route found at */user/login* and check that username and password match our dummy user:
```
app.post('/user/login', (req, res, next) => {
	const { body } = req;
	const { username } = body;
	const { password } = body;

if(username === user.username && password === user.password) {
```
And if user’s login was successful we generate a JWT token for the user with a secret key:
```
jwt.sign({user}, 'privatekey', { expiresIn: '1h' },(err, token) => {
	if(err) { console.log(err) }
		res.send(token);
	});
	} else {
	console.log('ERROR: Could not log in');
	}
})
```
The *“privatekey”* here is the secret key, *“expiresIn”* defines *“exp”* parameter. If an error is returned in the callback server is sending a **Forbidden (403) code**.

When JWT is issued by authentification server and client send **request** to resouse server with the token attached, we need to verify it:
```
jwt.verify(req.token, 'privatekey', (err, authorizedData) => {
	if(err){
		console.log('ERROR: Could not connect to the protected route');
		res.sendStatus(403);
	} else {
		res.json({
		message: 'Successful log in',
		authorizedData
	}); 
	console.log('SUCCESS: Connected to protected route');
	}
```
As for **expiration date**, you might need to implement some checking procedure. If token is expired you can ask user to login again, or create another access token after receiving **refresh token**. The first approach is more secure: for example, after few minutes of inactivity your bank account client will redirect you to login page.

And finally, the *AuthorizedData* parameter contains all of the protected data that we requested. And that’s how it all works, folks!
