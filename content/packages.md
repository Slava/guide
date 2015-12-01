---
title: Building Packages
---

In Meteor, there are two environments for writing code: apps and packages. The app environment is designed for rapid iteration and does a lot for you automatically. The package environment gives you much more control and enables you to ship more easily reusable and testable code.

You might want to build a package for two reasons:

1. You're building a medium or large-sized app following the [app structure guide](structure.md), and you want to put your app code in packages to enable better modularity and control.
2. You want to share some code between different apps in your organization - for example, you have several microservices that need to share utilities, or an admin UI for your site that connects to the same database.
2. You have some code you want to share with the community on [Atmosphere](https://atmospherejs.com/), Meteor's package repository.

This guide will cover the basics of building a Meteor package, which will apply to both use cases above. There are some additional guidelines to follow when building a package to publish to atmosphere, and that's covered in the guide about [building a great Atmosphere package](#XXX). Either way, you should read this first.

## Creating a package

To get started writing a package, use the Meteor command line tool:

```
meteor create --package my-package
```

If you run this inside an app, it will place the newly generated package in that app's `packages/` directory. Outside an app, it will just create a standalone package directory. The command also generates some boilerplate files for you:

```
my-package
├── README.md
├── package.js
├── my-package-tests.js
└── my-package.js
```

The `package.js` file is the main file in every Meteor package. This is a JavaScript file that defines the metadata, files loaded, architectures, NPM packages, and Cordova packages for your Meteor package.

In this guide article, we will go over some important points for building packages, but we won't explain every part of the `package.js` API. To learn about all of the options, [read about the `package.js` API in the Meteor docs.](http://docs.meteor.com/#/full/packagejs)

## onUse vs. onTest

XXX

## Adding files and assets

The main function of a Meteor package is to contain source code (JS, CSS, and any transpiled languages) and assets (images, fonts, and more). They act as a unit of code organization, give the developer complete control over which files are loaded where, and can be shared between different applications. Here's how you add files to a package:

```js
// Example for adding files to a package
// Pulled from the lists-show package in the Todos example app
Package.onUse(function(api) {
  // ...

  api.addFiles([
    'todos-item.html',
    'todos-item.js',
    'lists-show.html',
    'lists-show.js',
    'lists-show-page.html',
    'lists-show-page.js',
    'lists-show.less'
  ], 'client');

  // ...
});
```

And the same for assets:

```js
// Example for adding assets to a package
// Pulled from the less-imports package in the Todos example app
Package.onUse(function(api) {
  // ...

  api.addAssets([
    'font/OpenSans-Light-webfont.eot',
    'font/OpenSans-Light-webfont.svg',
    'font/OpenSans-Light-webfont.ttf',
    'font/OpenSans-Light-webfont.woff',
    'font/OpenSans-Regular-webfont.eot',
    'font/OpenSans-Regular-webfont.svg',
    'font/OpenSans-Regular-webfont.ttf',
    'font/OpenSans-Regular-webfont.woff',
  ], 'client');

  // ...
});
```

As you can see, the `addFiles` and `addAssets` functions allow you to pass a list of files as the first argument, and an architecture (or array of architectures) as the second argument. For more information about this, see the section about architectures below.

## Package dependencies

Another very important feature of Meteor packages is the ability to register dependencies on other packages. This is done via `api.use` and `api.imply`.

`api.use` is for recording the dependencies of your package. These dependencies are only available inside the package. However, Meteor's package system is single-loading so no two packages in the same app can have dependencies on conflicting versions of a single package. Read more about that in the section about version constraints below.

```js
// Example of using api.use to register dependencies internal to the package
api.use([
  'ecmascript',
  'check',
  'ddp',
  'underscore',
  'aldeed:simple-schema@1.3.3',
  'mdg:validation-error@0.1.0'
]);
```

`api.imply`, on the other hand, does not include the package as an internal dependency; instead, it makes these packages available to the user of the package. In the example below, we use `api.imply` to include Flow Router in the `todos-lib` package, so that any user of `todos-lib` also gets Flow Router. This can be helpful to avoid keeping long lists of dependencies up to date by creating meta-packages that encapsulate many dependencies of your app at once.

```js
// Example of using api.imply to make a meta-package
// Taken from the todos-lib package in the Todos example app
api.imply([
  'kadira:flow-router@2.7.0',
  'kadira:blaze-layout@2.2.0',
  'arillo:flow-router-helpers@0.4.5',
  'zimme:active-route@2.3.0',
]);
```

`api.use` and `api.imply` can also take an extra argument to specify the architecture on which these dependencies should be used - this way, you can register a dependency on only the server version of HTTP, for example. This can be helpful when you are trying to reduce the side of your client-side app bundle.

## Architectures

Meteor packages are built around the idea of multiple architectures where code might run. Here are all currently possible architectures:


- `web` or `client` - code that runs in a web browser; can be split between Cordova and browser.
    - `web.browser`
    - `web.cordova`

Keep in mind that when your app is loaded in a mobile web browser, the `web.browser` version of the code runs; the `web.cordova` architecture is only for code that uses native Cordova plugins - more on that below.

- `os` or `server` - code that runs in a Node.js server program.
    - `os.osx.x86_64`
    - `os.linux.x86_64`
    - `os.linux.x86_32`
    - `os.windows.x86_32`

As you can see, the architecture can be specified based on operating system, but in practice this is only necessary for packages with binary NPM dependencies, and in the overwhelming majority of cases can be done for you automatically - see the section on NPM below.

Note that Meteor always runs in 32 bit mode on Windows, due to some issues with 64 bit Node and Windows.

## Semantic versioning and version constraints

Meteor's package system relies heavily on [Semantic Versioning](http://semver.org/), or SemVer. When one package declares a dependency on another, it always comes with a version constraint. These version constraints are then solved by Meteor's industrial-grade Version Solver to arrive at a set of package versions that meet all of the requirements, or display a helpful error if there is no solution.

The mental model here is:

1. **The major version must always match exactly.** If package `a` depends on `b@2.0.0`, the constraint will only be satisfied if the version of package `b` starts with a `2`. This means that you can never have two different major versions of a package in the same app.
2. **The minor and patch version numbers must be greater or equal to the requested version.** If the dependency requests version `2.1.3`, then `2.1.4` and `2.2.0` will work, but `2.0.4` and `2.1.2` will not.

The constraint solver is necessary because Meteor's package system is **single-loading** - that is, you can never have two different versions of the same package loaded side-by-side in the same app. This is particularly useful for packages that include a lot of client-side code, or packages that expect to be singletons.

Note that the version solver also has a concept of "gravity" - when many solutions are possible for a certain set of dependencies, it always selects the oldest possible version. This is helpful if you are trying to develop a package to ship to lots of users, since it ensures your package will be compatible with the lowest common denominator of a dependency. If your package needs a newer version than is currently being selected for a certain dependency, you need to update your `package.js` to have a newer version constraint.

## Cordova plugins

Meteor packages can include [Cordova plugins](http://cordova.apache.org/plugins/) to ship native code for the Meteor mobile app container. This way, you can interact with the native camera interface, use the gyroscope, save files locally, and more.

Include Cordova plugins in your Meteor package by using [Cordova.depends](http://docs.meteor.com/#/full/Cordova-depends).

Read more about using Cordova in the [mobile guide](#XXX).

## NPM packages in a Meteor package

Meteor packages can include [NPM packages](https://www.npmjs.com/) to use JavaScript code from outside the Meteor package ecosystem, or to include JavaScript code with native dependencies.

Include NPM packages in your Meteor package by using [Npm.depends](http://docs.meteor.com/#/full/Npm-depends). For example, here's how you could include the `github` package from NPM:

```js
Npm.depends({
  github: '0.2.4'
});
```

You can compile client-side NPM packages into your package by using the [`cosmos:browserify`](https://github.com/elidoran/cosmos-browserify) package.

### Converting between callbacks and Fibers

Many NPM packages rely on an asynchronous, callback or promise-based coding style. For several reasons, Meteor is currently built around a synchronous-looking but still non-blocking style using [Fibers](https://github.com/laverdet/node-fibers).

Many Meteor APIs, for example collections, rely on running inside a fiber context; Meteor also keeps track of certain collection state on the server so that code can be associated with a particular client. This means you need to do a little extra work to use asynchronous Node code inside a Meteor app. Let's look at an example of some code that won't work, using the code example from the [node-github repository](https://github.com/mikedeboer/node-github):

```js
// Inside a Meteor method definition
updateGitHubFollowers() {
  github.user.getFollowingFromUser({
    user: 'stubailo'
  }, (err, res) => {
    // Using a collection here will throw an error
    // because the asynchronous code is not in a fiber
    Followers.insert(res);
  });
}
```

Let's look at a few ways to resolve this issue.

#### Option 1: Meteor.bindEnvironment

In most cases, simply wrapping the callback in `Meteor.bindEnvironment` will do the trick. This function both wraps the callback in a fiber, and does some work to maintain Meteor's server-side environment tracking. Here's the same code with `Meteor.bindEnvironment`:

```js
// Inside a Meteor method definition
updateGitHubFollowers() {
  github.user.getFollowingFromUser({
    user: 'stubailo'
  }, Meteor.bindEnvironment((err, res) => {
    // Everything is good now
    Followers.insert(res);
  }));
}
```

However, this won't work in all cases - since the code runs asynchronously, we can't use anything we got from an API in the method return value. We need a different approach that will convert the async API to a synchronous-looking one that will allow us to return a value.

#### Option 2: Meteor.wrapAsync

Many NPM packages adopt the convention of taking a callback that accepts `(err, res)` arguments. If your asynchronous function fits this description, like the one above, you can use `Meteor.wrapAsync` to convert to a fiberized API that uses return values and exceptions instead of callbacks, like so:

```js
// Setup sync API
github.user.getFollowingFromUserFiber =
  Meteor.wrapAsync(github.user.getFollowingFromUser, github.user);

// Inside a Meteor method definition
updateGitHubFollowers() {
  const result = github.user.getFollowingFromUserFiber({
    user: 'stubailo'
  });

  Followers.insert(res);

  // Return how many followers we have
  return res.length;
}
```

#### Option 3: Promises

Recently, a lot of NPM packages have been moving to Promises instead of callbacks for their API. This means you actually get a return value from the asynchronous function, but it's just an empty shell where the real value is filled in later. If you are using a package that has a promise-based API, you can convert it to synchronous-looking code very easily.

First, add the Meteor promise package:

```sh
meteor add promise
```

Now, you can use `Promise.await` to get a return value from a promise-returning function. For example, here is how you could send a text message using the Node Twilio API:

```js
sendTextMessage() {
  const promise = client.sendMessage({
    to:'+16515556677',
    from: '+14506667788',
    body: 'Hello world!'
  });

  // Wait for and return the result
  return Promise.await(promise);
}
```

Using the new `async`/`await` API for the above in the newest versions of JavaScript, the above code becomes even simpler: XXX does this work right now? What about in Meteor 1.3?

```js
// Mark the method as async
async sendTextMessage() {
  // Wait for the promise using the await keyword
  const result = await client.sendMessage({
    to:'+16515556677',
    from: '+14506667788',
    body: 'Hello world!'
  });

  return result;
}
```

## Local packages vs. published packages

If you've ever looked inside Meteor's package cache at `~/.meteor/packages`, you know that the on-disk format of a built Meteor package is completely different from the way the source code looks when you're developing the package. The idea is that the target format of a package can remain consistent even if the API for development changes. Read more about published packages in the [article about publishing packages](XXX).

## Package testing

Meteor packages are a great unit for code testing - you can test your packages individually or together with others. Read more in the [testing article](XXX).
