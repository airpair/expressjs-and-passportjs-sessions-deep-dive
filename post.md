Synopsis
> There are major optimizations (and bottlenecks) you can spot by passing
through your `ExpressJS Middleware` with a fine tooth comb. I was originally 
stuck for 6 hours on a middleware issue while settings up PassportJS the second
time round after noticing 10,000,000+ sessions in AirPair's MongoDB production
instance in late 2014. There's good stuff in this deep dive for all levels of
Node.js developers building on express. The main take away is **watch out for the order you add ExpressJS Middleware** to your app. As I'll cover more than once below, a tiny swap in middleware order can have HUGE consequences.

<sub>First published Oct 2014, Updated Aug 2015, intended to significantly 
enhance on discussion around bots, scrappers & analytics in Nov 2015.</sub>

## ExpressJS & PassportJS 101

Though I'd setup ExpressJS and PassportJS in 2013 for the *v0* version
of airpair.com, I didn't deeply understand each or their relationship with
one another. Around October 2014 I noticed some 10,000,000 active session documents in MongoDB, which was obviously not right. 

Luckily it never affected us in a material way, and at the time I didn't want to 
take my chances. So I spent some time observing how Express and PassportJS plug 
into each other, here's what I learned on the way to uncovering what was going
wrong.

### There is only ONE Session

