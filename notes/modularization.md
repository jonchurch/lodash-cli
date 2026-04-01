# Lodash Modularization Challenges

This note summarizes the main obstacles to breaking `lodash.js` into real source modules while preserving backward compatibility.

The short version:

- Lodash is not just a collection of utility functions.
- A large amount of behavior is assembled at runtime inside one shared closure.
- The hardest parts are shared mutable state, wrapper/chaining machinery, late prototype mutation, and environment-sensitive bootstrapping.

## Scope

These findings are based on inspecting the upstream [`lodash.js`](https://github.com/lodash/lodash/blob/cb0b9b9212521c08e3eafe7c8cb0af1b42b6649e/lodash.js) source at commit `cb0b9b9212521c08e3eafe7c8cb0af1b42b6649e`.

## Core Observation

The monolith behaves more like a bootstrapped runtime than a flat bag of independent helpers.

At a high level:

1. Top-level environment detection runs once.
2. `runInContext()` creates a new lodash instance with a large shared lexical scope.
3. Methods, aliases, placeholder behavior, wrapper methods, and exports are attached in later passes.

That means "split into files" is not enough on its own. The real task is to separate:

- pure helpers
- shared mutable state
- runtime assembly
- monolith-only wrapper/chaining behavior

## 1. Shared Per-Instance Closure State

The biggest structural issue is [`runInContext`](https://github.com/lodash/lodash/blob/cb0b9b9212521c08e3eafe7c8cb0af1b42b6649e/lodash.js#L1449-L17227). It creates a large amount of mutable state once, then many methods close over that state implicitly.

Examples include:

- `idCounter`
- `oldDash`
- `metaMap`
- `realNames`
- rebound built-ins from the chosen `context`

Snippet:

```js
var runInContext = (function runInContext(context) {
  context = context == null ? root : _.defaults(root.Object(), context, _.pick(root, contextProps));
  // ...
  var idCounter = 0;
  var oldDash = root._;
  var metaMap = WeakMap && new WeakMap;
  var realNames = {};
```

Source:

- [`runInContext` setup](https://github.com/lodash/lodash/blob/cb0b9b9212521c08e3eafe7c8cb0af1b42b6649e/lodash.js#L1449-L1558)

Why this matters:

- Splitting methods into independent modules can accidentally duplicate or disconnect that state.
- Some APIs depend on instance-local shared state, not global stateless helpers.

Concrete example:

- [`uniqueId`](https://github.com/lodash/lodash/blob/cb0b9b9212521c08e3eafe7c8cb0af1b42b6649e/lodash.js#L16317-L16328) depends on the shared `idCounter`. If `uniqueId` is moved into an isolated module with its own counter, behavior changes.

## 2. Shared Mutable Singletons and Runtime Customization

Several public APIs depend on mutable objects or constructor hooks that users are expected to change at runtime.

### `templateSettings`

[`template`](https://github.com/lodash/lodash/blob/cb0b9b9212521c08e3eafe7c8cb0af1b42b6649e/lodash.js#L14882-L14960) reads `lodash.templateSettings` dynamically when called, not from a frozen compile-time constant.

Snippet:

```js
lodash.templateSettings = {
  'escape': reEscape,
  'evaluate': reEvaluate,
  'interpolate': reInterpolate,
  'variable': '',
  'imports': { '_': lodash }
};
```

```js
function template(string, options, guard) {
  var settings = lodash.templateSettings;
  options = assignWith({}, options, settings, customDefaultsAssignIn);
```

Source:

- [`lodash.templateSettings`](https://github.com/lodash/lodash/blob/cb0b9b9212521c08e3eafe7c8cb0af1b42b6649e/lodash.js#L1764-L1814)
- [`template()` reading settings](https://github.com/lodash/lodash/blob/cb0b9b9212521c08e3eafe7c8cb0af1b42b6649e/lodash.js#L14882-L14895)

Why this matters:

- Users can mutate `_.templateSettings` and expect the whole instance to observe the change.
- A modular rewrite must preserve shared identity for this object.

### `memoize.Cache`

[`memoize`](https://github.com/lodash/lodash/blob/cb0b9b9212521c08e3eafe7c8cb0af1b42b6649e/lodash.js#L10639-L10660) exposes a writable constructor hook.

Snippet:

```js
function memoize(func, resolver) {
  // ...
  memoized.cache = new (memoize.Cache || MapCache);
  return memoized;
}

memoize.Cache = MapCache;
```

Source:

- [`memoize`](https://github.com/lodash/lodash/blob/cb0b9b9212521c08e3eafe7c8cb0af1b42b6649e/lodash.js#L10639-L10660)

Why this matters:

- Behavior is partly configured by runtime mutation of function properties.
- A modular system cannot treat all functions as pure exports with no shared mutation surface.

## 3. Runtime Initialization Order Is Semantically Important

Lodash surface area is assembled in multiple late passes. Load order is therefore part of behavior.

Relevant assembly steps include:

- direct method assignment to `lodash`
- alias assignment
- `mixin(lodash, lodash)`
- a second `mixin` pass with `{ chain: false }`
- placeholder assignment
- `LazyWrapper.prototype` augmentation
- `lodash.prototype` augmentation from `LazyWrapper`

Snippet:

```js
// Add methods to `lodash.prototype`.
mixin(lodash, lodash);

// Add aliases.
lodash.each = forEach;
lodash.eachRight = forEachRight;
lodash.first = head;
```

```js
mixin(lodash, (function() {
  var source = {};
  baseForOwn(lodash, function(func, methodName) {
    if (!hasOwnProperty.call(lodash.prototype, methodName)) {
      source[methodName] = func;
    }
  });
  return source;
}()), { 'chain': false });
```

Source:

- [`mixin(lodash, lodash)` and aliases](https://github.com/lodash/lodash/blob/cb0b9b9212521c08e3eafe7c8cb0af1b42b6649e/lodash.js#L16828-L16997)
- [`placeholder` assignment](https://github.com/lodash/lodash/blob/cb0b9b9212521c08e3eafe7c8cb0af1b42b6649e/lodash.js#L17010-L17013)
- [`LazyWrapper` augmentation](https://github.com/lodash/lodash/blob/cb0b9b9212521c08e3eafe7c8cb0af1b42b6649e/lodash.js#L17015-L17124)
- [`lodash.prototype` wiring](https://github.com/lodash/lodash/blob/cb0b9b9212521c08e3eafe7c8cb0af1b42b6649e/lodash.js#L17126-L17225)

Why this matters:

- If modules initialize in a different order, wrapper behavior can drift.
- A future module system needs an explicit runtime assembly phase.

## 4. Wrapper and Chaining Are a Separate Runtime Layer

The wrapper system is one of the strongest arguments against naive modularization.

Key constructors and relationships:

- [`lodash`](https://github.com/lodash/lodash/blob/cb0b9b9212521c08e3eafe7c8cb0af1b42b6649e/lodash.js#L1691-L1700)
- [`LodashWrapper`](https://github.com/lodash/lodash/blob/cb0b9b9212521c08e3eafe7c8cb0af1b42b6649e/lodash.js#L1743-L1749)
- [`LazyWrapper`](https://github.com/lodash/lodash/blob/cb0b9b9212521c08e3eafe7c8cb0af1b42b6649e/lodash.js#L1832-L1840)

Snippet:

```js
function lodash(value) {
  if (isObjectLike(value) && !isArray(value) && !(value instanceof LazyWrapper)) {
    if (value instanceof LodashWrapper) {
      return value;
    }
    if (hasOwnProperty.call(value, '__wrapped__')) {
      return wrapperClone(value);
    }
  }
  return new LodashWrapper(value);
}
```

Why this matters:

- `instanceof` identity has to remain coherent.
- Duplicating constructors across modules can break wrapping and cloning behavior.

Lazy wrapper methods are generated and then mirrored onto `lodash.prototype`:

```js
baseForOwn(LazyWrapper.prototype, function(func, methodName) {
  var lodashFunc = lodash[/* ... */];
  lodash.prototype[methodName] = function() {
    // lazy vs eager dispatch
  };
});
```

Source:

- [`LazyWrapper` to `lodash.prototype` bridging](https://github.com/lodash/lodash/blob/cb0b9b9212521c08e3eafe7c8cb0af1b42b6649e/lodash.js#L17126-L17169)

This is not ordinary module composition. It is runtime-generated surface assembly.

## 5. Metadata and Name-Based Introspection

Some behavior depends on metadata tables and function names, not just direct imports.

### `realNames`

Lodash populates `realNames` by scanning `LazyWrapper.prototype`, then uses it later for name recovery and wrapper logic.

Snippet:

```js
baseForOwn(LazyWrapper.prototype, function(func, methodName) {
  var lodashFunc = lodash[methodName];
  if (lodashFunc) {
    var key = lodashFunc.name + '';
    if (!hasOwnProperty.call(realNames, key)) {
      realNames[key] = [];
    }
    realNames[key].push({ 'name': methodName, 'func': lodashFunc });
  }
});
```

Source:

- [`realNames` population](https://github.com/lodash/lodash/blob/cb0b9b9212521c08e3eafe7c8cb0af1b42b6649e/lodash.js#L17189-L17204)
- [`getFuncName`](https://github.com/lodash/lodash/blob/cb0b9b9212521c08e3eafe7c8cb0af1b42b6649e/lodash.js#L5980-L5995)

Why this matters:

- A modular rewrite that changes naming or duplicates function identities can break wrapper optimizations.

### Wrapper metadata

[`createFlow`](https://github.com/lodash/lodash/blob/cb0b9b9212521c08e3eafe7c8cb0af1b42b6649e/lodash.js#L5164-L5208) uses wrapper metadata, names, and wrapper prototype behavior to decide whether shortcut fusion applies.

This means some "dependencies" are semantic, not just lexical.

## 6. Placeholder Behavior Depends on Shared Instance State

Binding and currying helpers are coupled to placeholder lookup through the active lodash instance.

Snippet:

```js
function getHolder(func) {
  var object = hasOwnProperty.call(lodash, 'placeholder') ? lodash : func;
  return object.placeholder;
}
```

```js
arrayEach(['bind', 'bindKey', 'curry', 'curryRight', 'partial', 'partialRight'], function(methodName) {
  lodash[methodName].placeholder = lodash;
});
```

Source:

- [`getHolder`](https://github.com/lodash/lodash/blob/cb0b9b9212521c08e3eafe7c8cb0af1b42b6649e/lodash.js#L6002-L6005)
- [`default placeholder` assignment](https://github.com/lodash/lodash/blob/cb0b9b9212521c08e3eafe7c8cb0af1b42b6649e/lodash.js#L17010-L17013)

Why this matters:

- Placeholder semantics are not local to one function.
- They depend on shared mutation of function properties and on the `lodash` instance itself.

## 7. Prototype Mutation Is Core to the Public Surface

`mixin` is not just a helper; it is part of the runtime model.

Snippet:

```js
function mixin(object, source, options) {
  // ...
  arrayEach(methodNames, function(methodName) {
    var func = source[methodName];
    object[methodName] = func;
    if (isFunc) {
      object.prototype[methodName] = function() {
        // chain-aware wrapper dispatch
      };
    }
  });
  return object;
}
```

Source:

- [`mixin`](https://github.com/lodash/lodash/blob/cb0b9b9212521c08e3eafe7c8cb0af1b42b6649e/lodash.js#L15828-L15862)

Why this matters:

- Public API assembly happens through side effects on a function object and its prototype.
- This strongly suggests a split between a modular core and an explicit assembly layer.

## 8. Environment Detection and Export Logic Are Entangled with Startup

Top-level environment detection runs before `runInContext()` and is used throughout the file.

Snippet:

```js
var freeGlobal = typeof global == 'object' && global && global.Object === Object && global;
var freeSelf = typeof self == 'object' && self && self.Object === Object && self;
var root = freeGlobal || freeSelf || Function('return this')();
```

Source:

- [`root` and environment detection](https://github.com/lodash/lodash/blob/cb0b9b9212521c08e3eafe7c8cb0af1b42b6649e/lodash.js#L431-L464)

At the end of the file, export behavior is selected based on AMD / CommonJS / global presence:

```js
var _ = runInContext();

if (typeof define == 'function' && typeof define.amd == 'object' && define.amd) {
  root._ = _;
  define(function() {
    return _;
  });
} else if (freeModule) {
  (freeModule.exports = _)._ = _;
  freeExports._ = _;
} else {
  root._ = _;
}
```

Source:

- [`final export block`](https://github.com/lodash/lodash/blob/cb0b9b9212521c08e3eafe7c8cb0af1b42b6649e/lodash.js#L17231-L17258)

Why this matters:

- Environment detection is part of runtime setup, not just packaging.
- In a modularized source tree, this should likely move into a monolith-only entry/adapter layer.

## 9. Some Shared Tables Live Outside `runInContext`

Not all shared state is per-instance. Some tables are top-level mutable lookups shared across all instances.

Examples:

- `contextProps`
- `typedArrayTags`
- `cloneableTags`
- `templateCounter`

These are populated imperatively, not declared as immutable data-only modules.

Source:

- [`contextProps`](https://github.com/lodash/lodash/blob/cb0b9b9212521c08e3eafe7c8cb0af1b42b6649e/lodash.js#L297-L304)
- [`templateCounter`](https://github.com/lodash/lodash/blob/cb0b9b9212521c08e3eafe7c8cb0af1b42b6649e/lodash.js#L306-L308)
- [`typedArrayTags`](https://github.com/lodash/lodash/blob/cb0b9b9212521c08e3eafe7c8cb0af1b42b6649e/lodash.js#L309-L323)
- [`cloneableTags`](https://github.com/lodash/lodash/blob/cb0b9b9212521c08e3eafe7c8cb0af1b42b6649e/lodash.js#L325-L338)

Why this matters:

- Even before entering `runInContext`, lodash has process-wide mutable lookup state.

## What This Suggests for Migration

The cleanest mental model is:

- not "split the monolith mechanically"
- but "extract a modular core plus a runtime assembly layer"

Likely buckets:

- Pure helper modules:
  - easiest to extract
  - low behavioral risk
- Shared state modules:
  - `templateSettings`
  - placeholder/default configuration
  - metadata registries
  - counters/caches
- Wrapper/chaining assembly:
  - probably monolith-specific at first
  - needs explicit boot order
- Environment/export adapters:
  - AMD
  - CJS
  - global/UMD-like entrypoints

## Bottom Line

Breaking `lodash.js` into modules is possible, but the main challenge is preserving the behavior that currently emerges from:

- one shared lexical universe inside `runInContext()`
- shared mutable singleton-like state
- late prototype and namespace mutation
- metadata and name-based wrapper logic
- environment-sensitive startup and export order

That means the first successful modularization design probably will not make every part of lodash "just another module". Some parts need to become explicit runtime assembly and shared-state layers.
