---
layout: post
title:  "Environment Configuration Settings in Javascript"
date:   2016-11-14 18:20:36
categories: javascript
---
I come from a .Net background and have been writing web applications using ASP.Net MVC since it was released.  One of the things I have taken for granted over the years is how easy configuration management is on server-side web applications.
In ASP.Net your `Web.config` file contains server-side web application settings.  You can create different configuration settings by creating a new build type in the Configuration Manager.  By default, there is a debug config `Web.debug.config` and release config `Web.release.config`. These files are transformation files which allow you to specify which settings you want to override in the `Web.config`. 

When I decided to write a static web site in Javascript using React, I struggled to find a good answer for managing configuration settings.  When hosting a static web site there isn't a way to transform configuration settings on the server so they must be packed with the client application.  
Here is the scenario: each project usually has 4 environments that need to be managed.  
Local development - run the web site on your machine in dev mode and run unit tests against the source files
Dev - production or dev build deployed to a development environment for automated testing usually after every commit 
QA - production build deployed to a production-like environment and controlled by our QA department
Production - production build deployed to a production environment

There are also 4 different API environments, local, dev, QA, and production.  The production build needs to point to one of these three API tiers depending on which environment it is deployed to.

### Development setup
React and Redux
Webpack for to prepare the site for distribution
npm for package management and tasks
ava for unit testing
babel for transpiling ES6 and ES7 features

##### Configuration issues
Our primary configuration issue was the URL for the API.  In local mode we wanted it to be a localhost address.  In dev we wanted to use the dev alias, qa the qa alias, and prod the production URL.
Hardcoding the URL in the javascript files obviously wouldn't work.  

##### Using Webpack
We have a webpack.dev.config and webpack.prod.config.  In dev mode the site from the src folder and a hot reloading feature is injected as a dependency. The production webpack config bundles and minifies the Javascript files and css.

### First Attempt
My first thought was to use webpack features to expose a [Config external](https://webpack.github.io/docs/configuration.html#externals).

To add the QA settings I copied the webpack.prod.config and created a webpack.qa.config.  The only difference was the Config external.
Here is an example of the production external:

{% highlight javascript %}
  externals: {
    'Config': JSON.stringify({ serverUrl: 'https://productionapi.azurewebsites.net' })
  }
{% endhighlight %}

This worked great.  In our Javascript file we referenced the external with an import statement. However when our unit tests ran the import statement would fail. The problem with this was our unit tests couldn't resolve the config import because we weren't packing the build first.  I didn't want to require packing the web site for our unit tests to run locally so I tried something different. 

### Second Attempt
Next, I tried using a [webpack alias](https://webpack.github.io/docs/configuration.html#resolve-alias) and specifying a Javascript file which contained our environment specific settings.

{% highlight javascript %}
      alias: {
          config: path.join(__dirname, 'config', process.argv[2] || 'prod')
      },
{% endhighlight %}

This obviously didn't work for the same reason as the first attempt.  When running tests locally we don't pack the web site.  I did however like this solution more simply because we were able to specify which config we wanted as a parameter in the build.
So this did take care of having a duplicate production webpack config for QA.

### Final Solution
The final solution was to simply go back to good ole' copying files.

I created a config folder and put our configuration settings in json files `dev.json`, `prod.json`, and `qa.json`.  
The trick is to copy one of the environment specific configuration files to a `default.json`.  The javascript code imports the `default.json` file like this:

{% highlight javascript %}
let config = require('../config/default.json');
{% endhighlight %}

For local development I added a prestart step to copy the `dev.json` file before our tests start:

{% highlight javascript %}
"prestart": "npm-run-all --parallel start-message remove-dist copy-config",
"copy-config": "node tools/testSetup.js",
{% endhighlight %}

Here is the testSetup.js:

{% highlight javascript %}
let shell = require('shelljs');

process.env.NODE_ENV = 'test';
global.__DEV__ = false;

shell.cp('-f', 'src/config/dev.json', 'src/config/default.json');
{% endhighlight %}

In this case the dev API URL does get copied.  In our tests we use sinon to mock the API call so the API doesn't actually get called.  We really just need the require statement not to fail on us.

Now for the dev, QA, and prod builds that use the webpack.prop.config.
Our build process calls a build.js file that calls webpack and pipes its output to the console.
Before calling webpack I added a copy block that copies the correct config based on arguments passed to it.

{% highlight javascript %}
if(process.argv.length >= 3 && process.argv[2] === "dev") {
    shell.cp('-f', 'src/config/dev.json', 'src/config/default.json');
} else {
    shell.cp('-f', 'src/config/prod.json', 'src/config/default.json');
}
{% endhighlight %}

I added an environment specific npm build task for dev and QA:

{% highlight javascript %}
"prebuild:dev": "npm run clean-dist && npm run lint && npm run test",
"build:dev": "babel-node tools/build.js dev",
{% endhighlight %}

Our CI environment will call the `build:dev` task for develop branch commits and build for master branch commits.

And that's it!

Check out the [webpack docs][webpack] to learn more about webpack and the [npm scripts doc][npm-scripts] to learn more about scripting with npm.

[webpack]: https://webpack.github.io
[npm-scripts]: https://docs.npmjs.com/misc/scripts
