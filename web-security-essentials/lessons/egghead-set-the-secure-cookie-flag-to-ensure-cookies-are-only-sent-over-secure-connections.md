[0:00] We've set up our site to serve traffic over https, and to redirect from http to https. The problem now is, we do this, we're still submitting the cookie over http. The reason this is is that cookies are set by default to send over both https and http. Luckily, there's a property of cookies called `secure`. If we pass secure when we set our cookie, it will only send over https.

[0:27] To do this in Express, we go down to our cookie settings for our session cookie, the `app.use`, and enter in `secure: true`. 

### index.js
```js
app.use(
    session({
        secret: "key",
        resave: false,
        saveUninitialized: true,
        cookie: {
            secure: true, 
            httpOnly: false
        }
    })
)
```

Hitting save will cause our server to reload, and we'll go ahead and, in our browser, clear our cookies completely, and go back to the network tab. Now, refresh the page. We can see that the response from the server, when it sets the cookie, now includes secure as part of the site cookie command.

![secure](https://res.cloudinary.com/dg3gyk0gu/image/upload/v1581384983/transcript-images/egghead-set-the-secure-cookie-flag-to-ensure-cookies-are-only-sent-over-secure-connections-secure.png)

[1:06] Regardless of how you accomplish this, whether using Express or any other framework, the important bit is that the site cookie header must have the secure property set on it. If we now once again enter in a http URL, we'll see that the http request no longer passes the cookie in the request headers. We now have a site that will redirect from http to https, but will no longer send cookies on http.