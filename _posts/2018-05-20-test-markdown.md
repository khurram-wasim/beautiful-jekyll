---
layout: post
title: Server side rendering in Angular Cli Applications
subtitle: Server side rendering using Angular Universal
tags: [angular universal, angular cli]
---



# Setup to create a build

## Install Dependencies

First you need to install three bundles by doing:

```javascript

npm install --save @angular/platform-server @nguniversal ts-loader

```

Make sure you use the same version as the other @angular packages in your project.

## Preparing app for Server side rendering

### Step 1

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
### Step 2

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

### Step 3

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
  {   ""extendsextends""::  ""../tsconfig.json../tsconfig.json"",
  ,   ""compilerOptionscompilerOptions"":: {
     {     ""outDiroutDir""::  ""../out-tsc/app../out-tsc/app"",
    ,     ""baseUrlbaseUrl""::  ""././"",
    ,     //// Set the module format to "commonjs": Set the module format to
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
