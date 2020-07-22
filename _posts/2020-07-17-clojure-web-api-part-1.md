---
title: "API Server in Clojure - Part 1"
published: true
---

# API Server in Clojure - Part 1
---

Disclaimer: I am new to Clojure.

Now that everyone has read the disclaimer, some background on myself. 

By trade, I am a web developer. I primarily use Java, Spring Boot, Angular, and TypeScript. I was introduced to Clojure by a fellow teammate and I was instantly hooked. 

Being a web developer I thought the best way to learn Clojure was to create an API server. Needless to say, it took **a lot** of scouring the internet, using the fantastic [Clojure Slack community](http://clojurians.net/), and lots of reading of dispersed blog posts to be able to glue all the pieces together. 

I finally understood how to get an API server up and running but I also understood a lot of people's frustration when trying to get into Clojure. Specifically web development with Clojure. There is no "one-stop-shop, show me how". 

So here we are. I am going to, hopefully, give all new Clojure peeps a resource that will help them get an API server up and running with a database. 

---

You will be using: Clojure (duh!), Leiningen, Compojure, Ring, Component, and HoneySQL. Don't worry if you do not know what all these things do, you will be by the end of this blog post. 

This is part of a three-part series on creating an API server with routes and database interactivity. 

Grab a glass of water (or your favorite beverage), a comfy chair, and let's get to work. 

## Series
1. API Server in Clojure - Part 1
2. API Server in Clojure - Part 2
3. API Server in Clojure - Part 3

## JDK & Leiningen 
Before you do anything you need to install a Java Development Kid (JDK). I prefer using [SDKMAN!](https://sdkman.io/) to manage my JDKs. 

After this, you will need to install Leiningen. Leiningen is an automation tool used for creating and managing Clojure projects. Instructions can be found on the [official Leiningen website](https://leiningen.org/). 

## Creating a Clojure application
You will need to create your Clojure application using Leiningen. Change into the directory where you want your project to live and then run the following command:
```
lein new app your-app
```
Leiningen will create a base Clojure project for you to work in. Your project structure should look similar to this
```
your-app
└─doc
└─resources    
└─src
│   └─your-app
|       └─core.clj 
└─test
└─.gitignore
└─.hgignore
└─CHANGELOG.md
└─LICENSE
└─project.clj
└─README.md
```

Now that you have your application, let's install the dependencies you will need to get your API server up and running.

## Installing Dependencies
There are several dependencies you will need to set up your API server for success. Below is a quick blurb for each and then instructions on where these dependencies go in your project.  

### [Ring - HTTP Server](https://github.com/ring-clojure/ring)
A wildly used HTTP server library is Ring. It is a library with all the standard features you will need to get an HTTP server up and running. 

### [Compojure - Routing](https://github.com/weavejester/compojure)
Compojure will allow you to build out robust routes for your API server with all the standard RESTful methods like `POST`, `GET`, `PUT`, `PATCH`, and `DELETE`. 

### [Component - Lifecycle Management](https://github.com/stuartsierra/component)
This library allows us to manage some lifecycles that you will want in your application. The one you will want to use is when the application boots up, you want to create an HTTP server and when the HTTP server is created, you want to create a connection to our database. 

### [PostgreSQL](https://www.postgresql.org/), [JDBC Connection](https://github.com/seancorfield/next-jdbc), & [Honey SQL](https://github.com/seancorfield/honeysql)
For this project you will be using PostgreSQL. Follow the link in the header to get it installed on your machine. 

Honey SQL helps you to generate SQL queries without having to handroll all your queries into strings. It is a fantastic DSL. It will also make your code more readable.


To add dependencies you will need to open the `project.clj` file and you will see a key called `:dependencies`. This data structure is where you will add any dependencies you will need for your project.
```
File: project.clj

(defproject your-app "0.1.0-SNAPSHOT
   ...
   :dependencies [[org.clojure/clojure "1.10.1]
                  [ring "1.8.1"]
                  [compojure "1.6.1"]
                  [honeysql "1.0.444"]
                  [seancorfield/next.jdbc "1.0.13"]
                  [org.postgresql/postgresql "42.2.10.jre7"]
                  [com.stuartsierra/component "0.4.0"]])
```

Next, you will create a component that will be in charge of starting and stopping the server. 

## Spin up a Server

Create a new directory called `system`. The path to this newly created directory should be `src/your-app/system`. Within this directory create a new file called `server.clj`. In this file, you will create the HTTP server component. 

Create a [namespace](https://clojure.org/reference/namespaces) for this Clojure file. You will need to import two additional namespaces from the dependencies you installed earlier. Those will be the jetty adapter that is bundled with Ring and the component namespace. Your namespace should look something like this
```
File: system.clj

(ns your-app.system.server
  (:require [ring.adapter.jetty :as jetty]
            [com.stuartsierra.component :as component]))
```

Now, let's create two functions: `start-server` and `stop-server`. These will be helper functions that your component will call in it's `start` and `stop` lifecycle functions. 

```
File: system.clj

(defn start-server [port]
  "Helper function to start the server when the component's start function is called"
  (let [server (jetty/run-jetty handler {:port (Integer. port)})]
    server))
```
In this function, you are passing a port variable that the jetty server will run on. You might have noticed already that you will be getting an error on the keyword `handler`. The `jetty/run-jetty` requires a handler as its first argument. Ideally, in this argument, you would pass in a handler method for your routes. You will get to that in a bit but for now, let's create a function called `handler` that responds with a map full of mocked request data. It will look like this
```
File: system.clj

(defn handler [request]
  {:status 200
   :headers {"Content-Type" "text/plain"}
   :body "Hello Clojure, Hello Ring!"})
```
The handler will expect an argument but in this temporary example you will not be using the argument. 

Now for the `stop-server` function
```
File: system.clj

(defn stop-server 
  "Helper function to stop the server when the component's stop function is called"
  [server]
  (when server
    (dissoc server :server)))
```
In this function, you will want to disassociate the server when the application is stopped. 


Component time. You will now create a component that will call these two helper functions. 
```
File: system.clj

(defrecord WebServer [port]
  component/Lifecycle
  (start [this]
    (assoc this :server (start-server port)))
  (stop [this]
    (stop-server (:server this))
    (dissoc this :server)))
```
This `defrecord` creates a component named `WebServer` that takes in a single argument that is the port number. As you can see, we have called the `start-server` helper function on the `start` function of the component's lifecycle, which is the running HTTP server. Identically you will call the `stop-server` helper function on the `stop` function of the component's lifecycle. 

The final function you will need is a function that kicks off the component. 
```
File: system.clj

(defn web-server 
  "Map web server to component"
  [port]
  (map->WebServer {:port port}))
```
This will be the function that you will call in the main method. Speaking of the main method...let's create one!

Navigate to `core.clj`. You can remove the `foo` function if you'd like. Now you will need to create a main method that calls the `web-server` function within your `system.server` namespace. You will need two namespaces in `core.clj`. The namespace of `system.server` as well as `component` from the dependency you installed earlier. Your namespace should look something like this
```
File: core.clj

(ns you-app.core
  (:require [your-app.system.server :as server]
            [com.stuartsierra.component :as component]))
```
You can now create a main method that will take in a single argument for the port number and then call the function that will kick off the web server component. The method should look something similar to 
```
File: core.clj

 (defn -main [port]
  (-> (component/start (component/system-map :web-server (server/web-server port))))
```
You will use the [thread macro](https://clojure.org/guides/threading_macros) to start the component and create a system map. The system map is a key-value pair of all components that are active. More information can be found about these functions by looking at the repositories provided in the links above. 

Now you should be able to get a server up and running! Head to your terminal and use the command `lein run 4000`. In your browser head to `localhost:4000`. You should see a message of "Hello Clojure, Hello Ring!" from your handler above. 

So you now have an HTTP server up and running. 

Yahoo! 

Stay tuned. In the next part of this series, you will hook up your HTTP server with some routes. 

There is a [GitHub repository](https://github.com/tjb/clojure-web-api-skeleton) that has this in a skeleton ready to go. 


Always remember: Be kind. Be compassionate. Be positive. 