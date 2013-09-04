Dashku - a developer's guide
===

_Paul Jensen, 4th August 2013_

Introduction
---

Dashku is an open source dashboard tool. It is available to [download on GitHub](https://github.com/Anephenix/dashku).

This guide is for developers who want to use Dashku within their organisation, and extend it to suite their needs.

Knowledge required
---

Dashku uses a set of technologies that are listed below:

- [Node.js](http://nodejs.org)         (the underlying programming framework)
- [SocketStream](https://github.com/socketstream/socketstream)    (a realtime single page app web framework)
- [CoffeeScript](http://coffeescript.org)    (a Ruby/Python flavoured language that compiles into JavaScript)
- [Jade](http://jade-lang.com/)      (a html template engine, comparable to Slim/Haml in Ruby)
- [Stylus](http://learnboost.github.io/stylus/) (CSS stylesheets with variables, similar to Less/SASS) 
- [MongoDB](http://mongodb.org)         (a document store database)
- [Redis](http://redis.io)           (a key/value store database)

Getting Started
---

The first action that I can recommend is to get a copy of the codebase from Github, and foliow the installation instructions in the Readme file.

Install the app, boot it up, and play with it for a bit. Once you've done that, I'll guide you in the next section through the codebase.

Dashku's structure explained
--- 

The first place to start is to look at the app.coffee file in the root folder:



    connectRoute      = require 'connect-route'
    http              = require 'http'
    ss                = require 'socketstream'
    internals         = require './server/internals'
    
    # Define a single-page client
    ss.client.define 'main',
      view: 'app.jade'
      css:  ['libs', 'app.styl']
      code: ['libs', 'app']
      tmpl: '*'
    
    api = require "#{__dirname}/server/api"
    
    ss.http.middleware.prepend ss.http.connect.bodyParser()
    ss.http.middleware.prepend ss.http.connect.query()
    ss.http.middleware.prepend connectRoute api
    
    # Serve this client on the root URL
    ss.http.route '/', (req, res) -> res.serveClient 'main'
    
    # Use redis for session store
    ss.session.store.use 'redis', ss.api.app.config.redis
    
    # Use redis for pubsub
    ss.publish.transport.use 'redis', ss.api.app.config.redis
    
    # Code Formatters
    ss.client.formatters.add require 'ss-coffee'
    ss.client.formatters.add require 'ss-jade'
    ss.client.formatters.add require 'ss-stylus'
    
    # Use server-side compiled Hogan (Mustache) templates. Other engines available
    ss.client.templateEngine.use require 'ss-hogan'
    
    # Minimize and pack assets if you type: SS_PACK=1 SS_ENV=production node app.js
    ss.client.packAssets ss.api.app.config.packAssets if ss.env is 'production'
    
    # Start SocketStream
    server = http.Server ss.http.middleware
    server.listen ss.api.app.config.port
    ss.start server
    
    # So that the process never dies
    process.on 'uncaughtException', (err) -> console.error 'Exception caught: ', err

__Dependencies__

Let's take a look at the first couple of lines of code:

    connectRoute      = require 'connect-route'
    http              = require 'http'
    ss                = require 'socketstream'
    internals         = require './server/internals'

In the first couple of lines, we're loading Node.js libraries that we depend upon via [npm](http://npmjs.org).

The first line requires a node module used for supporting Dashku's REST API. [Connect-Route](https://github.com/baryshev/connect-route) is a custom ConnectJS middleware that adds support for dynamic routes.

The 2nd line requires [Node's http library](http://nodejs.org/api/http.html), which we use to host the SocketStream app.

The 3rd line requires the SocketStream library. This is the underlying web framework behind Dashku, and it's worth playing with SocketStream separate to Dashku in order to understand what SocketStream provides for realtime single page web apps.

The 4th line loads an app-specific file called "internals". This file is responsible for loading the database connections, as well as the controllers and models underpinning the app. The reason that it is specified as a separate file to the app.coffee file is so that we could load the app without booting a http server, for example for use in a REPL.

TODO - Provide a link to a breakdown of what the internals file looks like

__Clients__

Let's look at the next set of lines in the file.

        # Define a single-page client
        ss.client.define 'main',
          view: 'app.jade'
          css:  ['libs', 'app.styl']
          code: ['libs', 'app']
          tmpl: '*'

In SocketStream, a client is a collection of the front-end assets used to display and run a given page. It is used to let SocketStream do the following:

- Know which files to watch for changes, so that you can display the effects of those changes without having to reload the page yourself.
- Know what order to load your css and javascript assets, especially in the context of dependencies.
- Know what css and javascript files to concatenate and minify for use in production mode. 

The client looks for files stored in the client directory, and loads files based on what type they are. So for example, the view file (app.jade) exists at client/views/app.jade. In the case of the css files, they live in the client/css folder, and for the javascript (code), they live in the client/code folder. The html template files live in the client/templates folder.

You'll notice in the code that we sometimes load a folder called 'libs', but don't specify the order in which the files inside that folder load. SocketStream will load the files inside that folder in alphabetical order.

__API__

The REST API is loaded via this line:

    api = require "#{__dirname}/server/api"

We will take a look at that file later TODO - provide link to breakdown of what the file contains.

You'll notice these lines as well:

    ss.http.middleware.prepend ss.http.connect.bodyParser()
    ss.http.middleware.prepend ss.http.connect.query()
    ss.http.middleware.prepend connectRoute api

SocketStream provides an API to prepend/append connect middleware to the stack, around the set of middleware that SocketStream loads in by default. The lines above are used for adding connect middleware that take care of the following:

- Parsing HTTP body data (for POST/PUT requests)
- Parsing HTTP query parameters (for identifying RESTful resources)
- loading the REST API into the connect-route middleware

These middleware items are loaded before SocketStream's default middleware stack, which is the following:

- Connect CookieParser
- Connect FavIcon
- Connect SessionStore

If you wish to add middleware items into the stack _after_ the default stack, then call the middleware like this:

    ss.http.middleware.append(MIDDLEWARE);

__Serving clients__

SocketStream includes a basic HTTP server binding that handles GET requests at given routes, like this:

    # Serve this client on the root URL
    ss.http.route '/', (req, res) -> res.serveClient 'main'

The line above tell's SocketStream's wrapper around Node's HTTP API to serve a specific client called 'main'. The serveClient function is appended to the HTTP Response object by SocketStream, and you'll notice that the value 'main' matches the name of the client we specified earlier on in the app.coffee file.

__Session Storage__

    # Use redis for session store
    ss.session.store.use 'redis', ss.api.app.config.redis

SocketStream provides 2 options for how you store session data: in memory, or in Redis. Here, we use the Redis session store, and pass the Redis configuration to the session store. The way in which we access this configuration is called via SocketStream's API object, which allows us to bind collections of functions and data onto SocketStream for use in other parts of our application.

The reason to store session data in Redis is so that you can persist the data across multiple instances of Dashku, as well as to maintain session state if the process running Dashku is terminated (i.e for deployments). 

__Publish/Subscribe__

    # Use redis for pubsub
    ss.publish.transport.use 'redis', ss.api.app.config.redis

SocketStream has 2 ways that you can support publish/subscribe: via in-memory, or via Redis. If you're running Dashku as a single process, then in-memory is fine, but the moment that you need to run Dashku on multiple processes, then you will want to use Redis to handle SocketStream's publish/subscribe data transmission.

__Code Formatters__

SocketStream supports writing your application using langauges that compile into other formats. Dashku has been written using a combination of CoffeeScript, Jade, and Stylus. In order to support the use of these code formatters, we require some npm modules that handle converting the pre-processed code into it's web format (CoffeeScript into JavaScript, Jade into HTML, Stylus into CSS):

    # Code Formatters
    ss.client.formatters.add require 'ss-coffee'
    ss.client.formatters.add require 'ss-jade'
    ss.client.formatters.add require 'ss-stylus'

You be interested to know that other variants exist: ss-gorilla is available for the GorillaScript language, ss-coffeekup for HTML templates, and ss-less for using Less as a CSS preprocessor.

__Client-Side HTML Template engine__

    # Use server-side compiled Hogan (Mustache) templates. Other engines available
    ss.client.templateEngine.use require 'ss-hogan'

To support the Single Page App setup, client-side HTML templates are compiled on the server side, and sent up to the client as functions that can be called with data for injecting into the template. In our case, we use a wrapper of Twitter's Hogan.js library to handle the compilation of client-side HTML templates. Later on we will show you how this is used in the app.

__Asset Packing__

When you want to serve Dashku in production, we call a code feature in SocketStream called Asset Packing. Asset Packing will do the following things:

- Concatenate your CSS and JavaScript files into single files, so that your app gets served in as few HTTP requests as possible.
- Use GZIP for reducing the file size of those files, so that they download in a shorter period of time.
- Dynamically handle changing the link and script tags inside the html head tag, so that they load these files.


    # Minimize and pack assets if you type: SS_PACK=1 SS_ENV=production node app.js
    ss.client.packAssets ss.api.app.config.packAssets if ss.env is 'production'

The line above will call ss.client.packAssets if the app is running in production mode, using the packAssets configuration as detailed in the config.coffee file in the root folder.

__Starting the server__

The next 3 lines (after the comment line) handle starting the SocketStream application:

    # Start SocketStream
    server = http.Server ss.http.middleware
    server.listen ss.api.app.config.port
    ss.start server

The 1st line loads a Node HTTP server with SocketStream's collection of connectJS middleware. The 2nd line makes the server listen on a prot number, specified in SocketStream's server/config.coffee file. The 3rd line tells SocketStream to start with the HTTP server loaded. This handles binding the Websocket server onto the HTTP server.

__Exception handling__

In order to prevent the Node.js process from exiting on an error, this line is used:

    # So that the process never dies
    process.on 'uncaughtException', (err) -> console.error 'Exception caught: ', err

This is a naive form of error handling which will intercept any unhandled errors that occur in the application. If you have setup your app to run with a service monitoring tool like upstart, then you won't need it.