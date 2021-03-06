
![BottleJS](/bottle-logo.jpg)
# BottleJS [![Build Status](https://travis-ci.org/young-steveo/bottlejs.svg?branch=master)](https://travis-ci.org/young-steveo/bottlejs)

> A powerful, extensible dependency injection micro container

## Introduction

BottleJS is a tiny yet powerful dependency injection container.  It features lazy loading, middleware hooks, and a clean api inspired by the [AngularJS Module API](https://docs.angularjs.org/api/ng/type/angular.Module) and the simple PHP library [Pimple](http://pimple.sensiolabs.org/).  You'll like BottleJS if you enjoy:

* building a stack from components rather than a kitchen-sink framework.
* uncoupled objects and dependency injection.
* an API that makes sense.
* lazily loaded objects.
* trying cool stuff :smile:

## Browser Support

BottleJS supports IE9+ and other ECMAScript 5 compliant browsers.

## Installation

BottleJS can be used in a browser or in a nodejs app.  It can be installed via bower or npm:

```bash
$ bower install bottlejs
```

```bash
$ npm install bottlejs
```

## Simple Example

The simplest recipe to get started with is `Bottle#service`.  Say you have a constructor for a service object:

```js
var Beer = function() { /* A beer service, :yum: */ };
```

You can register the constructor with `Bottle#service`:

```js
var bottle = new Bottle();
bottle.service('Beer', Beer);
```

Later, when you need the constructed service, you just access the `Beer` property like this:

```js
bottle.container.Beer;
```

A lot happened behind the scenes:

1. Bottle created a provider containing a factory function when you registered the Beer service.
2. When the `bottle.container.Beer` property was accessed, Bottle looked up the provider and executed the factory to build and return the Beer service.
3. The provider and factory were deleted, and the `bottle.container.Beer` property was set to be the Beer service instance.  Accessing `bottle.container.Beer` in the future becomes a simple property lookup.

## Injecting Dependencies

The above example is simple.  But, what if the Beer service had dependencies?  For eample:

```js
var Barley = function() {};
var Hops = function() {};
var Water = function() {};
var Beer = function(barley, hops, water) { /* A beer service, :yum: */ };
```

You can register services with `Bottle#service` and include dependencies like this:

```js
var bottle = new Bottle();
bottle.service('Barley', Barley);
bottle.service('Hops', Hops);
bottle.service('Water', Water);
bottle.service('Beer', Beer, 'Barley', 'Hops', 'Water');
```

Now, when you access `bottle.container.Beer`, Bottle will lazily load all of the dependencies and inject them into your Beer service before returning it.

### Service Factory

If you need more complex logic when generating a service, you can register a factory instead.  A factory function receives the container as an argument, and should return your constructed service:

```js
var bottle = new Bottle();
bottle.service('Barley', Barley);
bottle.service('Hops', Hops);
bottle.service('Water', Water);
bottle.factory('Beer', function(container) {
	var barley = container.Barley;
	var hops = container.Hops;
	var water = container.Water;

	barley.halved();
	hops.doubled();
	water.spring();
	return new Beer(barley, hops, water);
});
```

### Service Provider

This is the meat of the Bottle library.  The above methods `Bottle#service` and `Bottle#factory` are just shorthand for the provider function.  You usually can get by with the simple functions above, but if you really need more granular control of your services in different environments, regiser them as a provider.  To use it, pass a constructor for the provider that exposes a `$get` function.  The `$get` function is used as a factory to build your service.

```js
var bottle = new Bottle();
bottle.service('Barley', Barley);
bottle.service('Hops', Hops);
bottle.service('Water', Water);
bottle.provider('Beer', function() {
	// This environment may not support water.
	// We should polyfill it.
	if (waterNotSupported) {
		Beer.pollyfillWater();
	}

    // this is the service factory.
	this.$get = function(container) {
		var barley = container.Barley;
		var hops = container.Hops;
		var water = container.Water;

		barley.halved();
		hops.doubled();
		water.spring();
		return new Beer(barley, hops, water);
	};
});
```

## Middleware

Bottle supports injecting middleware into the provider pipeline with the `Bottle#middleware` method.  Bottle middleware are just simple functions that intercept a service in the provider phase after it has been created, but before it is accessed for the first time.  The function should return the service, or another object to be used as the service instead.

```js
var bottle = new Bottle();
bottle.service('Beer', Beer);
bottle.service('Wine', Wine);
bottle.middleware(function(service) {
	// this middleware will be run for both Beer and Wine services.
	service.stayCold();
	return service;
});

bottle.middleware('Wine', function(wine) {
	// this middleware will only affect the Wine service.
	wine.unCork();
	return wine;
});
```

## API

### constant(name, value)

Used to add a read only value to the container.

Param     | Type       | Details
:---------|:-----------|:--------
**name**  | *String*   | The name of the constant.  Must be unique to each Bottle instance.
**value** | *Mixed*    | A value that will be defined as enumerable, but not writable.

### factory(name, Factory)

Used to register a service factory

Param       | Type       | Details
:-----------|:-----------|:--------
**name**    | *String*   | The name of the service.  Must be unique to each Bottle instance.
**Factory** | *Function* | A function that should return the service object.  Will only be called once; the Service will be a singleton.  Gets passed an instance of the container to allow dependency injection when creating the service.

### middleware(name, func)

Used to register a middleware function that the provider will use to modify your services at creation time.

Param                      | Type       | Details
:--------------------------|:-----------|:--------
**name**<br />*(optional)* | *String*   | The name of the service this middleware will affect. Will run for all services if not passed.
**func**                   | *Function* | A function that will accept the service as the first parameter.  Should return the service, or a new object to be used as the service.

### provider(name, Provider)

Used to register a service provider

Param        | Type       | Details
:------------|:-----------|:--------
**name**     | *String*   | The name of the service.  Must be unique to each Bottle instance.
**Provider** | *Function* | A constructor function that will be instantiated as a singleton.  Should expose a function called `$get` that will be used as a factory to instantiate the service.

### service(name, Constructor [, dependency [, ...]])

Used to register a service constructor

Param                            | Type       | Details
:--------------------------------|:-----------|:--------
**name**                         | *String*   | The name of the service.  Must be unique to each Bottle instance.
**Constructor**                  | *Function* | A constructor function that will be instantiated as a singleton.
**dependency**<br />*(optional)* | *String*   | An optional name for a dependency to be passed to the constructor.  A dependency will be passed to the constructor for each name passed to `Bottle#service` in the order they are listed.

### value(name, val)

Used to add an arbitrary value to the container.

Param    | Type     | Details
:--------|:---------|:--------
**name** | *String* | The name of the value.  Must be unique to each Bottle instance.
**val**  | *Mixed*  | A value that will be defined as enumerable, but not writable.
