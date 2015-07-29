# `applyMiddleware(...middlewares)`

Middleware is the suggested way to extend Redux with custom functionality. Middleware lets you wrap the store’s [`dispatch`](Store.md#dispatch) method for fun and profit. The key feature of middleware is that it is composable. Multiple middleware can be combined together, where each middleware requires no knowledge of the what comes before or after it in the chain.

The most common use case for the middleware is to support asynchronous actions without much boilerplate code or a dependency on a library like [Rx](https://github.com/Reactive-Extensions/RxJS). For example, [redux-thunk](https://github.com/gaearon/redux-thunk) lets the action creators invert control by dispatching functions that receive [`dispatch`](Store.md#dispatch) as an argument and may call it asynchronously. Another example of middleware is [redux-promise](https://github.com/acdlite/redux-promise) that lets you dispatch a [Promise](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Global_Objects/Promise), and dispatches a raw action itself when the Promise resolves.

Middleware is not baked into [`createStore`](createStore.md) and is not a fundamental part of the Redux architecture, but we consider it useful enough to be supported right in the core. This way, there is a single standard way to extend [`dispatch`](Store.md#dispatch) in the ecosystem, and different middleware may compete in expressiveness and utility.

#### Arguments

* `...middlewares` (*arguments*): Functions that conform to the Redux *middleware API*. Each middleware receives [`Store`](Store.md)’s [`dispatch`](Store.md#dispatch) and [`getState`](Store.md#getState) functions as named arguments, and returns a function. That function will be given the `next` middleware’s dispatch method, and is expected to return a function of `action` calling `next(action)` with a potentially different argument, or at a different time, or maybe not calling it at all. The last middleware in chain will receive the real store’s [`dispatch`](Store.md#dispatch) method as the `next` parameter, thus closing the chain. So, the middleware signature is `({ getState, dispatch }) => next => action`.

#### Returns

(*Function*) A store enhancer that applies the given middleware. The store enhancer is a function that needs to be applied to `createStore`. It will return a different `createStore` which has the middleware enabled.

#### Example: Using Thunk Middleware for Async Actions

```js
import { createStore, applyMiddleware } from 'redux';
import thunk from 'redux-thunk';
import sandwichShop from './reducers';

function makeASandwich(secretSauce, forPerson) {
  return {
    type: 'MAKE_SANDWHICH',
    secretSauce,
    forPerson
  };
}

function apologize(error, toPerson) {
  return {
    type: 'APOLOGIZE',
    toPerson,
    error
  };
}

function fetchSecretSauce() {
  return fetch('https://www.google.com/search?q=secret+sauce');
}

// Thunk middleware lets us dispatch functions in addition to plain objects.
// Curiously, they will take `dispatch` as an argument.
// This gives action creators full control over when and what to dispatch.

function makeASandwichWithSecretSauce(forPerson) {
  return function (dispatch) {
    return fetchSecretSauce().then(
      secretSauce => dispatch(makeASandwich(secretSauce, forPerson)),
      error => dispatch(apologize(error, forPerson))
    );
  };
}

// Note that makeASandwichWithSecretSauce returns a Promise.
// Thunk middleware doesn’t have any special Promise support.
// However `dispatch` inside it returns the value from the thunk.
// This can be used to compose thunks into a larger flow.

function makeSandwhichesForEverybody() {
  return function (dispatch) {
    return dispatch(makeASandwichWithSecretSauce('My Grandma')).then(() =>
      Promise.all([
        dispatch(makeASandwichWithSecretSauce('Me')),
        dispatch(makeASandwichWithSecretSauce('My wife'))
      ]).then(() =>
        dispatch(makeASandwich('Our kids'))
      )
    )
  };
}

// applyMiddleware supercharges createStore with middleware:
let createStoreWithMiddleware = applyMiddleware(thunk)(createStore);

// We can use it exactly like “vanilla” createStore.
let store = createStoreWithMiddleware(sandwichShop);

// Very useful for server rendering!
store.dispatch(makeSandwhichesForEverybody()).then(() =>
  response.send(React.renderToString(<MyApp store={store} />))
);
```

#### Example: Custom Logger Middleware

```js
import { createStore, applyMiddleware } from 'redux';
import todos from './reducers';

function logger({ getState }) {
  return (next) => (action) => {
    console.log('will dispatch', action);

    // Call the next dispatch method in the middleware chain.
    let returnValue = next(action);

    console.log('state after dispatch', getState());

    // This will likely be the action itself, unless
    // a middleware further in chain changed it.
    return returnValue;
  };
}

let createStoreWithMiddleware = applyMiddleware(logger)(createStore);
let store = createStoreWithMiddleware(todos, ['Use Redux']);

store.dispatch({
  type: 'ADD_TODO',
  text: 'Understand the middleware'
});
// (These lines will be logged by the middleware:)
// will dispatch: { type: 'ADD_TODO', text: 'Understand the middleware' }
// state after dispatch: ['Use Redux', 'Understand the middleware']
```


#### Tips

* Middleware only wraps the store’s [`dispatch`](Store.md#dispatch) function. Technically, anything a middleware can do, you can do manually by wrapping every `dispatch` call, but it’s easier to manage this in a single place and define action transformations on the scale of the whole project.

* If you use other store enhancers in addition to `applyMiddleware`, make sure to put `applyMiddleware` before them in the composition chain because the middleware is potentially asynchronous. For example, it should go before [redux-devtools](https://github.com/gaearon/redux-devtools) because otherwise the DevTools won’t see the raw actions emitted by the Promise middleware and such.

* Ever wondered what `applyMiddleware` itself is? It ought to be an extension mechanism more powerful than the middleware itself. Indeed, `applyMiddleware` is an example of the most poweful Redux extension mechanism called *store enhancers*. It is highly unlikely you’ll ever want to write a store enhancer yourself. Another example of a store enhancer is [redux-devtools](https://github.com/gaearon/redux-devtools). Middleware is less powerful than a store enhancer, but it is easier to write.

* Middleware sounds much more complicated than it really is. The only way to really understand the middleware is to see how the existing middleware works, and try to write your own. The function nesting can be intimidating, but most of the middleware you’ll find are in fact 10-liners, and the nesting and composability is what makes the middleware system powerful.