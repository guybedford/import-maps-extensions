# Import map extensions

_Extending the [import maps](https://github.com/wicg/import-maps) proposal._

## Introduction

The [import maps](https://github.com/wicg/import-maps) proposal is now feature-complete and working towards a stable specification and release in browsers.

In the process of attaining this stability a number of future features were deemed out of scope for the specification.

## Motivation

Due to the original import maps group seeking stability, this proposal was created to enable ongoing collaboration and discussion to
explore possible extensions such as the above, with the goal to move towards finalizing future specifications
for import maps as the feature in browsers continues to evolve over time.

## Proposals

Currently the following new features have been [specified in this proposal](https://guybedford.github.io/import-maps-extensions/):

* [Depcache: Optimizing the unbounded latency cost of deep dependency discovery](#depcache) (https://github.com/WICG/import-maps/issues/21)
* [Supporting lazy-loading of import maps](#lazy-loading-of-import-maps) (https://github.com/WICG/import-maps/issues/19)

The following additional proposals are under consideration:

* `import:` URL support (https://github.com/WICG/import-maps/issues/149)
* Supporting import maps for other execution environments such as Web Workers (https://github.com/WICG/import-maps/issues/2)

## Depcache

> Status: Experimental

Specification: https://guybedford.github.io/import-maps-extensions/#parallelizing

Implementation Status: [Implemented in SystemJS](https://github.com/systemjs/systemjs/pull/2134)

This specifies a new `"depcache"` field in the import map to optimize the latency waterfall of dependency discovery.

### Problem Statement

Import maps form a source-of-truth for the resolution of bare module specifiers in browsers.

Dependency trees, by their nature, are designed to support arbitrary depths of dependency discovery - package A might import package B might import package C.

In addition, lazy loading of modules is a first-class feature in browsers through dynamic `import()` providing performance benefits in minimizing the code executed on initial page load.

The problem is that `import('A')` will first have to send a request over the network, before it knows that it will need to import package B, and in turn the same again for package C, incurring an unnecessary latency cost which is proportional to the dependency tree depth.

### Proposal

The proposal is for modules to be able to directly declare a _dependency cache_ upfront in the import map, as an optimization artifact created at build time (since import maps are populated by build time processes anyway):

```html
<script type="importmap">
{
  "imports": {
    "a": "/package-a.js",
    "b": "/package-b.js",
    "c": "/package-c.js"
  },
  "depcache": {
    "/package-a.js": ["b"],
    "/package-b.js": ["c"]
  }
}
</script>
```

With the dependency cache populated, any time a load to `import('a')` is made, the browser is able to infer the deep dependency tree before the network request completes, and thus send out network requests to packages A, B and C in parallel avoiding the latency waterfall.

### Alternatives

An alternative approach discussed as been a more general preload manifest that can work for all types of web assets.

The argument here is that most web assets don't typically have this level of encapsulation depth, and that this is a problem that surfaces uniquely to modules.

## Lazy Loading of Import Maps

> Status: Experimental

Specification: Pending

Implementation Status: [Implemented in SystemJS](https://github.com/systemjs/systemjs/pull/2215)

This proposal extends the "waiting for import maps" phase from being a single phase at startup to being a phase that can be retriggered at any time during the execution of the page.

### Problem Statement

Currently the import maps specification has an intial phase called "waiting for import maps" which corresponds to the completion of loading all import maps present on the page.

This is designed to support dynamic injection of import maps, with the phase transition out of "waiting for import maps" happening as soon as there is a `import()` or `<script type="module">` load triggered.

As a result, the import map for the page becomes fully locked down as soon as the first module has been loaded, thereby excluding all lazy-loading in performance optimization workflows or otherwise.

### Proposal

The proposal consists of two main components:

1. Carefully defining an immutable extension mechanism for the import map.
2. Re-triggering the "waiting for import maps" state whenever a new `<script type="importmap">` is injected into the HTML page.

#### 1. Defining Immutable Import Map Extension

It is important that the extension process does not break the idempotency requirement of the `HostResolveImportedModule` hook defined in the ECMA-262 specification.

There are two fields in import maps for which we need to define the immutable extension - `"imports"` and `"scopes"`.

For `"imports"`, the extension is straightforward - if a lazy-loaded map attempts to redefine an existing property of the import map, we throw a validation error.

For `"scopes"`, we have to be a little more strict to ensure there are no possible cascades. For example, consider:

```html
<script type="importmap">
{
  "imports": {
    "dep": "/path/to/dep.js"
  },
  "scopes": {
    "/scope/": {
      "pkg": "/path/to/pkg.js"
    }
  }
}
</script>
```

where later on we lazy load the new import map:

```html
{
  "scopes": {
    "/scope/scoped-package/": {
      "dep": "/path/to/scoped-dep.js"
    }
  }
}
```

In the above, if we had already loaded any module from `/scope/scoped-package/module.js`, that contained an `import 'dep'` then it would have resolved differently to
what is being defined in the new map, and we wouldn't necessarily know that to be the case.

To ensure cascading situations like this never break the import map immutability, we carefully define the rule for scope extension that 
when defining a new scope boundary, if any of the modules within that scope boundary have already been loaded in the module registry, then we throw a validation error.

As a result the above second lazy-loaded map would be a validation error if and only if `/scope/scoped-package/module.js` or any other module in this scope already
exists in the module registry.

#### 2. Re-triggering the "waiting for import maps" state

As soon as any new `<script type="importmap">` is injected into the HTML page, we switch back into the "waiting for import maps" state exactly as defined on init.

Currently, this state causes any new top-level import operations to wait on this state before proceeding, so by re-triggering this state that same mechanism is reinvoked.

There might still be in-progress module loads in the page, which are still unresolved. These will continue to resolve with the current import map or the extended import
map depending on network timing. As soon as the new import map is fully loaded it will apply to any new module resolutions immediately.

There is thus some timing dependency here, but this is mitigated by the fact that import maps are carefully defined to be immutably extensible.

The following guarantees are the primary guarantees that hold:

1. Any `<script type="module">` or dynamic `import()` that is called immediately after the lazy `<script type="importmap">` injection, will be able to rely on the new lazily-defined maps
2. Any existing in-progress top-level loads may or may not have these mappings available for resolutions (primarily in the case of a slow network), but they cannot rely on them.

#### Alternatives

An alternative approach to the "waiting problem" of lazy import map definitions would be to move the waiting period into the resolver function itself (`HostResolveImportedModule`).

This way, all resolve calls would be able to wait on the new import maps and we would have a full guarantee of predictability regardless of network profile.

Currently the resolver is not designed to be asynchronous in this way so this would be a larger change. In addition this would lead to an unnecessary delay for in-progress module loads since all
resolutions would suddently be paused while the new import map is fetched and process. Thus, while it may seem at some theoretical level more correct, it may not be the most practical in real workflows.
The primary guarantees of correctness to be relied upon is the well-defined nature of immutable map extension and when the individual top-level loads are initiated.

## Acknowledgements

These extension features are entirely possible thanks to the specification and implementation work on import maps by @domenic and @hiroshige-g.
