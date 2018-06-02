# Tree shaking demo

This repository demonstrates how tree shaking works in different module bundlers (webpack, Rollup, Parcel).

## How to run

```sh
$ yarn install

# Commands to bundle TypeScript files under ./src with each bundler
$ yarn webpack3
$ yarn webpack4
$ yarn rollup
$ yarn parcel
```

## Code to examine

To investigate how tree shaking works in each module bundler, the following files are used as entries (lines other than the `import` phrase are identical among the three).

1. [import-from-lodash.ts](./src/import-from-lodash.ts)

```ts
import { isEqual } from 'lodash';
import { Greeter } from './export-class';
import { b } from './export-possiblly-side-effects';
import { foo } from './reexport';

console.log(isEqual(new Greeter('foo').greet(), b), foo);
```

2. [import-from-lodash-es.ts](./src/import-from-lodash-es.ts)

```ts
import { isEqual } from 'lodash-es';
import { Greeter } from './export-class';
import { b } from './export-possiblly-side-effects';
import { foo } from './reexport';

console.log(isEqual(new Greeter('foo').greet(), b), foo);
```

3. [import-from-lodash-es-specify-path.ts](./src/import-from-lodash-es-specify-path.ts)

```ts
import isEqual from 'lodash-es/isEqual';
import { Greeter } from './export-class';
import { b } from './export-possiblly-side-effects';
import { foo } from './reexport';

console.log(isEqual(new Greeter('foo').greet(), b), foo);
```

`lodash` and `lodash-es` are CommonJS and ES Modules versions of [Lodash](https://lodash.com/) library, respectively.

## Results

The following table shows the sizes of the bundles (after gzipped) created with each bundler.

| Bundler   | `import { isEqual } from 'lodash';` | `import { isEqual } from 'lodash-es';` | `import isEqual from 'lodash-es/isEqual';` |
| --------- | ----------------------------------: | -------------------------------------: | -----------------------------------------: |
| webpack 3 |                            25,278 B |                               28,711 B |                                    4,595 B |
| webpack 4 |                            25,255 B |                                4,251 B |                                    4,265 B |
| Rollup    |                            24,783 B |                               28,256 B |                                    4,006 B |
| Parcel    |                            32,431 B |                               70,560 B |                                    7,854 B |

- No bundler can tree shake `import { isEqual } from 'lodash';`
    - CommonJS cannot be tree shaken due to its dynamic feature
    - If you are using a transpiler such as Babel or TypeScript, you must not transpile your code into CommonJS (Notice `"module": "es2015"` in [tsconfig.json](./tsconfig.json) of this example)
- Only webpack 4 can tree shake `import { isEqual } from 'lodash-es';`
    - Module bundlers do not eliminate lines which (possibly) contain side effects even if they are not imported from anywhere
    - It is difficult for module bundlers to accurately identify side effects (see https://github.com/rollup/rollup/wiki/Troubleshooting#tree-shaking-doesnt-seem-to-be-working)
    - Webpack 4 introduced [the `"sideEffects"` package.json property](https://webpack.js.org/guides/tree-shaking/#mark-the-file-as-side-effect-free) to solve this issue, which is used to mark some (or all) of the files within a package as side effect free
    - `"sideEffects: false"` affects tree shaking of not only library code but also application code (remove `"sideEffects: false"` from [package.json](./package.json) and check the result of `$ yarn webpack4`)
- Parcel does not support tree shaking
    - They are [working on it](https://github.com/parcel-bundler/parcel/issues/392)
- Rollup has [a list of side effect free functions](https://github.com/rollup/rollup/blob/v0.59.4/src/ast/nodes/shared/pureFunctions.ts) which is used to eliminate dead code
    - This is one of reasons (I'm not sure there is any other) why Rollup can tree shake more effectively than webpack in the absence of `sideEffects: "false"`
    - If you pass the `pure_funcs` [compress option](https://github.com/mishoo/UglifyJS2#compress-options) to UglifyJS in webpack config, you can mimic the Rollup's behavior (see [webpack4/index.js](./webpack4/index.js))
- If you are using TypeScript, replacing `/** @class */` with `/*#__PURE__*/` after transpiling and before uglifying may reduce your bundle's size
    - Classes are transpiled into IIFEs and it is generally hard to statically determine if an IIFE has side effects or not
    - `/*#__PURE__*/` annotation are used to notify UglifyJS that the function following the annotation is side effect free
    - TypeScript emits `/** @class */` annotation to classes and therefore by replacing `/** @class */` with `/*#__PURE__*/` unused classes can be tree shaken
    - Babel emits `/*#__PURE__*/` annotations for class IIFEs, so you do not need to add the annotation yourself
    - see https://github.com/webpack/webpack/issues/2899 in detail
    - You can use [babel-minify](https://github.com/babel/minify), which is an ES6+ aware minifier that can process classes as they are, as an alternative, though there may be [a significant increase in the build time](https://github.com/babel/minify#benchmarks)