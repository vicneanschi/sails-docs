# Deployment

### Before You Deploy

Before you launch any web application, you should ask yourself a few questions:

+ What is your expected traffic?
+ Are you contractually required to meet any uptime guarantees, e.g. a Service Level Agreement (SLA)?
+ What sorts of front-end apps will be "hitting" your infrastructure?
  + Android/iOS/Windows Phone apps
  + desktop web browsers
  + mobile web browsers (tablets, phones, iPad minis?)
  + Cordova/Electron apps
  + tvs, watches, toasters..?
+ And what kinds of things will they be requesting? (e.g. HTML? JSON? XML?)
+ Will you be taking advantage of realtime pubsub features with Socket.io?
  + e.g. chat, realtime analytics, in-app notifications/messages
+ How are you tracking crashes and errors?
  + Are you using `sails.log()`? Or are you using a custom logger from NPM like [Winston](https://github.com/winstonjs/winston)?



### Configuring Your App For Production

You can provide configuration which only applies in production in a [few different ways](http://sailsjs.org/documentation/reference/configuration).  Most apps find themselves using a mix of environment variables and `config/env/production.js`.  But regardless of how you go about it, this section and the [Scaling section](http://sailsjs.org/documentation/concepts/deployment/scaling) of the documentation cover the configuration settings you should review and/or change before going to production.



### Deploying On a Single Server

Node.js is pretty darn fast.  For many apps, one server is enough to handle the expected traffic-- at least at first.

> This section focuses on _single-server Sails deployment_.  This kind of deployment is inherently limited in scale.  See [Scaling](http://sailsjs.org/documentation/concepts/deployment/scaling) for information about deploying your Sails/Node app behind a load balancer.

Many teams decide to deploy their production app behind a load balancer or proxy (e.g. in a PaaS like Heroku or Modulus, or behind an nginx server).  This is often the right approach since it helps future-proof your app in case your scalability needs change and you need to add more servers.  If you are using a load balancer or proxy, there are a few things in the list below that you can ignore:

+ don't worry about configuring Sails to use an SSL certificate.  SSL will almost always be resolved at your load balancer/proxy server, or by your PaaS provider.
+ you _probably_ don't need to worry about setting your app to run on port 80 (if not behind a proxy like nginx). Most PaaS providers automatically figure out the port for you.  If you are using a proxy server, please refer to its documentation (whether or not you need to configure the port for your Sails app depends on how you set things up and can vary widely based on your needs).

> If your app uses sockets and you're using nginx, be sure to configure it to relay websocket messages to your server. You can find guidance on proxying WebSockets in [nginx's docs on the subject](http://nginx.org/en/docs/http/websocket.html).


##### Set the `NODE_ENV` environment variable to `'production'`

Configuring your app's environment config to `'production'` tells Sails to get its game face on; i.e. that your app is running in a production environment.  This is, hands down, the most important step. If you only have the time to change _one setting_ before deploying your Sails app, _this should be that setting_!

When your app is running in a production environment:
  + Middleware and other dependencies baked into Sails switch to using more efficient code.
  + All of your [models' migration settings](http://sailsjs.org/documentation/concepts/models-and-orm/model-settings) are forced to `migrate: 'safe'`.  This is a failsafe to protect against inadvertently damaging your production data during deployment.
  + Your asset pipeline runs in production mode (if relevant).  Out of the box, that means your Sails app will compile all stylesheets, client-side scripts, and precompiled JST templates into minified `.css` and `.js` files to decrease page load times and reduce bandwidth consumption.
  + Error messages and stack traces from `res.serverError()` will still be logged, but will not be sent in the response (this is to prevent a would-be attacker from accessing any sensitive information, such as encrypted passwords or the path where your Sails app is located on the server's file system)


>**Note:**
>If you set [`sails.config.environment`](http://sailsjs.org/documentation/reference/configuration/sails-config#?sailsconfigenvironment) to `'production'` some other way, that's totally cool.  Just note that Sails will set the `NODE_ENV` environment variable to `'production'` for you automatically.  The reason this environment variable is so important is that it is a universal convention in Node.js, regardless of the framework you are using.  Built-in middleware and dependencies in Sails _expect_ `NODE_ENV` to be set in production-- otherwise they use their less efficient code paths that were designed for development use only.




##### Configure your app to run on port 80

Whether it's by using the `sails_port` environment variable, setting the `--port` command-line option, or changing your production config file(s), add the following to the top level of your Sails config:

```javascript
port: 80
```

> As mentioned above, ignore this step if your app will be running behind a load balancer or proxy.



##### Set up production database(s) for your models

To set up one or more production databases, configure them in [`sails.config.connections`](http://sailsjs.org/documentation/reference/configuration/sails-config-connections), and then refer to them from [`sails.config.models.connection`](http://sailsjs.org/documentation/reference/configuration/sails-config-models) and/or from individual models.  For most apps, your production config changes are pretty simple:
1. add a connection representing your production database (e.g. `productionPostgresql: { ... }`)
2. override the default connection (`sails.config.models.connection`) to point to your production database (e.g. `productionPostgresql`)

If your app is using more than one database, your process will be similar.  However, you will probably find that it's easier to override existing connection settings rather than adding new connections and changing individual models to point at them. Regardless how you go about it, if you are using multiple databases you should be sure your models are pointed at the right connections when you deploy to production.

Keep in mind that if you are using version control (e.g. git), then any sensitive credentials (such as database passwords) will be checked in to the repo if you include them in your app's configuration files.  A common solution to this problem is to provide certain sensitive configuration settings as environment variables.  See [Configuration](http://sailsjs.org/documentation/concepts/configuration) for more information.

If you are using a relational database such as MySQL, there is an additional step.  Remember how Sails sets all your models to `migrate:safe` when run in production?  That means no auto-migrations are run when lifting the app...which means by default your tables won't exist.  A common approach to deal with this during the first-time setup of a relational database for your Sails app goes as follows:
  + Create the database on the production database server (e.g. `frenchfryparty`)
  + Configure your app locally to use this production database, but _don't set the environment to `'production'`, and leave your models' configuration set to `migrate: 'alter'`_.  Now run `sails lift` **once**-- and when the local server finishes lifting, kill it.
    + **Be careful!**  You should only do this when there is _no data_ in the production database.

If this makes you nervous or if you can't connect to the production database remotely, you can skip the steps above.  Instead, simply dump your local schema and import it into the production database.


##### Enable CSRF protection

Protecting against CSRF is an important security measure for Sails apps.  If you haven't already been developing with CSRF protection enabled (see [`sails.config.csrf`](http://sailsjs.org/documentation/reference/configuration/sails-config-csrf)), be sure to [enable CSRF protection](http://sailsjs.org/documentation/concepts/Security/CSRF.html?q=enabling-csrf-protection) before going to production.



##### Enable SSL

If your API or website does anything that requires authentication, you should use SSL in production.  To configure your Sails app to use an SSL certificate, use [`sails.config.ssl`](http://sailsjs.org/documentation/reference/configuration/sails-config).

> As mentioned above, ignore this step if your app will be running behind a load balancer or proxy.



##### Lift Your App

The last step of deployment is actually starting the server.  For example:

```bash
NODE_ENV=production node app.js
```

Or if you're more comfortable with command-line options you can use `--prod`:

```
node app.js --prod
# (Sails will set `NODE_ENV` automatically)
```

As you can see, instead of `sails lift` you should start your Sails app with `node app.js` in production.  This way, your app does not rely on having access to the `sails` command-line tool; it just runs the `app.js` file bundled in your Sails app (which does exactly the same thing).


##### ...And Keep It Lifted

Unless you are not deploying to a PaaS like Heroku or Modulus, you will want to use a tool like [`pm2`](http://pm2.keymetrics.io/) or [`forever`](https://github.com/foreverjs/forever) to make sure your app server will start back up if it crashes.  Regardless of the daemon you choose, you'll want to make sure that it starts the server as described above.

For convenience, here are example lift commands for both `pm2` and `forever`:

Using `pm2`:

```bash
pm2 start app.js -x -- --prod
```

Using `forever`:

```bash
forever start app.js --prod
```




<docmeta name="displayName" value="Deployment">
