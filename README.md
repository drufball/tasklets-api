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
const worker = new Worker('worker.js');
worker.postMessage({'cmd':'fetch', 'url':'example.com'});
```

A switch statement in the worker then routes messages to the correct API. The tasklet API exposes this behavior natively, by allowing workers to expose methods to outside contexts:

```javascript
// tasklet.js
class Speaker {
  // We have to list what methods are exposed.
  static get exposed() { return ['concat']; }

  concat(message) {
    return `${message} world!`;
  }
}
services.register('speak', Speaker);
```

From the main thread, we can access these exposed methods directly, awaiting them as we would await a normal promise:

```javascript
// main.html
const tasklet = new Tasklet('tasklet.js');
const speaker = await tasklet.connect('speak');

speaker.concat('Hello');
```

This makes interfacing with a worker much easier for computationally expensive tasks. However, it is still difficult to drive complex chains of work from a background thread, since any UI updates require bouncing back to the main thread. We can solve this problem with the same API but in reverse: the main thread can expose methods to background workers:

```javascript
// main.html
const tasklet = new Tasklet('tasklet.js');

tasklet.register('dom', class {
  // We have to list what methods are exposed.
  static get exposed() { return ['appendText', 'appendBox']; }

  appendText(text) {
    let div = document.createElement('div');
    document.body.appendChild(div).textContent = text;
  }

  appendBox(text, style) {
    let div = document.createElement('div');
    if (style) {
      for (let [property, value] of style)
        div.style[property] = value;
    }
    document.body.appendChild(div).textContent = text;
  }
});
```

And then access those methods within the worker, awaiting the promise (executed on the main thread) and then continuing without breaking the control flow:

```javascript
// tasklet.js
const dom = await services.connect('dom');

// Kick off some main thread work to update the UI
let text = "hello";
dom.appendText(text);

// And then continue with our work here
let box = " world!"
dom.appendBox(box);
```

# ES language support
While using this API provides numerous improvements, there are still some rough edges. There is a fair amount of boilerplate to register functions. A site's logic is split between multiple files for main and background work. There's no simple way to just run a function in the background.

A `remote` keyword could be used to signify a class that was meant to run on the background and replace all of the boilerplate:

```javascript
// tasklet.js
remote class Speaker {
  concat(message) {
    return `${message} world!`;
  }
}
```

```javascript
// main.html
import tasklet.js;

const speaker = new Speaker();
speaker.concat('Hello');
```

We could replace the need to register main thread APIs by allowing developers to pass a reference to the current context:

```javascript
// main.html
import tasklet.js; // defines TextAdder
import dom.js; // defines appendText(text) and appendBox(text, style)

const textAdder = new textAdder(document);
textAdder.doSomething();
```

```javascript
// tasklet.js
remote class TextAdder {
  constructor(context) {
    this.dom = context;
  }

  async doSomething() {
    await this.dom.appendText('Hello');
    await this.dom.appendBox(' World!');
  }
}
```

If we allowed for inline definitions of remote classes, we get a particularly compact API. Using web components:

```html
<template id="my-element">
  <div id="color">Red</div>
  <button onclick="swap">Click me!</button>
</template>
<script>
  class MyElement extends HTMLElement {
    constructor() {
      super();

      // Import template markup
      let markup = document.querySelector('#my-element'); // imagine this works
      let shadowRoot = this.attachShadow({mode:'open'});
      shadowRoot.appendChild(markup.content.cloneNode(true));

      // Initialize backgound work
      this.tasklet = new MyElementTasklet(this);
    }

    swap() {
      const currentColor = this.shadowRoot.querySelector('#text').textContent;
      this.tasklet.updateColor(currentColor);
    }

    updateUI(newColor) {
      this.shadowRoot.querySelector('#text').textContent = newColor;
    }
  }

  remote class MyElementTasklet {
    constructor(context) {
      this.dom = context;
    }

    updateColor(currentColor) {
      if(currentColor == 'Red') {
        this.dom.updateUI('Blue');
      }
      else {
        this.dom.updateUI('Red');
      }
    }
  }

  window.customElements.define('my-element', MyElement);
</script>
```

# A default worker
When composing functionality (via iframes or just web components), there is some concern that an abundance of workers may lead to memory bloat. Developers will need some way to share worker resources, even in cases where they do not have direct control of the code they are embedding.

iOS and Android both have a concept of a default background thread. This allows easy coordination of resources by default. We propose that any services registered via this API would share a common worker, unless otherwise specified:

```javascript
// tasklet.js
services.register('speak', Speaker, 'custom-worker');
```
