# Session Management

Although we've implemented user account registration and authentication, we have no way of verifying if a user is logged in. HTTP is a stateless protocol. This means that all requests are independent of each other. From the perspective of the server process any given request can be sent by anyone. There's no built-in way to tell who sent a particular request. So, we must implement a session management layer ourselves at the application level.

## Session Management by Example

Say you're going to a fair. To get in you need to pay a $15 fee. So you show up, pay your fee and go in. However, you forgot your phone in your car and leave to get it. This fair didn't give you anything to show that you've already paid and there are so many people that the guy at the entrance doesn't remember you. So when you come back you get stopped and have to pay the $15 fee again. To solve this problem fairs give people a bracelet or stamp their hands so they can prove they already paid if they have to leave and come back. That bracelet is just as good as $15 at that fair because from the perspective of the entrance guard that bracelet proves you have paid $15. This is session management in a nutshell. 

## Terminology

**Authentication** verifies credentials. Essentially asking the question *Are you who you claim to be?*

**Authorization** verifies the current entity has permission to access the resource they are requesting. Essentially asking the question *Do you have permission to access this?*

**Authentication ≠ Authorization**

In security authentication is often abbreviated as **AuthN** and authorization is often abbreviated as **AuthZ**. I'll stick to using the full names to avoid confusion; however, I just want you to be aware of the abbreviations in case you come across them.


## Session IDs

A session ID is an identifier or token that we give a user so that we can use to query the state associated with that user. 

![enter image description here](https://i.imgur.com/9SWDvx2.png)

This is a sequence diagram of how we can handle sessions. When a user successfully authenticates we generate a session ID for them and store any associated session data in our Redis data store. We then send them a cookie with the session ID so they can have a copy of the session ID.

A session ID should be unique, difficult to guess/predict and securely managed.

#### Uniqueness
If the session IDs aren't unique then we're back to the same problem of being unable to tell who sent the request. If Alice and Bob have the session ID `1234` then how can you tell their requests apart? *(you can't)*

#### Difficult to guess/predict
From the server's perspective the session ID is as good as a username and password (that's why we don't require them to log in every single time they make a request). So the ID has to be hard to guess, otherwise anyone could masquerade as another user. For example if I log in and my ID is `1000` then I log out and quickly log back in and see my ID is `1005` I could reasonably assume that session IDs are just being incremented overtime someone logs in (essentially it is just a global counter). So if Alice want to get Bob's ID she could wait until he logs in. Then she could quickly log in herself and see that her ID is `4530`; now she has an upper bound on the ID search space. If she logged in before him and had the ID `2301` then she has both and upper and lower bound. It wouldn't be difficult to brute force the rest. Relying on a timestamp has the same problem since the IDs will be chronological.

#### Securely managing session IDs

We must take care to manage session IDs since, again, they're equivalent to a username/password (at least while the session ID is valid). Fortunately we will implement our sessions with cookies which gives us access to powerful and easy to use security features we can enable in browsers. We should also make sure that a session IDs don't last indefinitely; we should set an expiration date so decrease the window of opportunity for an attacker.

## Cookies

Cookies are just text (like practically everything else we've dealt with). They're part of HTTP and built into browsers. The server sends its response with the `Set-Cookie` header and the browser will set the cookie. Then every-time the browser sends a request to the server it will also send along any cookies. Here's an example of the server's response:

```
HTTP/1.1 200 OK
Content-type: text/plain
Set-Cookie: id=123
Set-Cookie: name=Chris

```

Now when the browser sends a request it will send back:

```
GET /account/chris HTTP/1.1
Host: www.example.org
Cookie: id=123; name=Chris
```

The server can then use the cookie's information to make sure whoever sent the request is authorized to access the resource they're requesting.

However, we should store all of the user's session info in the cookie. First of all the more data we stuff into cookies then the more data the client will use on every request (also the spec places a maximum size on cookies at 4kB). Second the more info we store on the client the more info we have to validate on the server. Even though we sent the cookie what's stopping the client from changing it? (a few things actually but we'll get to that later and anyway we want a defense in depth approach). 

For example what if we were using the `name` key in the client's cookie to authorize the client's request. What if the client just changed the cookie to have `name=Alice`. Then they could access Alice's account. Instead we will only store the session ID in the client's cookie and store the session information on the server. Then whenever the client sends a request we can use their session ID to query the session data store and get any data associated with that ID. 

You may be thinking "what if a user gets access to someone else's session ID?".  That's entirely possible and we will employ security practices to mitigate this possibility. Obviously nothing can be 100% secure (for instance what if a user writes their password on a post it note); however, we should protect our users to the best of our ability. One such method is by making the IDs difficult to guess although we'll discuss methods for making them difficult to steal too.


## Storing Session Information

So we have a means of authenticating users and we have a way of sending unique tokens to client's and storing them on the client's browser. However, how do we store the client's session data?

We could use the SQLite database we already have to store session information and the module we will use for session management does support this. However, we need to take performance into account. We are using SQLite to store user information and we query it whenever a user logs in. A user will only log in once per session; however, they will send their cookie and session ID **many** times. The SQLite database requires reading from disk which, as we all know, is slow (at least it's slower than accessing data in main memory). Every-time the user sends a request we will need to query the DB and read from disk to get their session info. We don't want to pay this cost since it well happen **a lot**.

Instead of using SQLite as our session data store we will use Redis. Redis is an in-memory data store. Essentially it's a dictionary data structure that exists in main memory. Since it's in memory it's extremely fast and doesn't require reading from disk. It will also write the data to disk every 2 seconds, so even if the server goes down or the Redis process crashes we can recover the data. 

Redis is a key-value data store, just like javascript objects or python dictionaries. For session management the session ID will be our key and the session information will be the 'value'.

So to access the client's information, we use the session ID they pass in a cookie to query the Redis DB and get back their session information. Then we can use it however we need to because at that point it's just data.

Fortunately express has a session management module. We will configure our redis client in code and pass it to express. From then on express will handle managing the session for us (for the most part). It will expose a `session` object on the `request` object that we can access. For example:

```javascript
app.post("/login", errorHandler( async (req, res) => {
    if (req.body === undefined || (!req.body.username || !req.body.password)) {
        return res.sendStatus(400);
    }
    const {username, password} = req.body;
    const isVerified = await Auth.login(username, password);
    req.session.isAuthenticated = isVerified;
    const status = isVerified ? 500 : 401;
    res.sendStatus(status);
}));
```

Will associate the `isAuthenticated` attribute to that specific client's session ID. Then anytime we get a request we can use:

```javascript
if (req.session.isAuthenticated) {
	res.send(`You are authenticated`);
} else {
	res.send(`You are not authenticated`);
}
```
It's that easy.  Even if the `isAuthenticated` attribute hasn't been set yet, it will be `undefined` which evaluates to false. We will discuss how to configure the redis client and session object we will use in class.
