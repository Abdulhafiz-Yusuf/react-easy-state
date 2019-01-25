# React Easy State

Simple React state management. Made with :heart: and ES6 Proxies.

[![Build](https://img.shields.io/circleci/project/github/solkimicreb/react-easy-state/master.svg)](https://circleci.com/gh/solkimicreb/react-easy-state/tree/master) [![Coverage Status](https://coveralls.io/repos/github/solkimicreb/react-easy-state/badge.svg)](https://coveralls.io/github/solkimicreb/react-easy-state) [![JavaScript Style Guide](https://img.shields.io/badge/code_style-standard-brightgreen.svg)](https://standardjs.com) [![Package size](https://img.shields.io/bundlephobia/minzip/react-easy-state.svg)](https://bundlephobia.com/result?p=react-easy-state) [![Version](https://img.shields.io/npm/v/react-easy-state.svg)](https://www.npmjs.com/package/react-easy-state) [![dependencies Status](https://david-dm.org/solkimicreb/react-easy-state/status.svg)](https://david-dm.org/solkimicreb/react-easy-state) [![License](https://img.shields.io/npm/l/react-easy-state.svg)](https://www.npmjs.com/package/react-easy-state)

<a href="#platform-support"><img src="images/browser_support.png" alt="Browser support" width="450px" /></a>

<details>
<summary><strong>Table of Contents</strong></summary>
<!-- Do not edit the Table of Contents, instead regenerate with `npm run build-toc` -->

<!-- toc -->

* [Introduction](#introduction)
* [Installation](#installation)
* [Usage](#usage)
  * [Creating global stores](#creating-global-stores)
  * [Creating reactive views](#creating-reactive-views)
  * [Creating local stores](#creating-local-stores)
* [Examples with live demos](#examples-with-live-demos)
* [Articles](#articles)
* [FAQ and Gotchas](#faq-and-gotchas)
  * [Broken `this` in store methods](#broken-this-in-store-methods)
  * [Views not rendering](#views-not-rendering)
  * [Views rendering multiple times unnecessarily](#views-rendering-multiple-times-unnecessarily)
  * [PureComponent and memo](#purecomponent-and-memo)
  * [Deriving local stores from props (getDerivedStateFromProps)](#deriving-local-stores-from-props-getderivedstatefromprops)
  * [Naming local stores as state](#naming-local-stores-as-state)
  * [Usage with React Router](#usage-with-react-router)
  * [Usage with third party components](#usage-with-third-party-components)
* [Platform support](#platform-support)
* [Performance](#performance)
* [How does it work?](#how-does-it-work)
* [Alternative builds](#alternative-builds)
* [Contributing](#contributing)

<!-- tocstop -->

</details>

## Introduction

Easy State is a transparent reactive state management library with two functions and two accompanying rules.

1.  Always wrap your components with `view()`.
2.  Always wrap your state store objects with `store()`.

```jsx
import React from 'react'
import { store, view } from 'react-easy-state'

const counter = store({
  num: 0,
  incr: () => counter.num++
})

export default view(() => <button onClick={counter.incr}>{counter.num}</button>)
```

This is enough for it to automatically update your views when needed. It doesn't matter how you structure or mutate your state stores, any syntactically valid code works.

Check this [TodoMVC codesandbox](https://codesandbox.io/s/github/solkimicreb/react-easy-state/tree/master/examples/todo-mvc?module=%2Fsrc%2FtodosStore.js) or [raw code](/examples/todo-mvc/src/todosStore.js) for a more exciting example with nested data, arrays and computed values.

## Installation

`npm install react-easy-state`

<details>
<summary>Setting up a quick project</summary>

Easy State supports [Create React App](https://github.com/facebookincubator/create-react-app) without additional configuration. Just run the following commands to get started.

```sh
npx create-react-app my-app
cd my-app
npm install react-easy-state
npm start
```

_You need npm 5.2+ to use npx._

</details>

## Usage

### Creating global stores

`store` creates a state store from the passed object and returns it. A state store behaves just like the passed object. (To be precise, it is a transparent reactive proxy of the original object.)

```js
import { store } from 'react-easy-state'

const user = store({
  name: 'Rick'
})

// stores behave like normal JS objects
user.name = 'Bob'
```

<details>
<summary>State stores may have arbitrary structure and they may be mutated in any syntactically valid way.</summary>

```js
import { store } from 'react-easy-state'

// stores can include any valid JS structure (nested data, arrays, Maps, Sets, getters, setters, inheritance, ...)
const user = store({
  profile: {
    firstName: 'Bob',
    lastName: 'Smith',
    get name () {
      return `${user.firstName} ${user.lastName}`
    }  
  }
  hobbies: ['programming', 'sports'],
  friends: new Map()
})

// stores can be mutated in any syntactically valid way
user.profile.firstName = 'Bob'
delete user.profile.lastName
user.hobbies.push('reading')
user.friends.set('id', otherUser)
```

</details>

<details>
<summary>Async operations can be expressed with the standard async/await syntax.</summary>

```js
import { store } from 'react-easy-state'

const userStore = store({
  user: {},
  async fetchUser() {
    userStore.user = await fetch('/user')
  }
})

export default userStore
```

</details>

<details>
<summary>State stores may import and use other state stores in their methods.</summary>

Splitting large stores into multiple files is totally okay.

_userStore.js_

```js
import { store } from 'react-easy-state'

const userStore = store({
  user: {},
  async fetchUser() {
    userStore.user = await fetch('/user')
  }
})

export default userStore
```

_recipesStore.js_

```js
import { store } from 'react-easy-state'
import userStore from './userStore'

const recipesStore = store({
  recipes: [],
  async fetchRecipes() {
    recipesStore.recipes = await fetch(`/recipes?user=${userStore.user.id}`)
  }
})

export default recipesStore
```

</details>

<details>
<summary>Avoid using the <code>this</code> keyword in the methods of your state stores.</summary>

```jsx
const counter = store({
  num: 0,
  increment() {
    // DON'T DO THIS
    this.num++
  }
})

export default view(() => <div onClick={counter.increment}>{counter.num}</div>)
```

The above snippet won't work, because `increment` is passed as a callback and loses its `this`. You should use the direct object reference - `counter` - instead of `this`.

```js
const counter = store({
  num: 0,
  increment() {
    // DO THIS
    counter.num++
  }
})

export default counter
```

This works as expected, even when you pass store methods as callbacks.

</details>

<details>
<summary>Wrap your state stores with `store` as early as possible.</summary>

```js
// DON'T DO THIS
const person = { name: 'Bob' }
person.name = 'Ann'

export default store(person)
```

The above example wouldn't trigger re-renders on the `person.name = 'Ann'` mutation, because it is targeted at the raw object. Mutating the raw - none `store` wrapped object - won't schedule renders.

Do this instead of the above code.

```js
// DO THIS
const person = store({ name: 'Bob' })
person.name = 'Ann'

export default person
```

</details>

### Creating reactive views

Wrapping your components with `view` turns them into reactive views. A reactive view re-renders whenever a piece of store - used inside its render - changes.

```jsx
import React from 'react'
import { view, store } from 'react-easy-state'

// this is a global state store
const user = store({ name: 'Bob' })

// this is re-rendered whenever user.name changes
export default view(() => (
  <div>
    <input value={user.name} onChange={ev => (user.name = ev.target.value)} />
    <div>Hello {user.name}!</div>
  </div>
))
```

**Wrap ALL of your components with `view` - including class and function ones - even if they don't seem to directly use a store.**

<details>
<summary>A single reactive component may use multiple stores inside its render.</summary>

```jsx
import React from 'react'
import { view, store } from 'react-easy-state'

const user = store({ name: 'Bob' })
const timeline = store({ posts: ['react-easy-state'] })

// this is re-rendered whenever user.name or timeline.posts[0] changes
export default view(() => (
  <div>
    <div>Hello {user.name}!</div>
    <div>Your first post is: {timeline.posts[0]}</div>
  </div>
))
```

</details>

<details>
<summary><code>view</code> implements an optimal <code>shouldComponentUpdate</code> for your components.</summary>

The `view` wrapper optimizes the passed component with an optimal `shouldComponentUpdate` or `memo`, which shallow compares the current state and props with the next ones.

* Using `PureComponent` or `memo` will provide no additional performance benefits.

* Defining a custom `shouldComponentUpdate` may rarely provide performance benefits when you apply some use case specific heuristics inside it.

</details>

<details>
<summary>Always apply view as the latest (innermost) wrapper when you combine it with other Higher Order Components.</summary>

```jsx
import { view } from 'react-easy-state'
import { withRouter } from 'react-router-dom'
import { withTheme } from 'styled-components'

const Comp = () => <div>A reactive component</div>

// DO THIS
withRouter(view(Comp))
withTheme(view(Comp))

// DON'T DO THIS
view(withRouter(Comp))
view(withTheme(Comp))
```

</details>

<details>
<summary>Usage with React Router (pre 4.4 version)</summary>

When you use React Router together with `view` you have to do the same trick that applies to Redux's `connect` and MobX's `observer`.

* If routing is not updated properly, wrap your `view(Comp)` - with the `Route`s inside - in `withRouter(view(Comp))`. This lets react-router know when to update.

* The order of the HOCs matter, always use `withRouter(view(Comp))`.

This is not necessary if you use React Router 4.4+. You can find more details and some reasoning about this in [this react-router docs page](https://github.com/ReactTraining/react-router/blob/master/packages/react-router/docs/guides/blocked-updates.md).

</details>

<details>
<summary>Passing nested data to third party components.</summary>

Third party helpers - like data grids - may consist of many internal components which can not be wrapped by `view`, but sometimes you would like them to re-render when the passed data mutates. Traditional React components re-render when their props change by reference, so mutating the passed reactive data won't work in these cases. You can solve this issue by deep cloning the observable data before passing it to the component. This creates a new reference for the consuming component on every store mutation.

```jsx
import React from 'react'
import { view, store } from 'react-easy-state'
import Table from 'rc-table'
import cloneDeep from 'lodash/cloneDeep'

const dataStore = store({
  items: [
    {
      product: 'Car',
      value: 12
    }
  ]
})

export default view(() => <Table data={cloneDeep(dataStore.items)} />)
```

</details>

### Creating local stores

A singleton global store is perfect for something like the current user, but sometimes having local component states is a better fit. Just create a store inside a function component or as a class component property in these cases.

#### Local stores in function components

```jsx
import React from 'react'
import { view, store } from 'react-easy-state'

export default view(() => {
  const counter = store({ num: 0 })
  return <button={() => counter.num++}>{counter.num}</div>
})
```

You can use any React hook - including `useState` - in function components, Easy State won't interfere with them.

</details>

#### Local stores in class components

```jsx
import React, { Component } from 'react'
import { view, store } from 'react-easy-state'

class ClockComp extends Component {
  counter = store({ num: 0 })
  increment = () => counter.num++

  render() {
    return <button onClick={this.increment}>{this.counter.num}</button>
  }
}

export default view(ClockComp)
```

You can also use vanilla `setState` in your class components, Easy State won't interfere with it.

<details>
<summary>Don't name local stores as <code>state</code>.</summary>

Naming your local state stores as `state` may conflict with React linter rules, which guard against direct state mutations. Please use a more descriptive name instead.

```jsx
import React, { Component } from 'react'
import { view, store } from 'react-easy-state'

class Profile extends Component {
  // DON'T DO THIS
  state = store({})
  // DO THIS
  user = store({})
  render() {}
}
```

</details>

<details>
<summary>Deriving local stores from props (getDerivedStateFromProps)</summary>

Class components wrapped with `view` have an extra static `deriveStoresFromProps` lifecycle method, which works similarly to the vanilla `getDerivedStateFromProps`.

```jsx
import React, { Component } from 'react'
import { view, store } from 'react-easy-state'

class NameCard extends Component {
  userStore = store({ name: 'Bob' })

  static deriveStoresFromProps(props, userStore) {
    userStore.name = props.name || userStore.name
  }

  render() {
    return <div>{this.userStore.name}</div>
  }
}

export default view(NameCard)
```

Instead of returning an object, you should directly mutate the received stores. If you have multiple local stores on a single component, they are all passed as arguments - in their definition order - after the first props argument.

</details>

---

That's it, You know everything to master React state management! Check some of the [examples](#examples-with-live-demos) and [articles](#articles) for more inspiration or the [FAQ section](#faq-and-gotchas) for common issues.

## Examples with live demos

_Beginner_

* [Clock Widget](https://solkimicreb.github.io/react-easy-state/examples/clock/build) ([source](/examples/clock/)) ([codesandbox](https://codesandbox.io/s/github/solkimicreb/react-easy-state/tree/master/examples/clock)): a reusable clock widget with a tiny local state store.
* [Stopwatch](https://solkimicreb.github.io/react-easy-state/examples/stop-watch/build) ([source](/examples/stop-watch/)) ([codesandbox](https://codesandbox.io/s/github/solkimicreb/react-easy-state/tree/master/examples/stop-watch)) ([tutorial](https://hackernoon.com/introducing-react-easy-state-1210a156fa16)): a stopwatch with a mix of normal and computed state properties.

_Advanced_

* [TodoMVC](https://solkimicreb.github.io/react-easy-state/examples/todo-mvc/build) ([source](/examples/todo-mvc/)) ([codesandbox](https://codesandbox.io/s/github/solkimicreb/react-easy-state/tree/master/examples/todo-mvc)): a classic TodoMVC implementation with a lot of computed data and implicit reactivity.
* [Contacts Table](https://solkimicreb.github.io/react-easy-state/examples/contacts/build) ([source](/examples/contacts/)) ([codesandbox](https://codesandbox.io/s/github/solkimicreb/react-easy-state/tree/master/examples/contacts)): a data grid implementation with a mix of global and local state.
* [Beer Finder](https://solkimicreb.github.io/react-easy-state/examples/beer-finder/build) ([source](/examples/beer-finder/)) ([codesandbox](https://codesandbox.io/s/github/solkimicreb/react-easy-state/tree/master/examples/beer-finder)) ([tutorial](https://medium.com/@solkimicreb/design-patterns-with-react-easy-state-830b927acc7c)): an app with async actions and a mix of local and global state, which finds matching beers for your meal.

## Articles

* [Introducing React Easy State](https://blog.risingstack.com/introducing-react-easy-state/): making a simple stopwatch.
* [Stress Testing React Easy State](https://medium.com/@solkimicreb/stress-testing-react-easy-state-ac321fa3becf): demonstrating Easy State's reactivity with increasingly exotic state mutations.
* [Design Patterns with React Easy State](https://medium.com/@solkimicreb/design-patterns-with-react-easy-state-830b927acc7c): demonstrating async actions and local and global state management through a beer finder app.
* [The Ideas Behind React Easy State](https://medium.com/dailyjs/the-ideas-behind-react-easy-state-901d70e4d03e): a deep dive under the hood of Easy State.

## FAQ and Gotchas

### Views rendering multiple times unnecessarily

Re-renders are batched 99% percent of the time until the end of the state changing function. You can mutate your state stores multiple times in event handlers, async functions and timer and networking callbacks without worrying about multiple renders and performance.

If you mutate your stores multiple times synchronously from exotic task sources, multiple renders may happen though. In these rare occasions you can batch changes manually with the `batch` function. `batch(fn)` executes the passed function immediately and batches any subsequent re-renders until the function execution finishes.

```jsx
import React from 'react'
import { view, store, batch } from 'react-easy-state'

const user = store({ name: 'Bob', age: 30 })

// this makes sure the state changes will cause maximum one re-render,
// no matter where this function is getting invoked from
function mutateUser() {
  batch(() => {
    user.name = 'Ann'
    user.age = 32
  })
}

export default view(() => (
  <div>
    name: {user.name}, age: {user.age}
  </div>
))
```

**NOTE:** The React team plans to improve render batching in the future. The `batch` function and built-in batching may be deprecated and removed in the future in favor of React's own batching.

## Platform support

* Node: 6 and above
* Chrome: 49 and above
* Firefox: 38 and above
* Safari: 10 and above
* Edge: 12 and above
* Opera: 36 and above
* React Native: iOS 10 and above and Android with [community JSC](https://github.com/SoftwareMansion/jsc-android-buildscripts)
* IE is not supported and never will be

_This library is based on non polyfillable ES6 Proxies. Because of this, it will never support IE._

_React Native is supported on iOS and Android is supported with the community JavaScriptCore. Learn how to set it up [here](https://github.com/SoftwareMansion/jsc-android-buildscripts#how-to-use-it-with-my-react-native-app). It is pretty simple._

## Performance

You can compare Easy State with plain React and other state management libraries with the below benchmarks. It performs a bit better than MobX and a bit worse than Redux.

* [js-framework-benchmark](https://github.com/krausest/js-framework-benchmark) ([source](https://github.com/krausest/js-framework-benchmark/tree/master/react-v16.1.0-easy-state-v4.0.1-keyed)) ([results](https://rawgit.com/krausest/js-framework-benchmark/master/webdriver-ts-results/table.html))

## How does it work?

Under the hood Easy State uses the [@nx-js/observer-util](https://github.com/nx-js/observer-util) library, which relies on [ES6 Proxies](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy) to observe state changes. [This blog post](https://blog.risingstack.com/writing-a-javascript-framework-data-binding-es6-proxy/) gives a little sneak peek under the hood of the `observer-util`.

## Alternative builds

This library detects if you use ES6 or commonJS modules and serve the right format to you. The default bundles use ES6 features, which may not yet be supported by some minifier tools. If you experience issues during the build process, you can switch to one of the ES5 builds from below.

* `react-easy-state/dist/es.es6.js` exposes an ES6 build with ES6 modules.
* `react-easy-state/dist/es.es5.js` exposes an ES5 build with ES6 modules.
* `react-easy-state/dist/cjs.es6.js` exposes an ES6 build with commonJS modules.
* `react-easy-state/dist/cjs.es5.js` exposes an ES5 build with commonJS modules.

If you use a bundler, set up an alias for `react-easy-state` to point to your desired build. You can learn how to do it with webpack [here](https://webpack.js.org/configuration/resolve/#resolve-alias) and with rollup [here](https://github.com/rollup/rollup-plugin-alias#usage).

## Contributing

Contributions are always welcome. Just send a PR against the master branch or open a new issue. Please make sure that the tests and the linter pass and the coverage remains decent. Thanks!
