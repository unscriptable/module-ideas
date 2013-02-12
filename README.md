
Given the following "formal" syntax:

```js
// ModuleDeclaration ::= "module" [StringLiteral] "{" ModuleBody "}"
module "myModule" {
	var ack = import "./ack";
	var bar = import "other/bar";
	export function () {
		return ack('foo') ? 'foo' : bar('foo');
	};
}
```

Can we also have an "alternative" format that works in ES3/5?

This format would not be as robust, of course.  If devs attempt to use variabes
in `require()` or deference these functions, their code will break in ES6.

```js
// ModuleDeclaration ::= "module" ["("] [StringLiteral] [")"] ["("] ModuleFactory [")"]
// ModuleFactory ::= "function" ModuleFactoryParameterList "{" ModuleFactoryBody "}"
// ModuleFactoryParameterList ::= "(" [ "require" [ "," "exports" ] ] ")"
// ModuleFactoryBody ::= ModuleFactoryStatement ( ";" ModuleFactoryStatement )*
// ModuleFactoryStatement ::= ModuleFactoryImport | ModuleFactoryExport | <otherJsStuff>
// ModuleFactoryImport ::= "require"["("] StringLiteral [")"]
// ModuleFactoryExport ::= "exports"["("] StringLiteral [")"]
module ("myModule") (function (require, exports) {
	var ack = require("./ack");
	var bar = require("other/bar");
	exports(function () {
		return ack('foo') ? 'foo' : bar('foo');
	});
});
```

This alternative syntax could be "shimmed" for ES3/5 environs:

```js
var module = (function () {
"use strict";
	var globalCache = {};
	return function module (name) {
		return function (factory) {
			// scan factory.toString() for `require`.
			// note: this is the hairy part unless devs follow the "static analysis rules"
			var deps = parse(factory.toString());
			// serious oversimplification! (e.g. ignores async and lots of other stuff):
			var modules = deps.map(function (depname) {
				return resolveModule(depname);
			});
			var require = createRequire(modules);
			var exports = createExports(name);
			// store something in the cache that can be executed when require()d
			globalCache[name] = { factory: factory.call(null, require, exports) };
		};
	}
	function resolveModule (name) { /* ... */ }
}());
```