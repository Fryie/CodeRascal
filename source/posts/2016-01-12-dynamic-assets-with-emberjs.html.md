---
title: Dynamic assets with EmberJS
date: 2016-01-12 00:00 CET
category: javascript
tags: javascript, EmberJS, deployment
---

Sometimes, in a web app you might have assets whose paths are dynamically generated. For example, if you have a store, you might have several categories of items and each of them might have an icon. In EmberJS, the asset path to that icon might look something like this:

READMORE

```javascript
`assets/icons/${category.get('name')}.svg`
```


This works fine in development and also in the most naive of production environments where the asset URLs remain unchanged. However, typically you'll want to fingerprint your assets so that outdated assets are not loaded from the browser cache. In many cases (and certainly in the most common [ember-cli-deploy](needsurl) setup) the assets will also be served over a CDN.

With dynamic asset paths, this turns out to be a problem, as [several](forumthread) [discussions](github) highlight. The reason is that [broccoli-assets-rev](link), which replaces asset URLs with their fully resolved production counterparts, works in a way that is too simple to accomodate this setting: It basically just looks for strings that contain URLs to assets ([here](...) is the code that does that). This doesn't work with strings whose exact content is determined only at runtime (and not at build time). In that case, the ES6 code compiles to something like:

```javascript
'assets/icons/' + category.get('name') + '.svg'
```

and Broccoli isn't able to replace that correctly.

It took me many tries to find a working solution for this problem and I also had to build upon ideas from .... The key lies in the *asset map* that Ember generates upon building the app: It's a map of development asset URLs to the fully resolved production URLs. Normally, this is discarded after the build, but if we can keep it around in production we can use it to resolve asset paths at runtime.

So the first step is to enable the generation of the asset map. But, very importantly, we also have to *fingerprint* the asset map. Otherwise, a client might load an outdated version of the asset map causing URL resolution problems when an asset was added or modified. To do both of these things, you should add the following to your *ember-cli-build.js*:

```javascript
fingerprint: {
  ...
  generateAssetMap: true,
  fingerprintAssetMap: true
}
```
(If I remember correctly, you need to have a somewhat recent version of *broccoli-asset-rev* for that.)

But now comes the slightly trickier part: You have to create some sort of service that is able to map development URLs to production URLs at runtime. Ideally, the asset map is loaded only once (which also means that you don't have to wrap all your URL lookups in promises), so it makes sense to set the service up in an initializer:

```javascript
import Ember from 'ember';
import ENV from '<your-app>/config/environment';

export function initialize(container, application) {
  var AssetMap;

  application.deferReadiness();

  if (!ENV.APP.environment === 'production') {
    // use an asset map stub in development / testing
    AssetMap = Ember.Object.extend({
      resolve(path) {
        return path;
      }
    });

    container.register('assetMap:main', AssetMap, { singleton: true });
    application.inject('service:assets', 'assetMap', 'assetMap:main');
    application.advanceReadiness();
  } else {
    AssetMap = Ember.Object.extend();

    var promise = new Ember.RSVP.Promise(function(resolve, reject) {
      // Broccoli will replace this string with the properly resolved URL when building
      var assetMapURL = 'assets/assetMap.json';
      Ember.$.getJSON(assetMapURL, resolve).fail(reject);
    });

    promise.then(function(assetMap) {
      AssetMap.reopen({
        assetMap: assetMap,
        resolve: function(path) {
          return `${assetMap.prepend}${assetMap.assets[path]}`;
        }
      });
    }, function() {
      AssetMap.reopen({
        resolve: function(path) {
          return path;
        }
      });
    }).then(function() {
      container.register('assetMap:main', AssetMap, { singleton: true });
      application.inject('service:assets', 'assetMap', 'assetMap:main');
      application.advanceReadiness();
    });
  }
}

export default {
  name: 'asset-map',
  initialize: initialize
};
```

This code looks somewhat complex, so let's break it down a little. First, we block the initialisation of our app until we are actually done with our setup (this is the *application.deferReadiness()* call). We set up an *assetMap* service that exposes a *resolve* method. First, outside of production, we're just using a stub. The condition for that first branch can be changed, of course, in case you have a staging environment that also needs to have fingerprinted assets etc. In this case, the *resolve* method just returns the original path. After registering the service in the container and injecting it (where?), we can resume the application loading with *application.advanceReadiness()*.

The more complex setup happens when we actually have fingerprinted assets. Here, we first create a promise that will load the asset map JSON. Here, we can take advantage of *broccoli-asset-rev*'s default behaviour: It will replace this (static) URL to the asset map with the fully resolved one (remember that we chose to fingerprint the asset map, too).

We then wait for the promise to resolve (this is why it was important to defer the application readiness - otherwise the app would run before the *AssetMap* object was fully set up). In the case the promise fails to resolve, we just use the same *AssetMap* stub as in development, but if the asset map was successfully loaded, we pass it in to the *AssetMap* object (this wouldn't be strictly necessary). The *resolve* function now computes the resolved URL by first prepending the CDN path and then adding the fingerprinted asset path. Since the asset map is in the closure, the method doesn't need to make any additional network request any more when resolving an asset. Finally, we register and inject the *AssetMap* object and then finally allow the initialisation of the app to continue.
