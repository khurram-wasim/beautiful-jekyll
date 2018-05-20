---
layout: post
title: Server side rendering in Angular 5
subtitle: Server side rendering in Angular cli applications using Angular Universal
tags: [angular universal, angular cli]
---

This is a comprehensive guide to do server side rendering in angular cli applications,
as it also discusses the problems and their solutions which you can encounter along the
way.

# 1. Setup to create a build

## Install Dependencies

First you need to install following bundles by doing:

```javascript

npm install --save @angular/platform-server @nguniversal ts-loader webpack-node-externals
npm install --save domino fs path zone.js node-fetch reflect-metadata

```

Make sure you use the same version as the other @angular packages in your project.

## Preparing app for Server side rendering


### Step a: Edit app.module.ts

You would have edit you app.module.ts, by adding a .withServerTransition() property
along with your application id:

#### src/app/app.module.ts:

```javascript

@NgModule({
  bootstrap: [AppComponent],
  imports: [
    // Add .withServerTransition() to support Universal rendering.
    // The application ID can be any identifier which is unique on
    // the page.
    BrowserModule.withServerTransition({appId: 'my-app'}),
    ...
  ],

})
export class AppModule {}

```


### Step b: Create app.server.module.ts

Duplicate the file app.module.ts as app.server.module.ts and place the following
content in the file

#### src/app/app.server.module.ts:

```javascript

import {NgModule} from '@angular/core';
import {ServerModule} from '@angular/platform-server';
import {ModuleMapLoaderModule} from '@nguniversal/module-map-ngfactory-loader';

import {AppModule} from './app.module';
import {AppComponent} from './app.component';

@NgModule({
  imports: [
    // The AppServerModule should import your AppModule followed
    // by the ServerModule from @angular/platform-server.
    AppModule,
    ServerModule,
    ModuleMapLoaderModule // <-- *Important* to have lazy-loaded routes work
  ],
  // Since the bootstrapped component is not inherited from your
  // imported AppModule, it needs to be repeated here.
  bootstrap: [AppComponent],
})
export class AppServerModule {}

```


### Step c: Create main.server.ts and tsconfig.server.json

Create main.server.ts in src directory, its content would be:

#### src/main.server.ts:

```javascript

export { AppServerModule } from './app/app.server.module';

```

Duplicate the file tsconfig.app.json as tsconfig.server.json. Change it to build
with a "module" target of "commonjs".
Add a section "angularCompilerOptions" in it and set the "entryModule" to
AppServerModule. Now the file would look like:

#### src/tsconfig.server.json:

```javascript

{
  "extends": "../tsconfig.json",
  "compilerOptions": {
    "outDir": "../out-tsc/app",
    "baseUrl": "./",
    // Set the module format to "commonjs":
    "module": "commonjs",
    "types": []
  },
  "exclude": [
    "test.ts",
    "**/*.spec.ts"
  ],
  // Add "angularCompilerOptions" with the AppServerModule you wrote
  // set as the "entryModule".
  "angularCompilerOptions": {
    "entryModule": "app/app.server.module#AppServerModule"
  }
}

```


### Step d: Edit angular-cli.json

You would have one object in **apps** array in angular-cli.json.
Change **outDir** of this object to *dist/browser*.

Now, duplicate this object inside this array. Inside the second object:

1. Add a property **"platform": "server"**.
2. Change **outDir** to be *"dist/server"*.
3. Change **main** to be *"main.server.ts"*.
4. Change **tsconfig** to be *"tsconfig.server.json"*.



### Step e: Configure usage of absoulte paths in your applications

If you have used absolute paths in your app you might encounter errors while
building your bundle. To fix those errors do the following:

#### ./tsconfig.json

Set the property  **baseUrl** to **src**

**Congratulations !!** Your bundle is ready to be built.



# 2. Setting up an Express Server to run our Universal bundles

Now you have your bundle built, but how would we run it ? You can create an
express server to do this. Simply create a server.ts in the root of your project
and paste the following content:

#### ./server.ts (at the root of project)

