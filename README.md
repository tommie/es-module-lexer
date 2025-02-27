# ES Module Lexer

[![Build Status][actions-image]][actions-url]

A JS module syntax lexer used in [es-module-shims](https://github.com/guybedford/es-module-shims).

Outputs the list of exports and locations of import specifiers, including dynamic import and import meta handling.

A very small single JS file (4KiB gzipped) that includes inlined Web Assembly for very fast source analysis of ECMAScript module syntax only.

For an example of the performance, Angular 1 (720KiB) is fully parsed in 5ms, in comparison to the fastest JS parser, Acorn which takes over 100ms.

_Comprehensively handles the JS language grammar while remaining small and fast. - ~10ms per MB of JS cold and ~5ms per MB of JS warm, [see benchmarks](#benchmarks) for more info._

> [Built with](https://github.com/guybedford/es-module-lexer/blob/main/chompfile.toml) [Chomp](https://chompbuild.com/)

### Usage

```
npm install es-module-lexer
```

For use in CommonJS:

```js
const { init, parse } = require('es-module-lexer');

(async () => {
  // either await init, or call parse asynchronously
  // this is necessary for the Web Assembly boot
  await init;

  const [imports, exports] = parse('export var p = 5');
  exports[0] === 'p';
})();
```

An ES module version is also available:

```js
import { init, parse } from 'es-module-lexer';

(async () => {
  await init;

  const source = `
    import { name } from 'mod\\u1011';
    import json from './json.json' assert { type: 'json' }
    export var p = 5;
    export function q () {

    };

    // Comments provided to demonstrate edge cases
    import /*comment!*/ (  'asdf', { assert: { type: 'json' }});
    import /*comment!*/.meta.asdf;
  `;

  const [imports, exports] = parse(source, 'optional-sourcename');

  // Returns "modထ"
  imports[0].n
  // Returns "mod\u1011"
  source.substring(imports[0].s, imports[0].e);
  // "s" = start
  // "e" = end

  // Returns "import { name } from 'mod'"
  source.substring(imports[0].ss, imports[0].se);
  // "ss" = statement start
  // "se" = statement end

  // Returns "{ type: 'json' }"
  source.substring(imports[1].a, imports[1].se);
  // "a" = assert, -1 for no assertion

  // Returns "p,q"
  exports.toString();

  // Dynamic imports are indicated by imports[2].d > -1
  // In this case the "d" index is the start of the dynamic import bracket
  // Returns true
  imports[2].d > -1;

  // Returns "asdf" (only for string literal dynamic imports)
  imports[2].n
  // Returns "import /*comment!*/ (  'asdf', { assert: { type: 'json' } })"
  source.substring(imports[2].ss, imports[2].se);
  // Returns "'asdf'"
  source.substring(imports[2].s, imports[2].e);
  // Returns "(  'asdf', { assert: { type: 'json' } })"
  source.substring(imports[2].d, imports[2].se);
  // Returns "{ assert: { type: 'json' } }"
  source.substring(imports[2].a, imports[2].se - 1);

  // For non-string dynamic import expressions:
  // - n will be undefined
  // - a is currently -1 even if there is an assertion
  // - e is currently the character before the closing )

  // For nested dynamic imports, the se value of the outer import is -1 as end tracking does not
  // currently support nested dynamic immports

  // import.meta is indicated by imports[2].d === -2
  // Returns true
  imports[2].d === -2;
  // Returns "import /*comment!*/.meta"
  source.substring(imports[2].s, imports[2].e);
  // ss and se are the same for import meta
})();
```

### CSP asm.js Build

The default version of the library uses Wasm and (safe) eval usage for performance and a minimal footprint.

Neither of these represent security escalation possibilities since there are no execution string injection vectors, but that can still violate existing CSP policies for applications.

For a version that works with CSP eval disabled, use the `es-module-lexer/js` build:

```js
import { parse } from 'es-module-lexer/js';
```

Instead of Web Assembly, this uses an asm.js build which is almost as fast as the Wasm version ([see benchmarks below](#benchmarks)).

### Escape Sequences

To handle escape sequences in specifier strings, the `.n` field of imported specifiers will be provided where possible.

For dynamic import expressions, this field will be empty if not a valid JS string.

### Facade Detection

Facade modules that only use import / export syntax can be detected via the third return value:

```js
const [,, facade] = parse(`
  export * from 'external';
  import * as ns from 'external2';
  export { a as b } from 'external3';
  export { ns };
`);
facade === true;
```

### Environment Support

Node.js 10+, and [all browsers with Web Assembly support](https://caniuse.com/#feat=wasm).

### Grammar Support

* Token state parses all line comments, block comments, strings, template strings, blocks, parens and punctuators.
* Division operator / regex token ambiguity is handled via backtracking checks against punctuator prefixes, including closing brace or paren backtracking.
* Always correctly parses valid JS source, but may parse invalid JS source without errors.

### Limitations

The lexing approach is designed to deal with the full language grammar including RegEx / division operator ambiguity through backtracking and paren / brace tracking.

The only limitation to the reduced parser is that the "exports" list may not correctly gather all export identifiers in the following edge cases:

```js
// Only "a" is detected as an export, "q" isn't
export var a = 'asdf', q = z;

// "b" is not detected as an export
export var { a: b } = asdf;
```

The above cases are handled gracefully in that the lexer will keep going fine, it will just not properly detect the export names above.

### Benchmarks

Benchmarks can be run with `npm run bench`.

Current results for a high spec machine:

#### Wasm Build

```
Module load time
> 5ms
Cold Run, All Samples
test/samples/*.js (3123 KiB)
> 20ms

Warm Runs (average of 25 runs)
test/samples/angular.js (739 KiB)
> 2.12ms
test/samples/angular.min.js (188 KiB)
> 1ms
test/samples/d3.js (508 KiB)
> 3.04ms
test/samples/d3.min.js (274 KiB)
> 2ms
test/samples/magic-string.js (35 KiB)
> 0ms
test/samples/magic-string.min.js (20 KiB)
> 0ms
test/samples/rollup.js (929 KiB)
> 4.04ms
test/samples/rollup.min.js (429 KiB)
> 2.16ms

Warm Runs, All Samples (average of 25 runs)
test/samples/*.js (3123 KiB)
> 14.4ms
```

#### JS Build (asm.js)

```
Module load time
> 2ms
Cold Run, All Samples
test/samples/*.js (3123 KiB)
> 35ms

Warm Runs (average of 25 runs)
test/samples/angular.js (739 KiB)
> 3ms
test/samples/angular.min.js (188 KiB)
> 1.08ms
test/samples/d3.js (508 KiB)
> 3.04ms
test/samples/d3.min.js (274 KiB)
> 2ms
test/samples/magic-string.js (35 KiB)
> 0ms
test/samples/magic-string.min.js (20 KiB)
> 0ms
test/samples/rollup.js (929 KiB)
> 5.04ms
test/samples/rollup.min.js (429 KiB)
> 3ms

Warm Runs, All Samples (average of 25 runs)
test/samples/*.js (3123 KiB)
> 17ms
```

### Building

This project uses [Chomp](https://chompbuild.com) for building.

With Chomp installed, download the WASI SDK 12.0 from https://github.com/WebAssembly/wasi-sdk/releases/tag/wasi-sdk-12.

- [Linux](https://github.com/WebAssembly/wasi-sdk/releases/download/wasi-sdk-12/wasi-sdk-12.0-linux.tar.gz)
- [Windows (MinGW)](https://github.com/WebAssembly/wasi-sdk/releases/download/wasi-sdk-12/wasi-sdk-12.0-mingw.tar.gz)
- [macOS](https://github.com/WebAssembly/wasi-sdk/releases/download/wasi-sdk-12/wasi-sdk-12.0-macos.tar.gz)

Locate the WASI-SDK as a sibling folder, or customize the path via the `WASI_PATH` environment variable.

Emscripten emsdk is also assumed to be a sibling folder or via the `EMSDK_PATH` environment variable.

Example setup:

```
git clone https://github.com:guybedford/es-module-lexer
git clone https://github.com/emscripten-core/emsdk
wget https://github.com/WebAssembly/wasi-sdk/releases/download/wasi-sdk-12/wasi-sdk-12.0-linux.tar.gz
gunzip wasi-sdk-12.0-linux.tar.gz
tar -xf wasi-sdk-12.0-linux.tar
mv wasi-sdk-12.0-linux.tar wasi-sdk-12.0
cargo install chompbuild
cd es-module-lexer
chomp test
```

For the `asm.js` build, git clone `emsdk` from  is assumed to be a sibling folder as well.

### License

MIT

[actions-image]: https://github.com/guybedford/es-module-lexer/actions/workflows/build.yml/badge.svg
[actions-url]: https://github.com/guybedford/es-module-lexer/actions/workflows/build.yml

