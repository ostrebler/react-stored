# /!\ Important : This doc is relevant for up to version 1.4.6. New doc coming soon.

## react-stored

The ultimate `useStore` implementation. The power and simplicity of `useState`, but with persistence, global key-based synchronization without context, speed and reference optimization, safety checks, and other cool stuff.

Ever dreamed of such of feature but couldn't come up with a 100% satisfying solution ? Well, this package is for you.

### It is designed to :

1. provide you with a **reliable key-based** `useState`-like feature called `useStore`,
2. with **auto-sync** with other `useStore` calls sharing the same key,
3. **without** context,
4. with **persistence** across page reloads and browser sessions ([or not](#faq)),
5. with **configurable safety asserts** on deserialization and **default fallbacks**,
6. with **no unnecessary** rerender, ever,
7. with **SSR** capacities (see [FAQ](#what-if-you-are-rendering-server-side-)),
8. with **simplicity** and **cool local / global configuration** options,
9. with **zero dependency** (other than [React](https://reactjs.org/) of course),
10. with **very little** extra bundle size (+ 2.6 KB (4 times less than this very readme))

[Go to FAQ](#faq)

### Install

```console
yarn add react-stored
```

# Quick demo

Say you have the two following components far away from one another in the tree :

```javascript
import React from "react";
import { useStore } from "react-stored";

function FirstComponent() {
  const [counter, setCounter] = useStore("counter", 0);
  return (
    <>
      <h1>Counter : {counter}</h1>
      <button onClick={() => setCounter(counter + 1)}>Increment</button>
    </>
  );
}
```

```javascript
import React from "react";
import { useStore } from "react-stored";

function SecondComponent() {
  const [counter, setCounter] = useStore("counter", 0);
  return (
    <>
      <h1>Counter : {counter}</h1>
      <button onClick={() => setCounter(0)}>Reset</button>
    </>
  );
}
```

Since they share the same key (`'counter'`), they actually seemlessly share the _same value_ and keep one another in sync _no matter_ how hard you try to unsync them. **Even better** : if you refresh the page, nothing changes. The values are persistent. You better like them, because they ain't going anywhere unless you change the key or... the key _prefix_ (see [config](#3--the-config-function)).

The second argument to `useStore`, the number `0` in this case, represents the default counter value, as no persistent save can be found the first time around.

# Reference

## 1- The `useStore` hook

This is the cornerstone of this package. It 'connects' you to a specific store, identified by key, and returns the value at that location as well as an update function. Its overall feel mimics `useState`. It also listens to any outside change, and rerenders accordingly to keep all parts of your UI in sync.

It can take up to 3 arguments (**only the key is required**) :

```javascript
const [value, setValue] = useStore(key, defaultValue, assertFunction);
```

- `key` : Any string.
- `defaultValue` (optional) : The value affected by default to the store and returned by `useStore` when no previous save is found. This could be any JSON value (and [even more](#what-about-storing-non-json-values-like-dates-maps-and-simple-functions-)).
- `assertFunction` (optional) : On initial render, [or when any of `useStore`'s parameters changes](#identity-and-hook-optimization), the previous save passes through this function and has to return `true`. If it returns `false` or throws an error, `defaultValue` will be used and overwrite the save. This can be very handy, for example to prevent the hydration of `useStore` with ill-formed or outdated JSON. I would usually use [ajv](https://www.npmjs.com/package/ajv) in places like these.

### Identity and hook optimization

Just like most hooks, `useStore` relies on **object identity** to optimize internal recomputations. If your `defaultValue` is **an object or an array**, please use [`useRef`](https://reactjs.org/docs/hooks-reference.html#useref) or [`useMemo`](https://reactjs.org/docs/hooks-reference.html#usememo) to keep the same reference as long as possible :

```javascript
const init = useMemo(() => ({ x: props.x, y: 0 }), [props.x]);
const [coord, setCoord] = useStore("coord", init);
```

Similarly, use [`useCallback`](https://reactjs.org/docs/hooks-reference.html#usecallback) for the assert function :

```javascript
const assert = useCallback(state => ajv.validate(props.model, state), [
  props.model
]);
const [state, setState] = useStore("my-state", null, assert);
```

**Better (!)** : Whenever possible, set your `defaultValue` and `assertFunction` _outside_ the render tree using [`addSchema`](#2--the-addschema-function). This way, you don't have to worry about reference optimization. Plus, the separation between configuration and usage makes your code cleaner.

### The update function

Like `useState`, the update function can take a value, or a _function_ taking the old value as an argument and returning the new one. If you update the store to the same value as the current one (using `===` for the comparison), no update is actually triggered, thus preventing useless rerenders.

```javascript
const [counter, setCounter] = useStore('counter')
<button onClick={() => setCounter(counter + 1)}>
  Increment
</button>

// Equivalent to :

const setCounter = useStore('counter')[1]
<button onClick={() => setCounter(counter => counter + 1)}>
  Increment
</button>
```

The **identity** of this update function is preserved as long as the `key` stays the same.

## 2- The `addSchema` function

This configuration function allows you to set _default_- default values and _default_ assert functions to certain keys or key patterns outside of your React tree, typically in `index.js` before your `ReactDOM.render`. **If you don't rely on props to set default values and assert functions, you shouldn't set them at component-level and `addSchema` should be your primary configuration choice**.

It takes the same arguments as `useStore` except the key can be a regexp. If it is, then all keys matching the regexp will use the given configuration.

```javascript
import React from "react";
import ReactDOM from "react-dom";
import { addSchema } from "react-stored";
import Ajv from "ajv";
import App from "./App";

addSchema("counter", 0, counter => counter < 100);
// Any invocation of 'counter' will now use 0 as its default value, and ensure
// that any retrieved save is smaller than 100. If not, 0 will be used instead.

addSchema(/coord-v\d+/, { x: 0, y: 4 });
// Any invocation of 'coord-v1', 'coord-v43', 'coord-v9987', etc. will use the
// given object as its default value.

const isValidArray = new Ajv().compile({
  type: "array",
  items: { type: "string" }
});

addSchema(/array-[0-9A-F]{2}/, [], isValidArray);
// Any invocation from 'array-00' to 'array-FF' will be initialized with an
// empty array. Any previous save should be an array of strings, otherwise
// it will be overwritten with an empty array.

ReactDOM.render(<App />, document.getElementById("root"));
```

## 3- The `config` function

With this function, you can tweak some general stuff. It has to be called outside of the React structure, before any `useStore` call, so usually somewhere in your `index.js` before your `ReactDOM.render`.

```javascript
import React from "react";
import ReactDOM from "react-dom";
import { config } from "react-stored";
import App from "./App";

// Below are the DEFAULT settings, it is pointless
// to set them explicitly to these values :

config({
  // IMPORTANT : A seemless prefix to ALL your keys, this has to be specific
  // to your app :
  keyPrefix: "",

  // The storage to be used (could be replaced with an in-memory alternative,
  // sessionStorage, a cookie-based storage, etc. See FAQ) :
  storage: window.localStorage,

  // When true, real-time auto-sync is extended across all browser tabs
  // sharing the same origin :
  crossTab: false,

  // A function that should transform JSON into a string :
  serialize: JSON.stringify,

  // A function that should transform a string into JSON :
  deserialize: JSON.parse
});

ReactDOM.render(<App />, document.getElementById("root"));
```

### `keyPrefix` is important

Unless you have some very specific use cases, the `keyPrefix` is really the only important part to configure. You set it _once_, and everything stored or retrieved from the storage will use that prefix in addition to the keys used in your components. All this happens of course _seemlessly_, you don't have to think about it.

Yes, `localStorage` is compartmentalized by domain but you could have several apps by domain. It's just a good habit to set a `keyPrefix` that is specific to your app. It defines a namespace.

Also, imagine several customers already tested your app and have _their local copy_ of the store. Now say you wanna change the JSON structure because of some new requirements. You could just set `keyPrefix` to the **current version of the app**, thus preventing any hydration of outdated JSON saves.

Here is typically what my `index.js` looks like :

```javascript
import React from "react";
import ReactDOM from "react-dom";
import { config } from "react-stored";
import App from "./App";

config({
  keyPrefix: "my-app-v2.4.1-"
});

ReactDOM.render(<App />, document.getElementById("root"));
```

### Schema definition with `config`

You can also replace all your [`addSchema`](#2--the-addschema-function) calls with a single array using the `config` function :

```javascript
config({
  keyPrefix: "my-app-v4",
  schemas: [
    {
      key: "counter",
      init: 0,
      assert: counter => counter < 100
    },
    {
      key: /coord-v\d+/,
      init: { x: 1, y: 0 }
    }
  ]
});
```

## 4- The `readStore` function

This gives you the possibility to passively read the content of your store outside of any component. This was useful for me when I needed to pass a stored token to some server requests using [Apollo](https://www.apollographql.com/docs/react/) links. It takes only one argument, the key, and returns the corresponding JSON.

# FAQ

## What if no default value or assert function is set ?

Here is what's going on everytime you invoke `useStore` on a specific key :

- If a _previous save_ is found in the storage for that key, it is used. If not :
- If a default value is set _locally_ (as an argument of `useStore`), it is used. If not :
- If a default value is set _globally_ (as an argument of `addSchema`), it is used. If not :
- `null` will be used.

Things are easier with the assert function : if none could be found for a specific key (not locally nor globally), the functionality is simply discarded and hydration of previous saves goes without any checks.

## What if you _don't_ want your stores to be persistent ?

Since you have **absolute control** over the actual storage being used in the background (see [`config`](#3--the-config-function)), you can use your own custom in-memory version (it just has to implement [`getItem`](https://developer.mozilla.org/en-US/docs/Web/API/Storage/getItem) and [`setItem`](https://developer.mozilla.org/en-US/docs/Web/API/Storage/setItem)). There are _plenty_ of npm packages with such alternatives. I personally like [memorystorage](https://www.npmjs.com/package/memorystorage) :

```javascript
import { config } from "react-stored";
import MemoryStorage from "memorystorage";

config({
  keyPrefix: "whatever",
  storage: new MemoryStorage()
});
```

## What if you are rendering server-side ?

Dead simple. Again, you have to tweak your [`config`](#3--the-config-function) to use a universal version of `localStorage` (i.e. with in-memory fallback on server-side).

```javascript
import { config } from "react-stored";
import MemoryStorage from "memorystorage";

config({
  keyPrefix: "whatever",
  storage:
    typeof window === "undefined" ? new MemoryStorage() : window.localStorage
});
```

## What about storing non-JSON values like dates, maps and simple functions ?

This is possible with a careful use of [serialize-javascript](https://www.npmjs.com/package/serialize-javascript), [`eval`](https://javascriptweblog.wordpress.com/2010/04/19/how-evil-is-eval/) and a bit of configuration :

```javascript
import { config } from "react-stored";
import serialize from "serialize-javascript";

config({
  serialize,
  deserialize: str => eval(`(${str})`)
});
```

# About

I look forward to any suggestion, question or bug report. Please use [GitHub's issue tracker](https://github.com/ostrebler/react-stored/issues).
