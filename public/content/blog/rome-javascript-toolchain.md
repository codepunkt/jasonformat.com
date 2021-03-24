---
slug: rome-javascript-toolchain
title: Rome, a new JavaScript Toolchain
published: 2/28/2020, 12:40:00 PM
updated: 6/2/2020, 7:42:23 PM
_created: 2/28/2020, 12:15:32 PM
---

Sebastian McKenzie, the original creator of Yarn and Babel and a member of the React Native team at Facebook, has been working on an “all-in-one” solution for JavaScript and TypeScript development.

The [Rome](https://github.com/facebookexperimental/rome) project, a reference to “all roads lead to Rome”, was made public on Feb 26th 2020.

## What is Rome?

[Rome](https://github.com/facebookexperimental/rome) is a from-scratch implementation of a complete JavaScript toolchain. It compiles and bundles JavaScript projects, lints and type-checks, runs tests, and can also format code.

#### What does it look like?

While Rome is still very early-stage, the CLI provides some helpful information about its use:

<table>
  <tr>
   <td><strong>CLI Command</strong>
   </td>
   <td><strong>Description</strong>
   </td>
  </tr>
  <tr>
   <td><code>rome bundle</code>
   </td>
   <td>build a standalone js bundle for a package
   </td>
  </tr>
  <tr>
   <td><code>rome compile</code>
   </td>
   <td>compile a single file
   </td>
  </tr>
  <tr>
   <td><code>rome develop</code>
   </td>
   <td>start a web server
   </td>
  </tr>
  <tr>
   <td><code>rome parse</code>
   </td>
   <td>parse a single file and dump its ast
   </td>
  </tr>
  <tr>
   <td><code>rome resolve</code>
   </td>
   <td>resolve a file
   </td>
  </tr>
  <tr>
   <td><code>rome analyzeDependencies</code>
   </td>
   <td>analyze and dump the dependencies of a file
   </td>
  </tr>
</table>

For full usage details, [see CLI Usage](#cliusage).

#### Why might this be a good idea?

Rome takes a different approach to JavaScript tooling than existing Open Source stacks, and is perhaps more similar to the internal monorepo-based tooling found at very large companies. Rather than assembling a build pipeline by passing source code through multiple disparate tools for various tasks, Rome performs all build and compile steps itself.

This helps address one of the problems faced by popular bundlers like Webpack and Rollup, which is that whole-program analysis and optimization ends up being very difficult or expensive because each tool must parse and construct its own AST.

## Bundling

Rome’s architecture is relatively unique: all compilation happens on a per-module basis, which allows each module to be processed in a pool of worker threads. This works well for per-module transforms, but presents a challenge for bundling: in order to avoid having to re-parse every module produced by the workers, the modules need to be pre-namespaced such that they can all share a single scope.

To make bundling possible despite compilation being per-file, Rome prefixes all module-scoped variables with an identifier generated based on the module’s filename. For example, a `foo` variable in a file called `test.js` becomes `test_js_foo`.

This is also applied to each module’s imported and exported identifiers, which means any module export can be addressed using only the module’s filename and the export name:

<table border="1" bordercolor="#ddd">
  <tr>
   <td><strong>Filename</strong>
   </td>
   <td><strong>Contents</strong>
   </td>
   <td><strong>Output</strong>
   </td>
  </tr>
  <tr>
   <td>test.js
   </td>
   <td>

```js
export const foo = 1;
```

   </td>
   <td>

```js
const ___R$test_js$foo = 1;
```

   </td>
  </tr>
  <tr>
   <td>index.js
   </td>
   <td>

```js
import { foo } from './test.js';
console.log(foo);
```

   </td>
   <td>

```js
console.log(___R$test_js$foo);
```

   </td>
  </tr>
</table>

## Output Quality

For the modern web developer, tooling often dictates how efficient and size-conscious our applications can be. This means we have a vested interest in understanding the composition of our bundles, and Rome’s output is worth paying some attention to. In particular, I’m always interested in getting a sense of whether the bundles generated by a tool collapse modules into a shared closure like Rollup, or preserve module boundaries using closures and a runtime loader implementation like Webpack.

I conducted an initial investigation of what Rome’s output looks like. It appears to produce “scope-collapsed” single-closure bundles, fairly similar to those generated by Rollup:

<table border="1" bordercolor="#ddd">
  <tr>
   <td><strong>Input (module):</strong>
   </td>
   <td><strong>Output (bundle):</strong>
   </td>
  </tr>
  <tr>
   <td>

```js
function hello() {
  return 'Hello World';
}

console.log(hello());
```

   </td>
   <td>

```js
(function (global) {
  'use strict';
  // input.ts

  const ___R$rome$input_ts = {};
  function ___R$$priv$rome$input_ts$hello() {
    return 'Hello World';
  }

  console.log(___R$$priv$rome$input_ts$hello());

  return ___R$rome$input_ts;
})(typeof global !== 'undefined' ? global : typeof window !== 'undefined' ? window : this);
```

   </td>
  </tr>
</table>

There is currently no provided way to minify the bundled output, which is to be expected given the project’s early preview status. However, running the above result through Terser yields a very reasonable output:

```js
!(function (o) {
  'use strict';
  console.log('Hello World');
})('undefined' != typeof global ? global : 'undefined' != typeof window && window);
```

As you can see, there’s a small amount of low-hanging optimization fruit here even with this very simple bundle. Ideally the bundler could be made aware of the intended mode, and if it were known to be compiling for an ES Modules target it could omit the closure and strict mode directive. It could also hoist the declaration of “global” to module scope, which in the above case would allow Terser to dead-code-eliminate it.

## In Larger Projects

Let’s look at a very slightly more complex demo, involving two modules with a shared common dependency:

**entry.tsx**:

```js
import React from './react';
import title from './other';
// Note: dynamic import doesn't yet work in Rome
// const title = import('./other').then(m => m.default);

async function App(props: any) {
  return <div id="app">{await title()}</div>;
}

App({}).then(console.log);
```

**other.tsx**:

```js
import React from './react';
export default () => <h1>Hello World</h1>;
```

**react.tsx**:

```js
type VNode = {
  type: string,
  props: any,
  children: Array<VNode | string>
};
function createElement(type: string, props: any, ...children: Array<VNode | string>): VNode {
  return { type, props, children };
}
export default { createElement };
```

Bundling this using `rome bundle entry.tsx out` produces a directory with an index.js file (and a Source Map):

```js
(function (global) {
  'use strict';
  // rome/react.tsx

  function ___R$$priv$rome$react_tsx$createElement(type, props, ...children) {
    return { type: type, props: props, children: children };
  }
  const ___R$rome$react_tsx$default = {
    createElement: ___R$$priv$rome$react_tsx$createElement
  };

  // rome/other.tsx

  const ___R$rome$other_tsx$default = () => ___R$rome$react_tsx$default.createElement('h1', null, 'Hello World');

  // rome/test.tsx

  const ___R$rome$test_tsx = {};
  async function ___R$$priv$rome$test_tsx$App(props) {
    return ___R$rome$react_tsx$default.createElement('div', { id: 'app' }, await ___R$rome$other_tsx$default());
  }

  ___R$$priv$rome$test_tsx$App({}).then(console.log);

  return ___R$rome$test_tsx;
})(typeof global !== 'undefined' ? global : typeof window !== 'undefined' ? window : this);
```

It’s a little bit harder to follow, but we can see the same structure as the single-module example is still in place.

Stripping away the module implementations and dead CommonJS interop code from Rome’s output, our three source modules are inlined into a single wrapping closure:

```js
(function(global) {
  'use strict';
  // rome/react.tsx
  const ___R$rome$react_tsx$default = /* snip */;

  // rome/other.tsx
  const ___R$rome$other_tsx$default = /* snip */;

  // rome/entry.tsx
  ___R$$priv$rome$entry_tsx$App({}).then(console.log);
})(window);
```

## Production Minification

As I mentioned, Rome doesn’t currently include production minification, though its design does lend itself well to minification at the bundle level. We can run the above output through Terser to see what it looks like. I’ve beautified Terser’s output here for readability:

```js
!(function (e) {
  const n = {
    createElement: function (e, n, ...t) {
      return {
        type: e,
        props: n,
        children: t
      };
    }
  };
  (async function (e) {
    return n.createElement(
      'div',
      {
        id: 'app'
      },
      await n.createElement('h1', null, 'Hello World')
    );
  })().then(console.log);
})('undefined' != typeof global ? global : 'undefined' != typeof window && window);
```

After minification, the output actually looks pretty good! This is a very simple example application though, so we’re not able to see how this scales up to full applications quite yet.

## Further Optimization

I’ve been [working on a project](https://twitter.com/i/events/1218283019083157505) for the past half-year that aims to apply automated optimization to bundled JavaScript (it’s not released yet, sorry!). As a test, I tried passing Rome’s output through that compiler before passing it through Terser with the same settings as above. I’m happy to say this yielded something close to an ideal output: there’s no wrapping function, no dead code, and it leverages the size benefits of modern syntax:

```js
const e = {
  createElement: (e, n, ...t) => ({ type: e, props: n, children: t })
};
(async () => e.createElement('div', { id: 'app' }, await e.createElement('h1', null, 'Hello World')))().then(
  console.log
);
```

This is promising!

## Code Splitting

Rome does not yet appear to support dynamic import or Code Splitting. Using `import()` statements in code actually does discover the imported module, but it gets inlined into the bundle as if it were a static import. The original `import()` statement is left unmodified in the generated output, which causes an error.

It remains to be seen how Code Splitting and chunking will affect the output quality, since both rely on accessing variables enclosed in one bundle from another. I’m not yet familiar enough with Rome to even guess at what this might look like.

## CLI Usage

If you just want to take a peek at what Rome’s CLI has on offer, here’s the --help output you’ll get without having to build it yourself (though it’s very quick to build!):

```sh
$ rome --help
  Usage: rome [command] [flags]

  Options

    --benchmark
    --benchmark-iterations <num>
    --collect-markers
    --cwd <input>
    --focus <input>
    --grep <input>
    --inverse-grep
    --log-path <input>
    --logs
    --log-workers
    --markers-path <input>
    --max-diagnostics <num>
    --no-profile-workers
    --no-show-all-diagnostics
    --profile
    --profile-path <input>
    --profile-sampling <num>
    --profile-timeout <num>
    --rage
    --rage-path <input>
    --resolver-mocks
    --resolver-scale <num>
    --silent
    --temporary-daemon
    --verbose
    --verbose-diagnostics
    --watch

  Code Quality Commands

    ci    install dependencies, run lint and tests
    lint  run lint against a set of files
    test  run tests
      --no-coverage
      --show-all-coverage
      --update-snapshots

  Internal Commands

    evict  evict a file from the memory cache
    logs
    rage

  Process Management Commands

    restart  restart daemon
    start    start daemon (if none running)
    status   get the current daemon status
    stop     stop a running daemon if one exists
    web

  Project Management Commands

    config
    publish  TODO
    run      TODO

  Source Code Commands

    analyzeDependencies  analyze and dump the dependencies of a file
      --compact
      --focus-source <input>
    bundle               build a standalone js bundle for a package
    compile              compile a single file
      --bundle
    develop              start a web server
      --port <num>
    parse                parse a single file and dump its ast
      --no-compact
      --show-despite-diagnostics
    resolve              resolve a file
```