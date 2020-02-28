# Import map extensions

_Extending the [import maps](https://github.com/wicg/import-maps) proposal._

## Introduction

The [import maps](https://github.com/wicg/import-maps) proposal is now feature-complete and working towards a stable specification and release in browsers.

In the process of attaining this stability a number of future features were deemed out of scope for the specification, including:

* `import:` URL support (https://github.com/WICG/import-maps/issues/149)
* Supporting import maps for other execution environments such as Web Workers (https://github.com/WICG/import-maps/issues/2)
* Optimizing the unbounded latency cost of deep dependency discovery (https://github.com/WICG/import-maps/issues/21)
* Supporting lazy loading of import maps (https://github.com/WICG/import-maps/issues/92)
* Supporting multiple import maps (https://github.com/WICG/import-maps/issues/19)
* Treating import maps as a whitelisting feature for controlling importer contexts (https://github.com/WICG/import-maps/issues/99)

## Motivation

Due to the original import maps group seeking stability, this proposal was created to enable ongoing collaboration and discussion to
explore possible extensions such as the above, with the goal to move towards finalizing future specifications
for import maps as the feature in browsers continues to evolve over time.

## Proposal Summaries

Currently the following new features have been specified in this proposal:

### Depcache

> Status: Experimental

This specifies a new `"depcache"` field in the import map to optimize the latency waterfall of dependency discovery.

#### Problem Statement

Import maps form a source-of-truth for the resolution of bare module specifiers in browsers.

Dependency trees, by their nature, are designed to support arbitrary depths of dependency discovery - package A might import package B might import package C.

In addition, lazy loading of modules is a first-class feature in browsers through dynamic `import()` providing performance benefits in minimizing the code executed on initial page load.

The problem is that `import('A')` will first have to send a request over the network, before it knows that it will need to import package B, and in turn the same again for package C, incurring an unnecessary latency cost which is proportional to the dependency tree depth.

#### Proposal

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

#### Alternatives

An alternative approach discussed as been a more general preload manifest that can work for all types of web assets.

The argument here is that most web assets don't typically have this level of encapsulation depth, and that this is a problem that surfaces uniquely to modules.

## Under Consideration

The following additions are under consideration:

* Lazy loading of import maps
* `import:` URL support

## Acknowledgements

These extension features are entirely possible thanks to the specification and implementation work on import maps by @domenic and @hiroshige-g.
