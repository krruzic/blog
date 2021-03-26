+++
title = "(p)React: the good parts, from a backend developer"
date = 2021-03-06
draft = true
type = "post"

[taxonomies]
tags = ["react","frontend","web"]
+++

Web development has always been the bane of my existence. I prefer working on
the backend where I don't need to worry about display resolution or supporting
different devices. Occasionally I'm forced to write code that runs in a browser
however, and up until now I've gotten away with just jquery and some jinja
templates. I hate modern web UI frameworks but I hate the spaghetti code I
always end up with otherwise. So about a week ago I picked up react. Well to be
more specfic, preact, an API compatible minimal alternative. I'm lazy so I
didn't really look into the differences that much, the main reason I picked it
was because of their work towards first class support for buildless usage.

<!-- more --> 
# LOOK MA NO BUILD

I don't know what it is about npm that causes it to be so slow no matter how
basic the task, but because of this there no portion of the JS ecosystem I hate
more than it. So if I'm going to use preact, I need to get rid of this entirely.
I'm not going to be pulling in a million ridiculous dependencies myself, so
doing this shouldn't be too difficult. Especially since preact has no-build
instructions at the
[https://preactjs.com/guide/v10/getting-started#no-build-tools-route](very top
of their docs)! Right? Wrong. The instructions there only work to a point: if
you have everything in one file. And as mentioned above I have no idea how
javascript build systems work so I couldn't debug it. With some extensive
googling (could react terminology be more generic?) I found a
[https://blog.jim-nielsen.com/2020/react-without-build-tools/](blog entry)
discussing doing something similar and had all I needed.

First you're going to want to import all your requirements in your index.html.
If a package releases a `umd.js` file it will likely work like the ones below.

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8" />
    <title>preact experiment</title>
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <!-- necessary -->
    _
    <script src="//unpkg.com/htm@3.0.4/dist/htm.js"></script>
    <script src="//unpkg.com/preact@10.5.2/dist/preact.umd.js"></script>
    <script src="//unpkg.com/preact@10.5.2/hooks/dist/hooks.umd.js"></script>
    <!-- <script src="//unpkg.com/preact@10.5.2/compat/dist/compat.umd.js"></script> -->
    <!-- tested -->
    <!-- <script src="//unpkg.com/prop-types@15.7.2/prop-types.min.js"></script> -->
    <!-- <script src="//unpkg.com/unistore@3.5.2/full/preact.umd.js"></script> -->
    <!-- <script src="//unpkg.com/@preact-hooks/unistore@1.1.2/dist/hooks.umd.js"></script> -->
    <!-- dev dependencies: debugger -->
    <script src="//unpkg.com/preact@10.5.2/devtools/dist/devtools.umd.js"></script>
    <script src="//unpkg.com/preact@10.5.2/debug/dist/debug.umd.js"></script>
  </head>
  <body>
    <script src="app.js" type="module"></script>
  </body>
</html>
```

In the above file I've imported quite a few javascript modules... the important
ones are `htm`, `preact` and `hooks`. We use `htm` to replace jsx through the
use of ES6 template-literal string syntax. The `hooks` package provides the
mondern react component building framework(?), you don't need it but I prefer
the more concise syntax (although it has some confusing concepts). The
`preactcompat` package is useful as well but only if you want to use apis that
aren't implented in preact yet but exist in react such as `createPortal` to
render outside of the app root.

## Organizing our dependencies

In order to make using our js dependencies easier, we first want to re-export
them under a common namespace (to avoid re-importing and limitations to a
no-build system). As per Jim Neilsen's blog, we just do this all in a `deps.js`.
The following deps file exports everything you'll need to use hooks, htm and
preact across your app and corresponds to the uncommented bits of the
`index.html` file above.

```js
const { h, Component, render, createContext } = window.preact;
const html = window.htm.bind(h); // setup htm
const { useState, useEffect, useRef } = window.preactHooks;

export {
  h,
  Component,
  render,
  createContext,
  html,
  useState,
  useEffect,
  useRef,
};
```

## Normal Preact From Here

At this point you can write preact applications exactly like any tutorial
explains, with one exception: `htm` instead of `jsx`.

### htm: just add a dollar sign!

`htm` is used for our DOM rendering. Just like in jsx we can write what appears
to be plain html, but without all the fuss! Using template literals we can
include any javascript expression into our html to dynamically affect our
functions return value.

```js
const App = () => {
  return html`<h1>Hello, World!</h1>`;
};
render(h(App), document.body);
```

Don't worry jquery fans, your \$ is back. Using `htm` is no different than jsx,
except for that little bugger. If you're porting jsx code from some component,
the first thing you'll want to do is wrap it all in an `html` and add your money
sign. If you're unfamiliar with jsx, it's basically a template system where you
can inject variables that will be parsed into the generated html. `htm` does the
same thing, so you can access a var `myName` inside your html block like
`${myName}`. Two gotchas a noob must know are below.

```js
// Setting a callback for a button:
// with no args, works!
return html`<button onClick=${blah}>click me lol</button>`;
// DON'T DO THIS WITH ARGS!
// This will call the function as part of the rendering
return html`<button onClick=${blah(10)}>click me lol</button>`;
// DO THIS INSTEAD!
// Make an arrow function that calls your function with arguments
return html`<button onClick=${() => blah(10)}>I work!</button>`;
```

```js
// iterating over a collection,
// no need to worry about <Fragments> like in jsx
// but any expression used within MUST return a value.
// Use map() NOT iter() etc.
return html\`${Object.keys(data).map(k => html`<p>${k} is ${data[k]}</p>\`)}\`

```

### Hooks and Accessing Their State?

Fancy javascript nerds have become fixated on mathematical terms and have
decided to make everything stateless and magic for no real reason. That's why
everything returns a string[1]. I don't like this design philosophy and I find
it pretty outrageous to call it stateless but whatever. All it means is more
headaches and ungooglable phrases from my perspective. Here's how to make it
easier. When working with hooks managing state between renders is somewhat
confusing. If you're just storing a bool that changes on a button press, you
need to do something like this

```js
import { h, html, render, useEffect, useRef } from "./deps.js";
const ButtonToggler = () => {
  // the first var is the state, the second a function that updates that var
  const [buttonState, setButtonState] = useState(false);
  return html`
    <button onClick=${() => setButtonState(!buttonState)}
      <h1>${buttonState}</h1>
    </button>`;
};
```

Simple enough, I guess? But what if you're storing more complex data, like an
object. Or an array. Or an array of objects? Remember, these components are
stateless, so in order to "update" something like this we need to first copy the
old state back in. The below example stores a list of buttons and their state,
while providing a function for increasing the number of buttons

```js
import {h, html, render, useEffect, useRef} from "./deps.js";
const ButtonToggler = () => {
    // the first var is the state, the second a function that updates that var
    const addButton = ())
    const [buttonsState, setButtonsState] = useState([]);
    return html`
    <button onClick=${(() => setButtonState(!buttonState))}
      <h1>${buttonState}</h1>
    </button>`
}

```

```js
"use strict";

import {
  render,
  h,
  html,
  createStore,
  Provider,
  connect,
  PropTypes,
  Component,
  useState,
  useEffect,
  useRef,
  ReactDOM,
} from "./deps.js";


  return html` <div className="ddas">
    <h1>hello world!</h1>
    ${data.map(
      (conn) => html`
        <${DdasConnection}
          ddas_name=${conn.source_id}
          host=${conn.endpoint_ip}
          port=${conn.endpoint_port}
          props=${{
            keepalives: conn.keepalives,
            receivers: conn.receivers,
          }}
        />
      `
    )}
  </div>`;
};
render(h(App), document.body);

```
