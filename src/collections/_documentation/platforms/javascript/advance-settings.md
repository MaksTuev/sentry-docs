---
title: Advanced Usage
sidebar_order: 30001
---

## Using a Client directly

To be able to manage several Sentry instances without any conflicts between them you need to create your own `Client`.
This also helps to prevent tracking of any parent application errors in case your application is integrated
inside of it. In this example we use `@sentry/browser` but it's also applicable to `@sentry/node`.

```javascript
import { BrowserClient } from "@sentry/browser";

const client = new BrowserClient({
  dsn: '___PUBLIC_DSN___',
});

client.captureException(new Error('example'));
```

While the above sample should work perfectly fine, some methods like `configureScope` and `withScope` are missing on the `Client` because the `Hub` takes care of the state management. That's why it may be easier to create a new `Hub` and bind your `Client` to it. The result is the same but you will also get state management with it.

```javascript
import { BrowserClient, Hub } from "@sentry/browser";

const client = new BrowserClient({
  dsn: '___PUBLIC_DSN___',
});

const hub = new Hub(client);

hub.configureScope(function(scope) {
  scope.setTag("a", "b");
});

hub.addBreadcrumb({ message: "crumb 1" });
hub.captureMessage("test");

try {
  a = b;
} catch (e) {
  hub.captureException(e);
}

hub.withScope(function(scope) {
  hub.addBreadcrumb({ message: "crumb 2" });
  hub.captureMessage("test2");
});
```

### Dealing with integrations

Integrations are setup on the `Client`, if you need to deal with multiple clients and hubs you have to make sure to also do the integration handling correctly. 
Here is a working example of how to use multiple clients with multiple hubs running global integrations.

```js
import * as Sentry from "@sentry/browser";

// Very happy integration that'll prepend and append very happy stick figure to the message
class HappyIntegration {
  constructor() {
    this.name = "HappyIntegration";
  }

  setupOnce() {
    Sentry.addGlobalEventProcessor(event => {
      const self = Sentry.getCurrentHub().getIntegration(HappyIntegration);
      // Run the integration ONLY when it was installed on the current Hub
      if (self) {
        event.message = `\\o/ ${event.message} \\o/`;
      }
      return event;
    });
  }
}

HappyIntegration.id = "HappyIntegration";

const client1 = new Sentry.BrowserClient({
  dsn: "___PUBLIC_DSN___",
  integrations: [...Sentry.defaultIntegrations, new HappyIntegration()],
  beforeSend(event) {
    console.log("client 1", event);
    return null; // Returning null does not send the event
  }
});
const hub1 = new Sentry.Hub(client1);

const client2 = new Sentry.BrowserClient({
  dsn: "___PUBLIC_DSN___", // Can be a different DSN
  integrations: [...Sentry.defaultIntegrations, new HappyIntegration()],
  beforeSend(event) {
    console.log("client 2", event);
    return null; // Returning null does not send the event
  }
});
const hub2 = new Sentry.Hub(client2);

hub1.run(currentHub => { // The hub.run method makes sure that Sentry.getCurrentHub() returns this hub during the callback
  currentHub.captureMessage("a");
  currentHub.configureScope(function(scope) {
    scope.setTag("a", "b");
  });
});

hub2.run(currentHub => { // The hub.run method makes sure that Sentry.getCurrentHub() returns this hub during the callback
  currentHub.captureMessage("x");
  currentHub.configureScope(function(scope) {
    scope.setTag("c", "d");
  });
});
```
