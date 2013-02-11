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
		// using a label to flag exports
		export: return function () {
			return ack('foo') ? 'foo' : 'bar';
		};
	}, 
	module.get(['./ack']), 
	'./myLoader'
);
```

This is not a complete answer.  Devs could still programmatically decorate the
thing labeled by `export:`.  But it seems that the ES6 syntaxes can't prevent
this fully, either?  Also: `export` is a reserved word.

Also: how to declare imports declaratively?

What if there were a syntax that could be made to be ES3/5 compatible just by
adding parentheses, commas, braces, and/or other operators?

```js
// formal syntax:
// module ["name"] factory, ["dep1" [, "dep2" [, ... ]]];
module "myModule" function (ack, bar) {
	console.log(this); // undefined
	export function () {
		return ack('foo') ? 'foo' : bar('foo');
	};
}, "./ack", "other/bar";

// back-compat syntax
module ("myModule") (function (ack) {
	console.log(this); // undefined
	return "export", function () {
		return ack('foo') ? 'foo' : 'bar';
	};
}, "./ack", "other/bar");

// possible syntax for exporting a simple object
module "mySimpleModule" {
	answer: 42,
	question: "What is the meaning of life...blah?"
};

// back-compat syntax for exporting a simple object
module ("mySimpleModule") ({
	answer: 42,
	question: "What is the meaning of life...blah?"
});
```

The back-compat syntax would allow a shim to be created:

```js
var module = (function () {
"use strict";
	var globalCache = {};
	return function module (name) {
		return function (factory) {
			// TODO: support simple object syntax, too?
			return function (/*deps...*/) {
				// serious oversimplification (e.g. ignores async):
				var modules = [], deps = Array.prototype.slice(arguments);
				deps.forEach(function (depname) {
					modules.push(resolveModule(depname));
				});
				globalCache[name] = factory.apply(null, modules);
			};
		};
	}
	function resolveModule (name) { /* ... */ }
}());
```

I think it's optimal to place the dependencies at the bottom, rather than the
top.  The only advantage to having `imports` at the top are to gain an insight
into what interfaces each imported thing supports.  This is less meaningful in
Javascript.  With the dependencies at the bottom, the dev can review the code
at a high level before diving down into the deeper dependencies.

This proposed syntax doesn't allow for the following ES6-style feature:

`import A from "./ack";`

To be honest, I'd be happy with `var` or `let` statements:

```js
// import A from "./ack"; import B from "other/bar";
module "myModule" function (ack, bar) {
	var A = ack.A, B = bar.B;
	export {
		whatevs: 42
	};
}, "./ack", "other/bar";
```

We've got dead code removal from Google Closure Compiler and UglifyJS, so we 
shouldn't be worried about unused bits of the modules wasting space.  

I'm probably missing some other reason why destructuring the imports is so 
important.  What am I missing?

```js
// this is virtually AMD-wrapped CJSM without the function wrapper
module "myModule" {
	var A = import "./ack";
	var B = import "other/bar";
	export {
		whatevs: 42
	};
};

// back-compat needs the `import` and `export` in the function wrapper or else 
// the `import()` function and `export()` function must be global which can
// work (in all cases?) if their context is switched when `module()` executes.
module ("myModule") (function () { 
	var A = import ("./ack");
	var B = import ("other/bar");
	export ({
		whatevs: 42
	});
});
```

This can't work unless the functions are scanned for instances of `import`.
So, we're no better than the AMD-wrapped CJS case. *Dependencies must be listed
outside of the factory or else the factory source must be scanned.* Still, this
is a potential solution as long as we can prove that the developer can't do 
anything to coerce an `import()` or `export()` to execute in the wrong context.
For instance, could the developer export a function that has an `import()`
that then runs in the context of another module?  Need to think about this a
bit more to know if we can prevent this type of "mistake".

Argghh, except for the fact that `export` and `import` are reserved words.

Now we're back to `exports` and `require`!

Another variant:

```js
// using `exports` instead of `export`
module "myModule" function (ack, bar) {
	console.log(this); // undefined
	exports function () {
		return ack('foo') ? 'foo' : bar('foo');
	};
} ["./ack", "other/bar"];

// back-compat syntax
module ("myModule") (function (ack) {
	console.log(this); // undefined
	exports: return function () {
		return ack('foo') ? 'foo' : 'bar';
	};
})(["./ack", "other/bar"]);
```

Why can't we just return the exported module?

```js
// using `exports` instead of `export`
module "myModule" function (ack, bar) {
	console.log(this); // undefined
	return function () {
		return ack('foo') ? 'foo' : bar('foo');
	};
} ["./ack", "other/bar"];

// back-compat (nothing changes inside factory)
module ("myModule") (function (ack, bar) {
	console.log(this); // undefined
	return function () {
		return ack('foo') ? 'foo' : bar('foo');
	};
}) (["./ack", "other/bar"]);
```

Dang, stoopid devs create circular dependencies, that's why. 
The only way to allow circular deps is to use `exports` (but not 
`module.exports = ...`).

Screw it, why not just say that circular deps aren't allowed in the back-
compat syntax?  In the formal syntax, we could allow use of `exports`.

