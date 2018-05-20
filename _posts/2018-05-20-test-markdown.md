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

## Preparing app for Server side rendering

You would have edit you app.module.ts, by adding a .withServerTransition() property
along with your application id:

### src/app/app.module.ts:

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
