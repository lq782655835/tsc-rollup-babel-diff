# rollup VS babel vs tsc

## 前提知识

es和cjs模式打包区别： **es模式 import保留，在cjs中始终是require格式。**

## tsc

tsconfig 所有字段解释：https://www.typescriptlang.org/tsconfig#Emit_6246

* 描述文件参数：常用来和babel/rollup合作编译最终文件
  * declaration 是否产出描述文件
  * 和emitDeclarationOnly，控制只产出d.ts文件，不编译出js文件。
* 编译参数：
  * module：告知当前项目，编译源码用的那种模块。commonjs、umd、amd、es6
  * target：语法编译后支持度。如：是es5，会直接把const转为var。es6就不会

``` js
// tsconfit.json
{
  "compilerOptions": {
    "target": "es5" /* Specify ECMAScript target version: 'ES3' (default), 'ES5', 'ES2015', 'ES2016', 'ES2017', 'ES2018', 'ES2019', 'ES2020', or 'ESNEXT'. */,
    "module": "ESNext" /* Specify module code generation: 'none', 'commonjs', 'amd', 'system', 'umd', 'es2015', 'es2020', or 'ESNext'. */,
    "baseUrl": "./",
    "rootDir": "src",
    "outDir": "./dist/types",
    /* Strict Type-Checking Options */
    "strict": true /* Enable all strict type-checking options. */,

    /* Module Resolution Options */
    "esModuleInterop": true /* Enables emit interoperability between CommonJS and ES Modules via creation of namespace objects for all imports. Implies 'allowSyntheticDefaultImports'. */,
    "moduleResolution": "node",

    /* Advanced Options */
    "skipLibCheck": true /* Skip type checking of declaration files. */,
    "forceConsistentCasingInFileNames": true /* Disallow inconsistently-cased references to the same file. */,
    "declaration": true,
    "emitDeclarationOnly": true,
    "noEmitOnError": true
  },
  "include": ["src", "jest.config.js", ".eslintrc.js", "rollup.config.js"],
  "exclude": ["dist"]
}
```

## babel

* useESModules: 对应tsc target
* modules: cjs/es/umd。对应tsc module。

``` js
// .babelrc.js
let babel_env = process.env['BABEL_ENV'];
let loose = false, // Babel’s loose mode transpiles ES6 code to ES5 code that is less faithful to ES6 semantics.
  modules = false,
  useESModules = false;

console.log({ loose, modules, useESModules });

switch (babel_env) {
  case 'commonjs':
    loose = true;
    modules = 'cjs';
    useESModules = false;
    break;
  case 'es':
    loose = true;
    modules = false; // es6
    useESModules = true;
    break;
  case 'umd':
    loose = false;
    modules = false;
    useESModules = false;
    break;
}

const presets = [
  ['@babel/preset-env', { loose, modules }],
  '@babel/preset-typescript',
];

const plugins = [
  'transform-remove-console',
  '@babel/plugin-transform-typescript',
  // '@babel/plugin-proposal-class-properties',
  // '@babel/plugin-proposal-object-rest-spread',
  ['@babel/plugin-transform-runtime', { useESModules }],
];

module.exports = { presets, plugins };
```

## rollup

* output.format: cjs/es/umd。对应tsc module

> target不需要，rollup大部分是打包库的第一选择，不需要提供target编译选择。
``` js
import multiInput from 'rollup-plugin-multi-input';
import typescript from '@rollup/plugin-typescript';
// A Rollup plugin which locates modules using the Node resolution algorithm, for using third party modules in node_modules
import { nodeResolve } from '@rollup/plugin-node-resolve';
//  A Rollup plugin to convert CommonJS modules to ES6
import commonjs from '@rollup/plugin-commonjs';


import pkg from './package.json';

export default [
  {
    input: 'src/index.ts',
    output: {
      dir: 'dist/umd',
      format: 'umd',
      name: 'cnUtils',
    },
  },
  {
    input: ['src/index.ts', 'src/**/index.ts'],
    output: {
      dir: 'dist/cjs',
      format: 'cjs',
    },
  },
  {
    input: ['src/index.ts', 'src/**/index.ts'],
    output: {
      dir: 'dist/es',
      format: 'es',
    },
  },
].map((config) => ({
  ...config,
  output: {
    ...config.output,
    exports: 'named',
    sourcemap: true,
    banner: `/*!
* ${pkg.name} v${pkg.version}
*
* @license ${pkg.license}
* @build at ${new Date().toLocaleString()}
*/`,
  },
  plugins: [
    multiInput(),
    typescript({
      tsconfig: './tsconfig.json',
      outDir: config.output.dir,
      declaration: false,
    }),
    nodeResolve(),
    commonjs(),
  ],
}));
```