```javascript

// These are important and needed before anything else
import 'zone.js/dist/zone-node';
import 'reflect-metadata';

import { renderModuleFactory } from '@angular/platform-server';
import { enableProdMode } from '@angular/core';

import * as express from 'express';
import { join } from 'path';
import { readFileSync } from 'fs';

// Faster server renders w/ Prod mode (dev mode never needed)
enableProdMode();


// Express server
const app = express();

var compress = require('compression');
app.use(compress());

const PORT = process.env.PORT || 4000;
const DIST_FOLDER = join(process.cwd(), 'dist');

// Our index.html we'll use as our template
const template = readFileSync(join(DIST_FOLDER, 'browser', 'index.html')).toString();


// Hack for server side rendering starts here
const domino = require('domino');
const fs = require('fs');
const path = require('path');
const Zone = require('zone.js');
import fetch from 'node-fetch';

const win = domino.createWindow(template);
const files = fs.readdirSync(`${process.cwd()}/dist/server`);

win.fetch = fetch;
global['window'] = win;
global['document'] = win.document;
global['navigator'] = win.navigator;

// Hack for server side rendering ends here


// * NOTE :: leave this as require() since this file is built Dynamically from webpack
const { AppServerModuleNgFactory, LAZY_MODULE_MAP } = require('./dist/server/main.bundle');

const { provideModuleMap } = require('@nguniversal/module-map-ngfactory-loader');

app.engine('html', (_, options, callback) => {
  renderModuleFactory(AppServerModuleNgFactory, {
    // Our index.html
    document: template,
    url: options.req.url,
    // DI so that we can get lazy-loading to work differently (since we need it to just instantly render it)
    extraProviders: [
      provideModuleMap(LAZY_MODULE_MAP)
    ]
  }).then(html => {
    callback(null, html);
  });
});

app.set('view engine', 'html');
app.set('views', join(DIST_FOLDER, 'browser'));

// Server static files from /browser
app.get('*.*', express.static(join(DIST_FOLDER, 'browser')));

// All regular routes use the Universal engine
app.get('*', (req, res) => {
  res.render(join(DIST_FOLDER, 'browser', 'index.html'), { req });
});

// Start up the Node server
app.listen(PORT, () => {
  console.log(`Node server listening on http://localhost:${PORT}`);
});


```

## Universal gotchas

In the above script there is section of a hack for server side rendering
Angular Universal does not support DOM elements like **document, event, navigator etc.**
This is because when you develop an application, for best practices **you should
follow the guidelines and facilities provided by the framework you are using**.

In my case some of the external modules were using those DOM elements (window,
document and navigator.) To fix this issue I included the module **domino** which provides
DOM at the backend.

Another sample usage can also be found [here](https://github.com/angular/universal/issues/830).


# 3. Setup a webpack config to handle this Node server.ts file and serve your application!

The file we are going to create takes server.ts, compiles it and every dependency
it has into dist/server.js.
So, create webpack.server.config.js at the root.

#### ./webpack.server.config.js (at the root of project)

```javascript

const path = require('path');
const webpack = require('webpack');
var nodeExternals = require('webpack-node-externals');

module.exports = {
  entry: {  server: './server.ts' },
  resolve: { extensions: ['.js', '.ts'] },
  target: 'node',

  externals: [
    nodeExternals({
      whitelist: [/zone.js/, /reflect-metadata/]
    })
  ],
  output: {
    path: path.join(__dirname, 'dist'),
    filename: '[name].js'
  },
  module: {
    rules: [
      { test: /\.ts$/, loader: 'ts-loader' }
    ]
  },
  plugins: [
    // Temporary Fix for issue: https://github.com/angular/angular/issues/11580
    // for "WARNING Critical dependency: the request of a dependency is an expression"
    new webpack.ContextReplacementPlugin(
      /(.+)?angular(\\|\/)core(.+)?/,
      path.join(__dirname, 'src'), // location of your src
      {} // a map of your routes
    ),
    new webpack.ContextReplacementPlugin(
      /(.+)?express(\\|\/)(.+)?/,
      path.join(__dirname, 'src'),
      {}
    )
  ]
}

```

### nodeExternals
  In the above script,
  Inside **nodeExternals** you can pass a object of **whiteList**, containing an array of
  external modules **which you do require in your project**.
  (By default it does not bundle any module from node_modules folder)

  **TIP !** You should add modules to whitelist after you build and run the project and
  encounter errors about those modules.
  In my case those were zone.js and reflect-metadata




# 4. Finalizing things

Add these 4 following scripts in package.json
```javascript

"build:ssr": "npm run build:client-and-server-bundles && npm run webpack:server",

"serve:ssr": "node dist/server.js",

"build:client-and-server-bundles": "ng build --prod && ng build --prod --app 1 --output-hashing=false",

"webpack:server": "webpack --config webpack.server.config.js --progress --colors"

```

## To build and run the project

```javascript

npm run build:ssr && npm run serve:ssr

```
**GOOD LUCK !!**
