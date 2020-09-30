project_path: /web/tools/workbox/_project.yaml
book_path: /web/tools/workbox/_book.yaml
description: Common recipes to use with Workbox.

{# wf_updated_on: 2020-05-01 #}
{# wf_published_on: 2017-11-15 #}
{# wf_blink_components: N/A #}

# Common Recipes {: .page-title }

This page contains a set of example caching strategies you can use with Workbox.

## Google Fonts

The Google Fonts service consists of two parts:

- The stylesheet with the `@font-face` definitions, which link to the font
  files.
- The static, revisioned font files.

The stylesheet can change frequently, so it's best to use a caching strategy
like stale while revalidate that checks for updates with every request. The font
files themselves, on the other hand, do not change and can leverage a cache
first strategy.

Here we've limited the age of the cached font files to one year (matching the
HTTP `Cache-Control` header) and the max entries to 30 (to ensure we don't use
up too much storage on the user's device).

```javascript
import {registerRoute} from 'workbox-routing';
import {CacheFirst, StaleWhileRevalidate} from 'workbox-strategies';
import {CacheableResponsePlugin} from 'workbox-cacheable-response';
import {ExpirationPlugin} from 'workbox-expiration';

// Cache the Google Fonts stylesheets with a stale-while-revalidate strategy.
registerRoute(
  ({url}) => url.origin === 'https://fonts.googleapis.com',
  new StaleWhileRevalidate({
    cacheName: 'google-fonts-stylesheets',
  })
);

// Cache the underlying font files with a cache-first strategy for 1 year.
registerRoute(
  ({url}) => url.origin === 'https://fonts.gstatic.com',
  new CacheFirst({
    cacheName: 'google-fonts-webfonts',
    plugins: [
      new CacheableResponsePlugin({
        statuses: [0, 200],
      }),
      new ExpirationPlugin({
        maxAgeSeconds: 60 * 60 * 24 * 365,
        maxEntries: 30,
      }),
    ],
  })
);
```

## Caching Images

You might want to use a cache-first strategy for images, by matching against the intended
[destination](https://developer.mozilla.org/en-US/docs/Web/API/Request/destination) of the request.

```javascript
import {registerRoute} from 'workbox-routing';
import {ExpirationPlugin} from 'workbox-expiration';

registerRoute(
  ({request}) => request.destination === 'image',
  new CacheFirst({
    cacheName: 'images',
    plugins: [
      new ExpirationPlugin({
        maxEntries: 60,
        maxAgeSeconds: 30 * 24 * 60 * 60, // 30 Days
      }),
    ],
  })
);
```

## Cache CSS and JavaScript Files

You might want to use a stale-while-revalidate strategy for CSS and JavaScript files that aren't
precached, by matching against the
[destination](https://developer.mozilla.org/en-US/docs/Web/API/Request/destination) of the incoming
request.

```javascript
import {registerRoute} from 'workbox-routing';
import {StaleWhileRevalidate} from 'workbox-strategies';

registerRoute(
  ({request}) => request.destination === 'script' ||
                  request.destination === 'style',
  new StaleWhileRevalidate({
    cacheName: 'static-resources',
  })
);
```

## Caching Content from Multiple Origins

You can create regular expressions to cache similar requests from multiple
origins in a single route by combining multiple checks into a single
[`matchCallback` function](/web/tools/workbox/reference-docs/latest/module-workbox-routing#~matchCallback).

```javascript
import {registerRoute} from 'workbox-routing';
import {StaleWhileRevalidate} from 'workbox-strategies';

registerRoute(
  ({url}) => url.origin === 'https://fonts.googleapis.com' ||
             url.origin === 'https://fonts.gstatic.com',
  new StaleWhileRevalidate(),
);
```

## Restrict Caches for a Specific Origin

You can cache assets for a specific origin and apply expiration rules on
that cache. For example, the example below caches up to 50 requests for
up to 5 minutes.

```javascript
import {registerRoute} from 'workbox-routing';
import {CacheFirst} from 'workbox-strategies';
import {CacheableResponsePlugin} from 'workbox-cacheable-response';
import {ExpirationPlugin} from 'workbox-expiration';

registerRoute(
  ({url}) => url.origin === 'https://hacker-news.firebaseio.com',
  new CacheFirst({
      cacheName: 'stories',
      plugins: [
        new ExpirationPlugin({
          maxEntries: 50,
          maxAgeSeconds: 5 * 60, // 5 minutes
        }),
        new CacheableResponsePlugin({
          statuses: [0, 200],
        }),
      ],
  })
);
```

## Force a Timeout on Network Requests

There may be network requests that would be beneficial if they were served
from the network, but could benefit by being served by the cache if the
network request is taking too long.

For this, you can use a `NetworkFirst` strategy with the
`networkTimeoutSeconds` option configured.

```javascript
import {registerRoute} from 'workbox-routing';
import {NetworkFirst} from 'workbox-strategies';
import {ExpirationPlugin} from 'workbox-expiration';

registerRoute(
  ({url}) => url.origin === 'https://hacker-news.firebaseio.com',
  new NetworkFirst({
    networkTimeoutSeconds: 3,
    cacheName: 'stories',
    plugins: [
      new ExpirationPlugin({
        maxEntries: 50,
        maxAgeSeconds: 5 * 60, // 5 minutes
      }),
    ],
  })
);
```

## Cache Resources from a Specific Subdirectory

You can route requests to files in a specific directory on your local web app by checking the
`origin` and `pathname` properties of the
[URL object](https://developer.mozilla.org/en-US/docs/Web/API/URL) passed to the
[`matchCallback` function](/web/tools/workbox/reference-docs/latest/module-workbox-routing#~matchCallback):

```javascript
import {registerRoute} from 'workbox-routing';
import {StaleWhileRevalidate} from 'workbox-strategies';

registerRoute(
  ({url}) => url.origin === self.location.origin &&
             url.pathname.startsWith('/static/'),
  new StaleWhileRevalidate()
);
```

## Cache Resources Based on Resource Type

You can use the
[`RequestDestination`](https://developer.mozilla.org/en-US/docs/Web/API/RequestDestination)
enumerate type of the
[destination](https://medium.com/dev-channel/service-worker-caching-strategies-based-on-request-types-57411dd7652c)
of the request to determine a strategy.
For example, when the target is `<audio>` data:

```javascript
import {registerRoute} from 'workbox-routing';
import {CacheFirst} from 'workbox-strategies';
import {ExpirationPlugin} from 'workbox-expiration';

registerRoute(
  // Custom `matchCallback` function
  ({request}) => request.destination === 'audio',
  new CacheFirst({
    cacheName: 'audio',
    plugins: [
      new ExpirationPlugin({
        maxEntries: 60,
        maxAgeSeconds: 30 * 24 * 60 * 60, // 30 Days
      }),
    ],
  })
);
```

## Access Caches from Your Web App's Code

The [Cache Storage
API](/web/fundamentals/instant-and-offline/web-storage/cache-api) is available
for use in both service worker and in the context of `window` clients. If you
want to make changes to caches—add or remove entries, or get a list of cached
URLs—from the context of your web app, you can do so directly, without having to
communicate with the service worker via `postMessage()`.

For instance, if you wanted to add a URL to the a given cache in response to a
user action in your web app, you can use code like the following:

```javascript
// Inside app.js:

async function addToCache(urls) {
  const myCache = await window.caches.open('my-cache');
  await myCache.addAll(urls);
}

// Call addToCache whenever you'd like. E.g. to add to cache after a page load:
window.addEventListener('load', () => {
  // ...determine the list of related URLs for the current page...
  addToCache(['/static/relatedUrl1', '/static/relatedUrl2']);
});
```

The cache name, `'my-cache'`, can then be referred to when setting up a route in
your service worker, and that route can take advantage of any cache entries
that were added by the web page itself:

```javascript
// Inside service-worker.js:

import {registerRoute} from 'workbox-routing';
import {StaleWhileRevalidate} from 'workbox-strategies';

registerRoute(
  ({url}) => url.origin === self.location.origin &&
             url.pathname.startsWith('/static/'),
  new StaleWhileRevalidate({
    cacheName: 'my-cache', // Use the same cache name as before.
  })
);
```