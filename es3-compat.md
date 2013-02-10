# Motivation

* to eliminate need to be cross-compiled to ES3/5 environs.
* to either have the transport wrapper already included or to make it simple 
  to add it.

First thing to come to mind:

```js
// simplest form, anonymous module
new Module(function (import, export) {
	var ack = import('./ack');
	console.log(this);
	/* { 
		uri: 'file:///blah/blah/myModule.js', 
		import: function () { [native code] },
		export: function () { [native code] }
	} */
	export(function () {
		return ack('foo') ? 'foo' : 'bar';
	});
});

// `new` is optional / implied
Module(function (import, exports) {
	var ack = import('./ack');
	console.log(this);
	/* { 
		uri: 'file:///blah/blah/myModule.js', 
		import: function () { [native code] },
		export: function () { [native code] }
	} */
	export(function () {
		return ack('foo') ? 'foo' : 'bar';
	});
});

// modules must have names when there are multiple per file
Module('myModule', function (import, exports) {
	var ack = import('./ack');
	console.log(this);
	/* { 
		uri: 'file:///blah/blah/myModule.js', 
		import: function () { [native code] },
		export: function () { [native code] }
	} */
	export(function () {
		return ack('foo') ? 'foo' : 'bar';
	});
});
```

Something even more "functional" would be nice. Remove the constructor, remove
`this`, and simply return the exports.  Hm, this is actually AMD with different
variable names (module == define; import == require):

```js
module(function (import) {
	var ack = import('./ack');
	console.log(this); // undefined
	return function () {
		return ack('foo') ? 'foo' : 'bar';
	};
});
```

It'd be nice to not have to scan the function for `import` calls in order to
ensure the dependencies are available before the factory executes.  (This is
the only way AMD loaders can support an r-value `require`.)  How to do this?

We could do it by capturing the import function

```js
(function (require) {
var ack = require('./ack');

module(function () {
	console.log(this); // undefined
	return function () {
		return ack('foo') ? 'foo' : 'bar';
	};
}, require); // signals `module()` to wait for deps before executing factory

}(new MyLocator()));

// a little more compact:
module(function (ack) {
	console.log(this); // undefined
	return function () {
		return ack('foo') ? 'foo' : 'bar';
	};
}, new MyLocator(['./ack']));

// and even more functional:
module(function (ack) {
	console.log(this); // undefined
	return function () {
		return ack('foo') ? 'foo' : 'bar';
	};
}, myLocator(['./ack']));

// named:
module('myModule', function (ack) {
	console.log(this); // undefined
	return function () {
		return ack('foo') ? 'foo' : 'bar';
	};
}, myLocator(['./ack']));

// named and using the built-in locator. aka `module.get`:
module('myModule', function (ack) {
	console.log(this); // undefined
	return function () {
		return ack('foo') ? 'foo' : 'bar';
	};
}, module.get(['./ack']));
```

Next, we need a way to indicate that a module's factory needs to defer its 
result.  How can we do this without forcing the use of promises/futures?

```js
// first param is callback func
module.defer(function (done, ack) {
	console.log(this); // undefined
	// using `setTimeout()` just for illustration
	setTimeout(function () {
		done(function () {
			return ack('foo') ? 'foo' : 'bar';
		});
	}, 0);
}, module.get(['./ack']));
```

Actually, at the application level, it probably doesn't make sense for a module
to specify a loader.  It's packages that may have special loader requirements
since they could load css, text/html, cofeescript, alternate module formats, 
etc.  Therefore, we should allow packages to register loaders, kinda like node
allows packages to register require.extensions:

```js
module.registerLoader(function () {
	// need a way to specify:
	// 1. a url-to-module resolver (async)
	// 2. a predicate to indicate if this loader should be used?
});
```

Where does this call to registerLoader go? obviously, it should go at
the top of a bundle, but what if there is no bundle?  It could be specified in
a package.json, but then the browser would have to load package.json.  What if
the loader were specified in the module definition?

```js
module(
	function (ack) {
		console.log(this); // undefined
		return function () {
			return ack('foo') ? 'foo' : 'bar';
		};
	}, 
	module.get(['./ack']), 
	'./myLoader'
);
```

The myLoader module must be loadable using the built-in loader.  We'd have to 
do some parameter sniffing if the module required a special loader, and didn't
have any dependencies since there'd only be two params:

```js
module(
	function (ack) {
		console.log(this); // undefined
		return function () {
			return ack('foo') ? 'foo' : 'bar';
		};
	}, 
	// no dependencies so no param here??
	'./myLoader'
);
```

Unfortunately, this doesn't force declarative syntax, which allows devs to 
write crazy shiz like the following which prevents compilers from doing satic
analysis:

```js
var myDeps, myLoader;
myDeps = ['foo'].concat('bar').map(function (id) { return 'mypkg' + id; });
myLoader = 'myPkg' + (typeof window != 'undefined' ? 'barLoader' : 'fooLoader';
module(
	function (ack) {
		console.log(this); // undefined
		return function () {
			return ack('foo') ? 'foo' : 'bar';
		};
	}, 
	myDeps,
	myLoader
);
```

I can't think of a way to allow static analysis and be functional at the same 
time.  Hmmmm, what if it were optional to declare exports in a way that a 
static analyzer could find?

```js
module(
	function (ack) {
		console.log(this); // undefined
		export: return function () {
			return ack('foo') ? 'foo' : 'bar';
		};
	}, 
	module.get(['./ack']), 
	'./myLoader'
);
```

This isn't a complete answer.  Devs could still programmatically decorate the 
thing labeled by `export:`.

Also: how to declare imports declaratively?


