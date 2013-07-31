
# juju charm to deploy a user-defined node.js app

This is an example of how to use mongodb with mongoosejs in a node.js cloud app.
It is part of a tutorial on development to deployent on the cloud.
Clone this repo into to ~/charms/precise

[juju](http://juju.ubuntu.com)
charm to deploy a user-defined node app directly from revision control.

    - App server file must be named 'server.js'.

# Using this charm

First, edit `config.yaml` to add info about your app.

Then deploy some basic services

    juju deploy --repository ~/charms local:node-app mongonode-app
    juju deploy mongodb
    juju deploy haproxy

relate them

    juju add-relation mongodb mongonode-app
    juju add-relation mongonode-app haproxy

scale up your app (to 10 nodes for example)

    juju add-unit -n 10 mongonode-app

open it up to the outside world

    juju expose haproxy

Find the haproxy instance's public URL from 

    juju status

(or attach it to an elastic IP via the aws console)
and open it up in a browser.


## What the formula does

During the `install` hook,

- installs `node`/`npm`
- clones your node app from the repo specified in `app_repo`
- runs `npm` if your app contains `package.json`
- configures networking if your app contains `config/config.js`
- waits to be joined to a `mongodb` service

when related to a `mongodb` service, the formula

- configures db access if your app contains `config/config.js`
- starts your node app as a service


## Charm configuration

Configurable aspects of the charm are listed in `config.yaml`
and can be set by either editing the default values directly
in the yaml file or passing a `myapp.yaml` configuration
file during deployment

    juju deploy --config ~/config.yaml node-app mongonode-app

Some of these parameters are used directly by the charm,
and some are passed through to the node app using `config/config.js`.

## Application configuration

The formula looks for `config/config.js` in your app which
starts off looking something like this

    module.exports = config = {
      "name" : "mongonode"
      ,"listen_port" : 8000
      ,"mongo_host" : "localhost"
      ,"mongo_port" : 27017
    }


and gets modified with contextually correct configuration information during
either deployment (via the `install` hook) or relation to another service 
(`relation-changed` hook).

This config can be used from within your 
application using snippets like with mongodb

    ...
    var config = require('./config/config')
    ...
      new mongo.Server(config.mongo_host, config.mongo_port, {}),
    ...
    server.listen(config.listen_port);
    ...

Alternatively you could use a "Procfile" in root directory like this:

    web: node server.js

and then get the environment variables from the running environment like this:

    app.set('port', process.env.PORT);

The defined environment variables are:

    NAME
    PORT
    NODE_ENV
    MONGO_HOST
    MONGO_PORT
    MONGO_REPLSET

## Network access

This charm does not open any public ports itself.
The intention is to relate it to a proxy service like
`haproxy`, which will in turn open port 80 to the outside world.
This allows for instant horizontal scalability.

If your node app is itself a proxy and you want it directly exposed,
this can easily be done by adding 

    open-port $app_port

to the bottom of the `install` hook, and then once your stack
is started, you expose

    juju expose mongonode-app

it to the outside world.

By default, juju services within the same environment
can talk to each other on any port over
internal network interfaces.


# Making this work with your node.js app

This charm makes some strong assumptions
about the structure of the node application 
(`config/config.js`) that might not apply to your app.
Please treat this formula as a template that 
you can fork and modify to suit your needs.

The biggest difference between how the charm
behaves for different kind of apps is application
startup.  A simple application will want to start
upon install (startup code goes in the `install` hook),
whereas some applications will not want
to start up until a database has be associated
(startup code goes in the `db-relation-joined` hooks).