As per the PassportJS docs<sup>The official [PassportJS docs](http://passportjs.org/guide/configure/)</sup> configuring Passport via
express middleware looks something like this:

<!--?prettify lang=javascript linenums=false?-->

    app.configure(function() {
      app.use(express.static('public'));
      app.use(express.cookieParser());
      app.use(express.bodyParser());
      app.use(express.session({ secret: 'keyboard cat' }));
      app.use(passport.initialize());
      app.use(passport.session());
      app.use(app.router);
    });

The syntax is a bit misleading I think ... 

The first thing to conceptually get your head around is that even though you configure `express.session()` and`passport.session()`, there is really only one session, which is managed by Express. Passport merely piggy backs off the ExpressJS session to store data for authenticated users. 

Let's take a look at how:

### ExpressJS Sessions & `req.session`

`req.session` is just a json object that gets persisted by the `express-session` middleware, using a store of your choice e.g. Mongo or Redis. Logging to the terminal, you'll see the default session object looks something like:

<!--?prettify lang=javascript linenums=false?-->

    req.session = { cookie: 
      { path: '/',
        _expires: Thu Nov 07 2014 09:39:45 GMT-0700 (PDT),
        originalMaxAge: 100000,
        httpOnly: true } }

If you open up your persistence store and look at the documents, you'll see the req.session object is an attribute of the the entire document that also comes with an `id` and `expires` attribute.

<!--?prettify lang=javascript linenums=false?-->

    {
        _id : "svZGs99EIvDkwW_60PRwlafU9Y_7_m-N",
        session : { cookie:{originalMaxAge:100000,expires:"2014-11-07T02:11:16.320Z",httpOnly:true,path:"/"},
        expires : ISODate("2014-11-07T02:11:16.320Z")
    }

Express stuffs the id of the session object into a cookie on the client's browser, which gets passed back to express in a header on every request. This is how Express identifies multiple requests as belonging to a single session even if the user is not logged in. With the id from the cookie header, Express reads the session store and puts all the info onto req.session, which is available to you on each request.

#### Harnessing `req.session` yourself

You can stuff anything you like onto the req.session object and Express will automatically persist it back to the session store for a given session (unique id). For example, if a user can't access a page because they are not logged in, airpair.com uses a custom made-up attribute called `return_to` to direct the user to the right page after login. 

<!--?prettify lang=javascript linenums=true?-->

    { cookie: 
      { path: '/',
        _expires: Thu Oct 09 2014 09:39:45 GMT-0700 (PDT),
        originalMaxAge: 100000,
        httpOnly: true },
        passport: { user: { _id: '5175efbfa3802cc4d5a5e6ed' } },
        return_to: '/workshops' }

It's worth noting that since you can put anything you want into an anonymous session, provided you can correlate it to a user in your database on login or signup, you can start persisting info for a user even though you don't know who they are straight away. You will see some cool interactions for anonymous users on airpair.com coming in the next month or two.

#### Passport also harnesses `req.session`

As you can see on line 6 in the code snippet above, using the `passport` attribute allows PassportJS to use the session object to keep track of a logged in user  associated with a given session. 

It then uses the `deserializeUser` function, which receives `req.session.passport.user` as its first parameter, and as the default behavior suggested in the PassportJS documentation, makes an additional read to your persistence store to retrieve the user object associated with the userId.

<!--?prettify lang=javascript linenums=false?-->

    passport.serializeUser(function(user, done) {
      done(null, user.id);
    });

    passport.deserializeUser(function(id, done) {
      User.findById(id, function(err, user) {
        done(err, user);
    });

### Passport `req.user`

`req.user` is a PassportJS specific property that is the result of the `deserializeUser` function using the data from `req.session.passport.user`

## Optimize PassportJS 

I realized that in the old app, we followed the default suggestion and were hitting the database twice on every single API call to populate all the user's information in memory. But in practice, we rarely needed more than the userId in our backend code. So this time around, I've made the decision to stuff the name and email into the session object and avoid making multiple database trips on every single API call. With many pages on the site making 5-10 calls to render a single page, this seemed like a cheap way to significantly reduce database load. Here's what the new app looks like:

<!--?prettify lang=javascript linenums=false?-->

    passport.serializeUser( (user, done) => {
      var sessionUser = { _id: user._id, name: user.name, email: user.email, roles: user.roles }
      done(null, sessionUser)
    })

    passport.deserializeUser( (sessionUser, done) => {
      // The sessionUser object is different from the user mongoose collection
      // it's actually req.session.passport.user and comes from the session collection
      done(null, sessionUser)
    })

## Optimize ExpressJS Sessions

### When are sessions created?

Express will create a new session (and write it to the database) whenever it
does not detect a session cookie. As it turns out, the order in which you set the session
middleware and tell Express where your static directory is has some pretty
dramatic nuances. Here's what the new AirPair index.js looks like:

<!--?prettify lang=javascript linenums=false?-->

    var express = require('express')
    var app = express()
	
    //-- We don't want to serve sessions for static resources
    //-- Save database write on every resources
    app.use(express.static(config.appdir+'/public'))

    mongo.connect()
    session(app, mongo.initSessionStore)

    //-- Do not move connect-livereload before session middleware
    if (config.livereload) app.use(require('connect-livereload')({ port: 35729 }))
	
    hbsEngine.init(app)
    routes.init(app)	
    app.listen(config.port, function() {})

### Avoid Sessions for Static Resources

I learned that if you add the session middleware before your static directory, 
Express will generate sessions for requests on static files like stylesheets, images, and JavaScript. 

If a new visitor without a session loads a page with 10 static files, the 
client's browser will not yet have a cookie and will send 10 cookieless requests 
all triggering Express to create sessions. Ouch! So that's what was happening... 
If you haven't done something smart to detect bots and scrapers, things can blow
out pretty quickly!

Simply put your static files first, or better yet on a CDN that has nothing to
do with your Node.js app and your session collection should stay much healthier.

## ExpressJS 4.0 Middleware Order

The middleware setup for ExpressJS 4.0 is quite different from ExpressJS 3.0 where everything came baked in. You now need to include each piece manually with its own NPM package. In case you want to see an up-to-date version of how each  piece is chained together, because there are many non-working legacy examples floating around, this is what I ended up with.

A couple of gotchas sunk half an hour or so for me, including that Cookie Parser now requires a secret and Body Parser required and Extend Url.

<!--?prettify lang=javascript linenums=false?-->

    // Passport does not directly manage your session, it only uses the session.
    // So you configure session attributes (e.g. life of your session) via express
    var sessionOpts = {
      saveUninitialized: true, // saved new sessions
      resave: false, // do not automatically write to the session store
      store: sessionStore,
      secret: config.session.secret,
      cookie : { httpOnly: true, maxAge: 2419200000 } // configure when sessions expires
    }

    app.use(bodyParser.json())
    app.use(bodyParser.urlencoded({extended: true}))
    app.use(cookieParser(config.session.secret))
    app.use(session(sessionOpts))

    app.use(passport.initialize())
    app.use(passport.session())


### The AHA! **live-reload** middleware

So it turns out, the problem that held me up was the position of the live-reload middleware. LiveReload injects script into every response to listen for changes emitted from the server. I don't know the exact issue, but having it before the session middleware broke the session cookie being sent correctly to the client.

## Check your Middleware!

I'm glad I took the time to dive deep with Express and Passport for the new 
airpair.com site. A few tweaks have lead to significantly less database traffic 
and my deepening understanding of how to utilize req.session will empower us to 
build some cool interactions and personalization for anonymous visitors.

I highly highly recommend going through all your middleware one at a time
to understand everything you app is doing on every request. There are many performance gains to be had when you understand things
piece by piece.
