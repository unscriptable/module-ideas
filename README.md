
Given the following "formal" syntax:

```js
// module ["name"] factory [,] "[" ["dep1" [, "dep2" [, ... ]]] "]";
module "myModule" function (ack, bar) {
	export function () {
		return ack('foo') ? 'foo' : bar('foo');
	};
} ["./ack", "other/bar"];
```

Cn we also have an "alternative" format that works in ES3/5?

```js
module ("myModule") (function (ack, bar) {
	return "export", function () {
		return ack('foo') ? 'foo' : bar('foo');
	};
}) (["./ack", "other/bar"]);
```

This alternative syntax could be "shimmed" for ES3/5 environs:

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