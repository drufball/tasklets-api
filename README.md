# Problem
Most modern development platforms favor a multi-threaded approach by default. Typically, the split for work is:
- __Main thread__: UI manipulation, event/input routing
- __Background thread__: All other work
iOS and Android native platforms, for example, restrict (by default) the usage of any APIs not critical to UI manipulation on the main thread.

The web has support for this model via `WebWorkers`, though the `postMessage()` interface is clunky and difficult to use. As a result, worker adoption has been minimal at best and the default model is still all work on the main thread. In order to encourage worker adoption, we need to explore a more ergonomic API.

In anticipation of increased `WebWorker` usage, we should also address potential resource overhead concerns that may come from heavier worker usage.

# API
__Note__: APIs described below are just strawman proposals. We think they're pretty cool but there's always room for improvement.

Today, many uses of `WebWorker`s follow a structure similar to:

```javascript
const worker = new Worker();
worker.postMessage({'cmd':'fetch', 'url':'example.com'});
```

A switch statement in the worker then routes messages to the correct API. The tasklet API exposes this behavior natively, by allowing workers to expose methods to outside contexts:

```javascript
class Speaker {
  // We have to list what methods are exposed.
  static get exposed() { return ['concat']; }

  concat(message) {
    return `${message} world!`;
  }
}
services.register('speak', Speaker);
```

From the main thread, we can access these exposed methods directly, awaiting them as we would awwait a normal promise:
