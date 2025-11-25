# React-Redux: A Deep Systems Exploration

Let me take you through the entire architecture of React-Redux, from the conceptual foundations to the internal mechanics that make it work.

---

## Part 1: Foundational Concepts — Why Redux Exists

### The State Management Problem

In a React application without Redux, state lives in components. When you have:

```jsx
function ParentComponent() {
  const [user, setUser] = useState(null);
  return (
    <div>
      <Header user={user} />
      <Sidebar user={user} />
      <MainContent user={user} setUser={setUser} />
      <Footer user={user} />
    </div>
  );
}
```

You face "prop drilling" — passing data through intermediate components that don't need it. This becomes unmaintainable as your application grows.

**React's Context API** solves this partially, but it has fundamental limitations:
- Context re-renders all consumers when any part of the context value changes
- No built-in middleware system for async logic
- No time-travel debugging or action history
- Difficult to implement complex update patterns

Redux provides a **predictable state container** with a single source of truth, making state mutations explicit and traceable.

---

## Part 2: Core Redux Architecture (Pre-React Integration)

Before we discuss React-Redux, let's understand pure Redux.

### The Three Principles

1. **Single source of truth**: The entire application state is stored in a single object tree
2. **State is read-only**: The only way to change state is to emit an action
3. **Changes are made with pure functions**: Reducers are pure functions that take previous state and an action, and return new state

### Core Components

#### 1. Store (`createStore` internals)

The store is the heart of Redux. Let's look at what happens when you create one:

```javascript
// Simplified version of Redux's createStore
function createStore(reducer, preloadedState, enhancer) {
  let currentReducer = reducer;
  let currentState = preloadedState;
  let currentListeners = [];
  let nextListeners = currentListeners;
  let isDispatching = false;

  function getState() {
    if (isDispatching) {
      throw new Error('You may not call store.getState() while the reducer is executing.');
    }
    return currentState;
  }

  function subscribe(listener) {
    if (isDispatching) {
      throw new Error('You may not call store.subscribe() while the reducer is executing.');
    }

    let isSubscribed = true;
    ensureCanMutateNextListeners();
    nextListeners.push(listener);

    // Return unsubscribe function
    return function unsubscribe() {
      if (!isSubscribed) return;
      if (isDispatching) {
        throw new Error('You may not unsubscribe from a store listener while the reducer is executing.');
      }
      isSubscribed = false;
      ensureCanMutateNextListeners();
      const index = nextListeners.indexOf(listener);
      nextListeners.splice(index, 1);
      currentListeners = null;
    };
  }

  function dispatch(action) {
    if (!isPlainObject(action)) {
      throw new Error('Actions must be plain objects.');
    }
    if (typeof action.type === 'undefined') {
      throw new Error('Actions may not have an undefined "type" property.');
    }
    if (isDispatching) {
      throw new Error('Reducers may not dispatch actions.');
    }

    try {
      isDispatching = true;
      currentState = currentReducer(currentState, action);
    } finally {
      isDispatching = false;
    }

    // Notify all subscribers
    const listeners = (currentListeners = nextListeners);
    for (let i = 0; i < listeners.length; i++) {
      const listener = listeners[i];
      listener();
    }

    return action;
  }

  function ensureCanMutateNextListeners() {
    if (nextListeners === currentListeners) {
      nextListeners = currentListeners.slice();
    }
  }

  // Initialize store with a dummy action
  dispatch({ type: '@@redux/INIT' });

  return {
    dispatch,
    subscribe,
    getState,
    replaceReducer
  };
}
```

**Key implementation details:**

- **`isDispatching` flag**: Prevents recursive dispatch calls and ensures state consistency
- **Listener array cloning**: `ensureCanMutateNextListeners()` creates a copy before mutations to prevent issues when subscribers are added/removed during dispatch
- **Initialization dispatch**: The `@@redux/INIT` action ensures all reducers run and return their initial state

#### 2. Reducers (State Mutation Logic)

```javascript
// Typical reducer structure
function userReducer(state = initialState, action) {
  switch (action.type) {
    case 'USER_LOGIN':
      return {
        ...state,
        isAuthenticated: true,
        user: action.payload
      };
    case 'USER_LOGOUT':
      return {
        ...state,
        isAuthenticated: false,
        user: null
      };
    default:
      return state;
  }
}
```

**Why pure functions?**
- Predictable: Same inputs always produce same outputs
- Testable: No side effects to mock
- Time-travel debugging: Can replay actions
- Server-side rendering: State is serializable

#### 3. Middleware (The Dispatch Pipeline)

Middleware intercepts actions before they reach reducers:

```javascript
// Simplified applyMiddleware implementation
function applyMiddleware(...middlewares) {
  return (createStore) => (reducer, preloadedState) => {
    const store = createStore(reducer, preloadedState);
    let dispatch = store.dispatch;
    let chain = [];

    const middlewareAPI = {
      getState: store.getState,
      dispatch: (action) => dispatch(action)
    };

    // Each middleware gets access to getState and dispatch
    chain = middlewares.map(middleware => middleware(middlewareAPI));
    
    // Compose middlewares: right-to-left function composition
    dispatch = compose(...chain)(store.dispatch);

    return {
      ...store,
      dispatch
    };
  };
}

// compose implementation
function compose(...funcs) {
  if (funcs.length === 0) return (arg) => arg;
  if (funcs.length === 1) return funcs[0];
  return funcs.reduce((a, b) => (...args) => a(b(...args)));
}
```

**Example: Redux Thunk Middleware**

```javascript
// redux-thunk source code (simplified)
function createThunkMiddleware(extraArgument) {
  return ({ dispatch, getState }) => next => action => {
    if (typeof action === 'function') {
      return action(dispatch, getState, extraArgument);
    }
    return next(action);
  };
}

const thunk = createThunkMiddleware();
```

**How middleware composition works:**

If you have middlewares `[A, B, C]`:
1. Compose creates: `A(B(C(store.dispatch)))`
2. When you dispatch an action:
   - A intercepts first → can modify/block/log
   - A calls `next(action)` → passes to B
   - B intercepts → can modify/block/log
   - B calls `next(action)` → passes to C
   - C intercepts → calls actual `store.dispatch`

---

## Part 3: React-Redux Bridge — Connecting Two Systems

React-Redux is the **official binding library** that connects React's component system to Redux's state container. Let's explore how this bridge is built.

### The Provider Component

The `Provider` makes the Redux store available to all nested components via React Context:

```javascript
// Simplified Provider implementation from react-redux source
import React from 'react';
import { ReactReduxContext } from './Context';

function Provider({ store, context, children }) {
  const contextValue = useMemo(() => {
    const subscription = new Subscription(store);
    subscription.onStateChange = subscription.notifyNestedSubs;
    return {
      store,
      subscription
    };
  }, [store]);

  const previousState = useMemo(() => store.getState(), [store]);

  useEffect(() => {
    const { subscription } = contextValue;
    subscription.trySubscribe();

    if (previousState !== store.getState()) {
      subscription.notifyNestedSubs();
    }
    
    return () => {
      subscription.tryUnsubscribe();
      subscription.onStateChange = null;
    };
  }, [contextValue, previousState]);

  const Context = context || ReactReduxContext;

  return <Context.Provider value={contextValue}>{children}</Context.Provider>;
}
```

**Key mechanisms:**

1. **Subscription system**: Not directly subscribing components to the store
2. **Context value memoization**: Prevents unnecessary re-renders
3. **Nested subscription management**: Ensures parent components update before children

### The Subscription Class

This is crucial for performance — it manages a hierarchy of subscriptions:

```javascript
// Simplified Subscription class from react-redux
class Subscription {
  constructor(store, parentSub) {
    this.store = store;
    this.parentSub = parentSub;
    this.unsubscribe = null;
    this.listeners = [];
    this.handleChangeWrapper = this.handleChangeWrapper.bind(this);
  }

  addNestedSub(listener) {
    this.trySubscribe();
    return this.listeners.push(listener) - 1;
  }

  notifyNestedSubs() {
    const length = this.listeners.length;
    for (let i = 0; i < length; i++) {
      this.listeners[i]();
    }
  }

  handleChangeWrapper() {
    if (this.onStateChange) {
      this.onStateChange();
    }
  }

  trySubscribe() {
    if (!this.unsubscribe) {
      this.unsubscribe = this.parentSub
        ? this.parentSub.addNestedSub(this.handleChangeWrapper)
        : this.store.subscribe(this.handleChangeWrapper);
      
      this.listeners = [];
    }
  }

  tryUnsubscribe() {
    if (this.unsubscribe) {
      this.unsubscribe();
      this.unsubscribe = null;
      this.listeners = [];
    }
  }
}
```

**Why nested subscriptions?**

Without this, when state updates:
1. All components subscribed to store get notified simultaneously
2. Parent and child components might render out of order
3. Child might render with stale parent-provided props

With nested subscriptions:
1. Store notifies Provider's subscription
2. Provider notifies its direct children
3. Children notify their children (cascading)
4. Ensures top-down rendering order

---

## Part 4: The `useSelector` Hook — Deep Dive

`useSelector` is where the magic of React-Redux happens. It subscribes a component to store changes and optimizes re-renders.

### Full Implementation Walkthrough

```javascript
// Simplified useSelector from react-redux v8+
import { useContext, useReducer, useRef, useLayoutEffect } from 'react';
import { ReactReduxContext } from './Context';
import { useIsomorphicLayoutEffect } from './utils';

function useSelectorWithStoreAndSubscription(
  selector,
  equalityFn,
  store,
  contextSub
) {
  const [, forceRender] = useReducer(s => s + 1, 0);

  const subscription = useMemo(
    () => new Subscription(store, contextSub),
    [store, contextSub]
  );

  const latestSubscriptionCallbackError = useRef();
  const latestSelector = useRef();
  const latestStoreState = useRef();
  const latestSelectedState = useRef();

  const storeState = store.getState();
  let selectedState;

  try {
    if (
      selector !== latestSelector.current ||
      storeState !== latestStoreState.current ||
      latestSubscriptionCallbackError.current
    ) {
      const newSelectedState = selector(storeState);
      
      // If selector output hasn't changed, use previous reference
      if (
        latestSelectedState.current === undefined ||
        !equalityFn(newSelectedState, latestSelectedState.current)
      ) {
        selectedState = newSelectedState;
      } else {
        selectedState = latestSelectedState.current;
      }
    } else {
      selectedState = latestSelectedState.current;
    }
  } catch (err) {
    if (latestSubscriptionCallbackError.current) {
      err.message += `\nThe error may be correlated with this previous error:\n${
        latestSubscriptionCallbackError.current.stack
      }\n\n`;
    }
    throw err;
  }

  useIsomorphicLayoutEffect(() => {
    latestSelector.current = selector;
    latestStoreState.current = storeState;
    latestSelectedState.current = selectedState;
    latestSubscriptionCallbackError.current = undefined;
  });

  useIsomorphicLayoutEffect(() => {
    function checkForUpdates() {
      try {
        const newStoreState = store.getState();
        
        if (newStoreState === latestStoreState.current) {
          return;
        }

        const newSelectedState = latestSelector.current(newStoreState);

        if (equalityFn(newSelectedState, latestSelectedState.current)) {
          return;
        }

        latestSelectedState.current = newSelectedState;
        latestStoreState.current = newStoreState;
      } catch (err) {
        latestSubscriptionCallbackError.current = err;
      }

      forceRender();
    }

    subscription.onStateChange = checkForUpdates;
    subscription.trySubscribe();

    checkForUpdates();

    return () => subscription.tryUnsubscribe();
  }, [store, subscription]);

  return selectedState;
}

export function useSelector(selector, equalityFn = refEquality) {
  const { store, subscription: contextSub } = useContext(ReactReduxContext);
  
  return useSelectorWithStoreAndSubscription(
    selector,
    equalityFn,
    store,
    contextSub
  );
}

function refEquality(a, b) {
  return a === b;
}
```

### Critical Implementation Details

#### 1. **Double-Invocation Protection**

Notice the refs: `latestSelector`, `latestStoreState`, `latestSelectedState`. These prevent unnecessary re-renders:

```javascript
// Scenario: useSelector called during render
const user = useSelector(state => state.user);

// If store hasn't changed since last render
if (storeState !== latestStoreState.current) {
  // Run selector
} else {
  // Return cached value - avoid selector re-execution
}
```

#### 2. **Equality Check Strategy**

```javascript
// Default equality: reference equality
function refEquality(a, b) {
  return a === b;
}

// Problem with derived state:
const todos = useSelector(state => 
  state.todos.filter(t => !t.completed) // New array every time!
);

// Solution: shallow equality
import { shallowEqual } from 'react-redux';
const todos = useSelector(
  state => state.todos.filter(t => !t.completed),
  shallowEqual
);

// shallowEqual implementation
function shallowEqual(objA, objB) {
  if (objA === objB) return true;
  
  if (typeof objA !== 'object' || objA === null ||
      typeof objB !== 'object' || objB === null) {
    return false;
  }

  const keysA = Object.keys(objA);
  const keysB = Object.keys(objB);

  if (keysA.length !== keysB.length) return false;

  for (let i = 0; i < keysA.length; i++) {
    if (!objB.hasOwnProperty(keysA[i]) || objA[keysA[i]] !== objB[keysA[i]]) {
      return false;
    }
  }

  return true;
}
```

#### 3. **The `checkForUpdates` Function**

This is the subscription callback that runs when Redux store updates:

```javascript
function checkForUpdates() {
  try {
    const newStoreState = store.getState();
    
    // Optimization 1: Bail if state reference hasn't changed
    if (newStoreState === latestStoreState.current) {
      return;
    }

    // Optimization 2: Run selector
    const newSelectedState = latestSelector.current(newStoreState);

    // Optimization 3: Bail if selected value hasn't changed
    if (equalityFn(newSelectedState, latestSelectedState.current)) {
      return;
    }

    // Update refs
    latestSelectedState.current = newSelectedState;
    latestStoreState.current = newStoreState;
  } catch (err) {
    latestSubscriptionCallbackError.current = err;
  }

  // Force re-render
  forceRender();
}
```

**Three-level optimization:**
1. Store reference check (cheapest)
2. Selector execution (moderate cost)
3. Equality check (varies by `equalityFn`)

#### 4. **Why `useIsomorphicLayoutEffect`?**

```javascript
// react-redux/src/utils/useIsomorphicLayoutEffect.js
export const useIsomorphicLayoutEffect =
  typeof window !== 'undefined' &&
  typeof window.document !== 'undefined' &&
  typeof window.document.createElement !== 'undefined'
    ? useLayoutEffect
    : useEffect;
```

**Reason**: During SSR, `useLayoutEffect` triggers a warning. This abstraction:
- Uses `useLayoutEffect` on client (synchronous, before paint)
- Uses `useEffect` on server (avoids warning)

For subscription setup, we need `useLayoutEffect` because:
- Subscription must be ready before browser paint
- Ensures no missed updates between render and subscription
- Prevents "zombie child" problem (child subscribes before parent updates)

---

## Part 5: The `useDispatch` Hook

Much simpler than `useSelector`:

```javascript
// react-redux/src/hooks/useDispatch.js
import { useContext } from 'react';
import { ReactReduxContext } from '../components/Context';

export function useDispatch() {
  const { store } = useContext(ReactReduxContext);
  return store.dispatch;
}
```

**Why so simple?**
- `dispatch` function never changes for a given store
- No need for subscription or optimization
- Components using only `dispatch` (no `useSelector`) never re-render from state changes

---

## Part 6: Performance Patterns and Pitfalls

### Problem 1: Selector Instability

```javascript
// BAD: Creates new selector function every render
function TodoList() {
  const [filter, setFilter] = useState('all');
  
  // This selector closure is created on every render!
  const todos = useSelector(state => 
    state.todos.filter(t => t.category === filter)
  );
}
```

**Why this is bad:**
- `checkForUpdates` compares `latestSelector.current !== selector`
- New function reference every render triggers re-selection
- Defeats memoization

**Solution 1: useCallback**

```javascript
const selectTodos = useCallback(
  state => state.todos.filter(t => t.category === filter),
  [filter]
);
const todos = useSelector(selectTodos);
```

**Solution 2: Memoized selectors with Reselect**

```javascript
import { createSelector } from 'reselect';

// Input selectors
const selectTodos = state => state.todos;
const selectFilter = (state, filter) => filter;

// Memoized selector
const selectFilteredTodos = createSelector(
  [selectTodos, selectFilter],
  (todos, filter) => todos.filter(t => t.category === filter)
);

// In component
function TodoList() {
  const [filter, setFilter] = useState('all');
  const todos = useSelector(state => selectFilteredTodos(state, filter));
}
```

**How Reselect works:**

```javascript
// Simplified createSelector
function createSelector(inputSelectors, resultFunc) {
  let lastArgs = null;
  let lastResult = null;

  return function memoizedSelector(...args) {
    const inputResults = inputSelectors.map(selector => selector(...args));
    
    // Shallow compare input results
    if (lastArgs === null || !shallowEqual(inputResults, lastArgs)) {
      lastArgs = inputResults;
      lastResult = resultFunc(...inputResults);
    }
    
    return lastResult;
  };
}
```

### Problem 2: Derived State in Reducers vs Selectors

```javascript
// BAD: Computing in reducer
function todosReducer(state, action) {
  switch (action.type) {
    case 'ADD_TODO':
      const newState = {
        ...state,
        items: [...state.items, action.payload],
      };
      
      // Computing completed count in reducer
      newState.completedCount = newState.items.filter(t => t.completed).length;
      return newState;
  }
}

// GOOD: Computing in selector
const selectCompletedCount = createSelector(
  state => state.todos.items,
  items => items.filter(t => t.completed).length
);
```

**Why?**
- Reducers run on every action (expensive)
- Selectors run only when subscribed components render
- Selectors can be memoized per-component

### Problem 3: Multiple `useSelector` Calls

```javascript
// Causes two separate subscriptions
function UserProfile() {
  const userName = useSelector(state => state.user.name);
  const userEmail = useSelector(state => state.user.email);
  
  // If user object updates, this component renders TWICE
  // Once for each selector detecting a change
}

// Better: Select once
function UserProfile() {
  const user = useSelector(state => ({
    name: state.user.name,
    email: state.user.email
  }), shallowEqual); // MUST use shallowEqual!
}
```

---

## Part 7: Redux Toolkit — Modern Redux Architecture

Redux Toolkit (RTK) is the official, opinionated toolset. It simplifies Redux significantly.

### `configureStore` — Enhanced Store Creation

```javascript
// What configureStore does internally
function configureStore({ reducer, middleware, devTools, preloadedState, enhancers }) {
  let rootReducer;

  if (typeof reducer === 'function') {
    rootReducer = reducer;
  } else {
    // Combine reducers if object passed
    rootReducer = combineReducers(reducer);
  }

  // Default middleware includes thunk + dev checks
  let finalMiddleware = middleware;
  if (typeof finalMiddleware === 'function') {
    finalMiddleware = finalMiddleware(getDefaultMiddleware);
  } else if (!finalMiddleware) {
    finalMiddleware = getDefaultMiddleware();
  }

  // Apply middleware
  const middlewareEnhancer = applyMiddleware(...finalMiddleware);

  // Add Redux DevTools
  let storeEnhancers = [middlewareEnhancer];
  if (devTools) {
    const devToolsEnhancer = window.__REDUX_DEVTOOLS_EXTENSION__?.();
    if (devToolsEnhancer) {
      storeEnhancers.push(devToolsEnhancer);
    }
  }

  const composedEnhancer = compose(...storeEnhancers);

  return createStore(rootReducer, preloadedState, composedEnhancer);
}

// getDefaultMiddleware includes:
function getDefaultMiddleware() {
  return [
    thunkMiddleware,
    
    // Warns if you mutate state
    immutableStateInvariantMiddleware,
    
    // Warns if non-serializable values in state/actions
    serializableStateInvariantMiddleware,
  ];
}
```

### `createSlice` — Reducer + Actions Generator

```javascript
// Using createSlice
const todosSlice = createSlice({
  name: 'todos',
  initialState: [],
  reducers: {
    addTodo: (state, action) => {
      // Immer magic: "mutate" draft state
      state.push(action.payload);
    },
    toggleTodo: (state, action) => {
      const todo = state.find(t => t.id === action.payload);
      if (todo) {
        todo.completed = !todo.completed;
      }
    }
  }
});

// Auto-generated:
// todosSlice.actions.addTodo(payload) => { type: 'todos/addTodo', payload }
// todosSlice.reducer => actual reducer function
```

**How createSlice works internally:**

```javascript
function createSlice({ name, initialState, reducers, extraReducers }) {
  const actionCreators = {};
  const sliceReducers = {};

  // Process each reducer
  Object.keys(reducers).forEach(reducerName => {
    const type = `${name}/${reducerName}`;
    
    // Create action creator
    actionCreators[reducerName] = createAction(type);
    
    // Wrap reducer with Immer
    sliceReducers[type] = (state, action) => {
      return produce(state, draft => {
        reducers[reducerName](draft, action);
      });
    };
  });

  // Build final reducer
  const reducer = (state = initialState, action) => {
    const caseReducer = sliceReducers[action.type];
    if (caseReducer) {
      return caseReducer(state, action);
    }
    return state;
  };

  return {
    actions: actionCreators,
    reducer,
    name,
  };
}
```

### Immer Integration — The "Mutable" Syntax

RTK uses [Immer](https://github.com/immerjs/immer) library:

```javascript
// Immer's produce function (simplified concept)
function produce(baseState, recipe) {
  // Create a Proxy wrapper around state
  const draft = createDraft(baseState);
  
  // Run user's "mutation" code on draft
  recipe(draft);
  
  // Generate new immutable state from draft changes
  return finalize(draft);
}

// Example of Proxy-based draft
function createDraft(base) {
  return new Proxy(base, {
    get(target, prop) {
      // Track reads
      return target[prop];
    },
    set(target, prop, value) {
      // Track writes - mark as modified
      markModified(target, prop);
      target[prop] = value;
      return true;
    }
  });
}
```

**Why this works:**
- Immer tracks which parts of state you "mutate"
- Only those parts are cloned; rest is structurally shared
- Returns brand new root object (for Redux's equality checks)
- Maintains immutability guarantees at the interface level

---

## Part 8: Async Logic — createAsyncThunk

Async operations are common in Redux. RTK provides `createAsyncThunk`:

```javascript
import { createAsyncThunk, createSlice } from '@reduxjs/toolkit';

// Define async thunk
const fetchUser = createAsyncThunk(
  'user/fetchUser',
  async (userId, thunkAPI) => {
    const response = await fetch(`/api/users/${userId}`);
    const data = await response.json();
    return data; // becomes action.payload
  }
);

// Slice handles lifecycle actions
const userSlice = createSlice({
  name: 'user',
  initialState: {
    entity: null,
    loading: false,
    error: null,
  },
  reducers: {},
  extraReducers: (builder) => {
    builder
      .addCase(fetchUser.pending, (state) => {
        state.loading = true;
        state.error = null;
      })
      .addCase(fetchUser.fulfilled, (state, action) => {
        state.loading = false;
        state.entity = action.payload;
      })
      .addCase(fetchUser.rejected, (state, action) => {
        state.loading = false;
        state.error = action.error.message;
      });
  },
});
```

**How createAsyncThunk works:**

```javascript
function createAsyncThunk(typePrefix, payloadCreator) {
  // Generate action types
  const pending = `${typePrefix}/pending`;
  const fulfilled = `${typePrefix}/fulfilled`;
  const rejected = `${typePrefix}/rejected`;

  // Action creators
  const actionCreators = {
    pending: createAction(pending),
    fulfilled: createAction(fulfilled),
    rejected: createAction(rejected),
  };

  // The thunk itself
  const thunk = (arg) => async (dispatch, getState, extra) => {
    const requestId = generateRequestId();
    
    // Dispatch pending
    dispatch(actionCreators.pending({ arg, requestId }));

    try {
      // Execute async logic
      const result = await payloadCreator(arg, {
        dispatch,
        getState,
        extra,
        requestId,
        signal: abortController.signal,
      });

      // Dispatch fulfilled
      dispatch(actionCreators.fulfilled(result, { arg, requestId }));
      return result;
    } catch (err) {
      // Dispatch rejected
      dispatch(actionCreators.rejected(err, { arg, requestId }));
      throw err;
    }
  };

  // Attach action creators to thunk for extraReducers
  thunk.pending = actionCreators.pending;
  thunk.fulfilled = actionCreators.fulfilled;
  thunk.rejected = actionCreators.rejected;

  return thunk;
}
```

**Key features:**
1. **Automatic lifecycle actions**: pending/fulfilled/rejected
2. **AbortController integration**: Can cancel requests
3. **Request deduplication**: Via requestId
4. **Error handling**: Normalized error format

---

## Part 9: RTK Query — The Data Fetching Layer

RTK Query is built on top of Redux Toolkit and provides a complete data fetching solution.

### Basic Setup

```javascript
import { createApi, fetchBaseQuery } from '@reduxjs/toolkit/query/react';

const api = createApi({
  baseQuery: fetchBaseQuery({ baseUrl: '/api' }),
  endpoints: (builder) => ({
    getUser: builder.query({
      query: (userId) => `/users/${userId}`,
    }),
    updateUser: builder.mutation({
      query: ({ id, ...patch }) => ({
        url: `/users/${id}`,
        method: 'PATCH',
        body: patch,
      }),
    }),
  }),
});

// Auto-generated hooks
export const { useGetUserQuery, useUpdateUserMutation } = api;
```

### Internal Architecture

RTK Query generates:
1. **Slice reducer** for cache management
2. **Middleware** for request lifecycle
3. **Custom hooks** for components

```javascript
// What createApi generates (simplified)
function createApi(options) {
  const { baseQuery, endpoints } = options;

  // 1. Create the cache slice
  const cacheSlice = createSlice({
    name: 'api',
    initialState: {
      queries: {},
      mutations: {},
      subscriptions: {},
    },
    reducers: {
      // Reducers for cache management
      queryStarted(state, action) {
        state.queries[action.meta.requestId] = {
          status: 'pending',
          startedTimeStamp: Date.now(),
        };
      },
      queryFulfilled(state, action) {
        state.queries[action.meta.requestId] = {
          status: 'fulfilled',
          data: action.payload,
          fulfilledTimeStamp: Date.now(),
        };
      },
      queryRejected(state, action) {
        state.queries[action.meta.requestId] = {
          status: 'rejected',
          error: action.payload,
        };
      },
    },
  });

  // 2. Create middleware
  const middleware = (storeAPI) => (next) => (action) => {
    // Handle query/mutation actions
    if (isQueryAction(action)) {
      const promise = baseQuery(action.meta.arg).then(
        result => {
          storeAPI.dispatch(
            cacheSlice.actions.queryFulfilled({ ...result, meta: action.meta })
          );
          return result;
        },
        error => {
          storeAPI.dispatch(
            cacheSlice.actions.queryRejected({ error, meta: action.meta })
          );
          throw error;
        }
      );
      
      // Attach promise to action for hook consumption
      action.meta.promise = promise;
    }
    
    return next(action);
  };

  // 3. Generate endpoint hooks
  const hooks = {};
  Object.keys(endpoints).forEach(endpointName => {
    const endpoint = endpoints[endpointName];
    
    if (endpoint.type === 'query') {
      hooks[`use${capitalize(endpointName)}Query`] = (arg, options) => {
        return useQuery(endpointName, arg, options);
      };
    } else if (endpoint.type === 'mutation') {
      hooks[`use${capitalize(endpointName)}Mutation`] = () => {
        return useMutation(endpointName);
      };
    }
  });

  return {
    reducer: cacheSlice.reducer,
    middleware,
    ...hooks,
    endpoints: endpointBuilder(endpoints),
  };
}
```

### The `useQuery` Hook Implementation

```javascript
function useQuery(endpointName, arg, options = {}) {
  const dispatch = useDispatch();
  const [requestId] = useState(() => generateRequestId());
  
  // Select data from cache
  const result = useSelector(state => 
    selectQueryResult(state, endpointName, arg)
  );

  // Manage subscription
  useEffect(() => {
    // Subscribe to this query
    dispatch({
      type: 'api/subscriptionAdded',
      payload: { endpointName, arg, requestId },
    });

    // Start fetching if needed
    if (shouldFetch(result, options)) {
      dispatch({
        type: 'api/queryStarted',
        meta: { endpointName, arg, requestId },
      });
    }

    return () => {
      // Unsubscribe
      dispatch({
        type: 'api/subscriptionRemoved',
        payload: { endpointName, arg, requestId },
      });
    };
  }, [endpointName, arg, requestId]);

  // Refetch function
  const refetch = useCallback(() => {
    dispatch({
      type: 'api/queryStarted',
      meta: { endpointName, arg, requestId, forceRefetch: true },
    });
  }, [endpointName, arg, requestId]);

  return {
    data: result?.data,
    error: result?.error,
    isLoading: result?.status === 'pending',
    isSuccess: result?.status === 'fulfilled',
    isError: result?.status === 'rejected',
    refetch,
  };
}
```

### Cache Management and Invalidation

RTK Query implements sophisticated caching:

```javascript
const api = createApi({
  baseQuery: fetchBaseQuery({ baseUrl: '/api' }),
  tagTypes: ['User', 'Post'],
  endpoints: (builder) => ({
    getUser: builder.query({
      query: (id) => `/users/${id}`,
      providesTags: (result, error, id) => [{ type: 'User', id }],
    }),
    getUserPosts: builder.query({
      query: (userId) => `/users/${userId}/posts`,
      providesTags: (result, error, userId) => 
        result
          ? [
              ...result.map(({ id }) => ({ type: 'Post', id })),
              { type: 'Post', id: 'LIST' },
            ]
          : [{ type: 'Post', id: 'LIST' }],
    }),
    updateUser: builder.mutation({
      query: ({ id, ...patch }) => ({
        url: `/users/${id}`,
        method: 'PATCH',
        body: patch,
      }),
      invalidatesTags: (result, error, { id }) => [{ type: 'User', id }],
    }),
    addPost: builder.mutation({
      query: (post) => ({
        url: '/posts',
        method: 'POST',
        body: post,
      }),
      invalidatesTags: [{ type: 'Post', id: 'LIST' }],
    }),
  }),
});
```

**How tag-based invalidation works:**

```javascript
// Inside RTK Query middleware
function handleMutationFulfilled(action, storeAPI) {
  const { invalidatesTags } = action.meta.endpoint;
  
  if (!invalidatesTags) return;
  
  // Get tags to invalidate
  const tagsToInvalidate = typeof invalidatesTags === 'function'
    ? invalidatesTags(action.payload, null, action.meta.arg)
    : invalidatesTags;

  // Find all queries that provided these tags
  const state = storeAPI.getState();
  const queriesToInvalidate = [];

  Object.entries(state.api.queries).forEach(([queryCacheKey, queryState]) => {
    const providedTags = queryState.providedTags || [];
    
    // Check if any provided tag matches invalidation
    const shouldInvalidate = providedTags.some(providedTag =>
      tagsToInvalidate.some(invalidateTag =>
        matchesTag(providedTag, invalidateTag)
      )
    );

    if (shouldInvalidate) {
      queriesToInvalidate.push(queryCacheKey);
    }
  });

  // Refetch invalidated queries
  queriesToInvalidate.forEach(queryCacheKey => {
    const queryState = state.api.queries[queryCacheKey];
    if (queryState.subscriptionCount > 0) {
      // Query is actively subscribed - refetch it
      storeAPI.dispatch({
        type: 'api/queryStarted',
        meta: {
          ...queryState.meta,
          forceRefetch: true,
        },
      });
    } else {
      // Mark as invalidated but don't refetch until next subscription
      storeAPI.dispatch({
        type: 'api/queryInvalidated',
        payload: queryCacheKey,
      });
    }
  });
}

function matchesTag(provided, invalidate) {
  return (
    provided.type === invalidate.type &&
    (invalidate.id === undefined || provided.id === invalidate.id)
  );
}
```

**Key concepts:**

1. **providesTags**: Queries declare what data they contain
2. **invalidatesTags**: Mutations declare what data they affect
3. **Automatic refetch**: Subscribed queries refetch when invalidated
4. **Lazy invalidation**: Unsubscribed queries marked stale, refetch on next mount

### Optimistic Updates

```javascript
const api = createApi({
  // ...
  endpoints: (builder) => ({
    updatePost: builder.mutation({
      query: ({ id, ...patch }) => ({
        url: `/posts/${id}`,
        method: 'PATCH',
        body: patch,
      }),
      async onQueryStarted({ id, ...patch }, { dispatch, queryFulfilled }) {
        // Optimistically update cache
        const patchResult = dispatch(
          api.util.updateQueryData('getPost', id, (draft) => {
            Object.assign(draft, patch);
          })
        );

        try {
          await queryFulfilled;
        } catch {
          // Undo optimistic update on error
          patchResult.undo();
        }
      },
    }),
  }),
});
```

**How `updateQueryData` works:**

```javascript
// Simplified implementation
function updateQueryData(endpointName, arg, updateRecipe) {
  return (dispatch, getState) => {
    const queryCacheKey = getQueryCacheKey(endpointName, arg);
    const state = getState();
    const queryState = state.api.queries[queryCacheKey];

    if (!queryState || !queryState.data) {
      return { undo: () => {} };
    }

    const originalData = queryState.data;
    
    // Use Immer to apply optimistic update
    const newData = produce(originalData, updateRecipe);

    // Dispatch update action
    dispatch({
      type: 'api/queryDataUpdated',
      payload: {
        queryCacheKey,
        data: newData,
      },
    });

    // Return undo function
    return {
      undo: () => {
        dispatch({
          type: 'api/queryDataUpdated',
          payload: {
            queryCacheKey,
            data: originalData,
          },
        });
      },
    };
  };
}
```

---

## Part 10: Advanced Patterns and Architecture

### Pattern 1: Normalized State Structure

For relational data, normalize your state:

```javascript
// BAD: Nested structure
{
  posts: [
    {
      id: 1,
      title: "Post 1",
      author: { id: 1, name: "Alice" },
      comments: [
        { id: 1, text: "Comment 1", author: { id: 2, name: "Bob" } }
      ]
    }
  ]
}

// GOOD: Normalized structure
{
  users: {
    byId: {
      1: { id: 1, name: "Alice" },
      2: { id: 2, name: "Bob" }
    },
    allIds: [1, 2]
  },
  posts: {
    byId: {
      1: { id: 1, title: "Post 1", authorId: 1, commentIds: [1] }
    },
    allIds: [1]
  },
  comments: {
    byId: {
      1: { id: 1, text: "Comment 1", authorId: 2, postId: 1 }
    },
    allIds: [1]
  }
}
```

**Why normalize?**

1. **Single source of truth**: Each entity exists once
2. **Easy updates**: Change Alice's name in one place
3. **Efficient selectors**: Direct lookup by ID
4. **Prevents stale data**: No duplicate copies to sync

**Use `@reduxjs/toolkit`'s `createEntityAdapter`:**

```javascript
import { createEntityAdapter, createSlice } from '@reduxjs/toolkit';

const usersAdapter = createEntityAdapter();

const usersSlice = createSlice({
  name: 'users',
  initialState: usersAdapter.getInitialState(),
  reducers: {
    userAdded: usersAdapter.addOne,
    usersReceived: usersAdapter.setAll,
    userUpdated: usersAdapter.updateOne,
  },
});

// Generates state shape:
// {
//   ids: [1, 2, 3],
//   entities: {
//     1: { id: 1, name: 'Alice' },
//     2: { id: 2, name: 'Bob' },
//     3: { id: 3, name: 'Charlie' }
//   }
// }

// Auto-generated selectors
const selectors = usersAdapter.getSelectors(state => state.users);
// selectors.selectIds
// selectors.selectEntities
// selectors.selectAll
// selectors.selectById
// selectors.selectTotal
```

**createEntityAdapter internals:**

```javascript
function createEntityAdapter(options = {}) {
  const { selectId = (entity) => entity.id, sortComparer } = options;

  // Initial state structure
  function getInitialState(additionalState = {}) {
    return {
      ids: [],
      entities: {},
      ...additionalState,
    };
  }

  // Add one entity
  function addOne(state, entity) {
    const id = selectId(entity);
    if (id in state.entities) return;

    state.ids.push(id);
    state.entities[id] = entity;

    if (sortComparer) {
      state.ids.sort((aId, bId) => 
        sortComparer(state.entities[aId], state.entities[bId])
      );
    }
  }

  // Update one entity
  function updateOne(state, update) {
    const { id, changes } = update;
    if (!(id in state.entities)) return;

    Object.assign(state.entities[id], changes);

    if (sortComparer) {
      state.ids.sort((aId, bId) => 
        sortComparer(state.entities[aId], state.entities[bId])
      );
    }
  }

  // Remove one entity
  function removeOne(state, id) {
    if (!(id in state.entities)) return;

    delete state.entities[id];
    state.ids = state.ids.filter(existingId => existingId !== id);
  }

  // Set all entities
  function setAll(state, entities) {
    state.ids = [];
    state.entities = {};

    entities.forEach(entity => {
      const id = selectId(entity);
      state.ids.push(id);
      state.entities[id] = entity;
    });

    if (sortComparer) {
      state.ids.sort((aId, bId) => 
        sortComparer(state.entities[aId], state.entities[bId])
      );
    }
  }

  // Selectors
  function getSelectors(selectState) {
    const selectIds = (state) => selectState(state).ids;
    const selectEntities = (state) => selectState(state).entities;
    
    const selectAll = createSelector(
      selectIds,
      selectEntities,
      (ids, entities) => ids.map(id => entities[id])
    );

    const selectById = (state, id) => selectState(state).entities[id];
    
    const selectTotal = (state) => selectState(state).ids.length;

    return {
      selectIds,
      selectEntities,
      selectAll,
      selectById,
      selectTotal,
    };
  }

  return {
    getInitialState,
    getSelectors,
    addOne: createReducer(addOne),
    addMany: createReducer(addMany),
    setAll: createReducer(setAll),
    updateOne: createReducer(updateOne),
    updateMany: createReducer(updateMany),
    removeOne: createReducer(removeOne),
    removeMany: createReducer(removeMany),
  };
}
```

### Pattern 2: Redux Middleware for Side Effects

Custom middleware example - API call logging:

```javascript
const apiLogger = (store) => (next) => (action) => {
  if (action.type.endsWith('/pending')) {
    console.log('API Request Started:', {
      type: action.type,
      timestamp: new Date().toISOString(),
      payload: action.payload,
    });
  }

  if (action.type.endsWith('/fulfilled')) {
    console.log('API Request Succeeded:', {
      type: action.type,
      timestamp: new Date().toISOString(),
      data: action.payload,
    });
  }

  if (action.type.endsWith('/rejected')) {
    console.error('API Request Failed:', {
      type: action.type,
      timestamp: new Date().toISOString(),
      error: action.error,
    });
  }

  return next(action);
};
```

**Analytics middleware:**

```javascript
const analyticsMiddleware = (store) => (next) => (action) => {
  // Track specific user actions
  if (action.type === 'user/login') {
    analytics.track('User Login', {
      userId: action.payload.id,
      method: action.payload.method,
    });
  }

  if (action.type === 'checkout/completed') {
    analytics.track('Purchase', {
      orderId: action.payload.orderId,
      revenue: action.payload.total,
      items: action.payload.items.length,
    });
  }

  return next(action);
};
```

**Crash reporting middleware:**

```javascript
const crashReporter = (store) => (next) => (action) => {
  try {
    return next(action);
  } catch (err) {
    console.error('Caught an exception!', err);
    
    // Send to error tracking service
    Sentry.captureException(err, {
      extra: {
        action,
        state: store.getState(),
      },
    });

    throw err;
  }
};
```

### Pattern 3: Redux Persistence

Persisting state to localStorage:

```javascript
// redux-persist simplified concept
const persistMiddleware = (store) => (next) => (action) => {
  const result = next(action);
  
  // After every action, save state
  const state = store.getState();
  
  try {
    const serialized = JSON.stringify(state);
    localStorage.setItem('redux-state', serialized);
  } catch (err) {
    console.error('Failed to persist state:', err);
  }

  return result;
};

// Load persisted state on startup
function loadState() {
  try {
    const serialized = localStorage.getItem('redux-state');
    if (serialized === null) {
      return undefined; // Let reducers initialize
    }
    return JSON.parse(serialized);
  } catch (err) {
    console.error('Failed to load state:', err);
    return undefined;
  }
}

// Create store with persisted state
const store = configureStore({
  reducer: rootReducer,
  preloadedState: loadState(),
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware().concat(persistMiddleware),
});
```

**Using redux-persist library:**

```javascript
import { persistStore, persistReducer } from 'redux-persist';
import storage from 'redux-persist/lib/storage'; // localStorage

const persistConfig = {
  key: 'root',
  storage,
  whitelist: ['user', 'settings'], // Only persist these reducers
  blacklist: ['ui'], // Don't persist UI state
};

const persistedReducer = persistReducer(persistConfig, rootReducer);

const store = configureStore({
  reducer: persistedReducer,
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware({
      serializableCheck: {
        ignoredActions: ['persist/PERSIST', 'persist/REHYDRATE'],
      },
    }),
});

const persistor = persistStore(store);

// In React app
<Provider store={store}>
  <PersistGate loading={<LoadingSpinner />} persistor={persistor}>
    <App />
  </PersistGate>
</Provider>
```

### Pattern 4: Selector Composition and Memoization

Building complex selectors from simple ones:

```javascript
import { createSelector } from 'reselect';

// Input selectors (not memoized)
const selectTodos = (state) => state.todos;
const selectVisibilityFilter = (state) => state.visibilityFilter;
const selectSearchTerm = (state) => state.searchTerm;

// Memoized selectors
const selectFilteredTodos = createSelector(
  [selectTodos, selectVisibilityFilter],
  (todos, filter) => {
    switch (filter) {
      case 'SHOW_COMPLETED':
        return todos.filter(t => t.completed);
      case 'SHOW_ACTIVE':
        return todos.filter(t => !t.completed);
      default:
        return todos;
    }
  }
);

const selectSearchedTodos = createSelector(
  [selectFilteredTodos, selectSearchTerm],
  (todos, searchTerm) => {
    if (!searchTerm) return todos;
    
    const lowerSearch = searchTerm.toLowerCase();
    return todos.filter(t =>
      t.text.toLowerCase().includes(lowerSearch)
    );
  }
);

// Statistics selector
const selectTodoStats = createSelector(
  [selectTodos],
  (todos) => ({
    total: todos.length,
    completed: todos.filter(t => t.completed).length,
    active: todos.filter(t => !t.completed).length,
  })
);

// Composed selector with multiple inputs
const selectTodoListData = createSelector(
  [selectSearchedTodos, selectTodoStats, selectSearchTerm],
  (todos, stats, searchTerm) => ({
    todos,
    stats,
    hasSearch: Boolean(searchTerm),
    isEmpty: todos.length === 0,
  })
);
```

**Parametric selectors:**

```javascript
// Factory function for creating selectors with parameters
const makeSelectTodoById = () =>
  createSelector(
    [selectTodos, (state, todoId) => todoId],
    (todos, todoId) => todos.find(t => t.id === todoId)
  );

// In component
function TodoDetail({ todoId }) {
  // Create memoized selector instance per component
  const selectTodoById = useMemo(makeSelectTodoById, []);
  
  const todo = useSelector(state => selectTodoById(state, todoId));
  
  return <div>{todo.text}</div>;
}
```

**Why create selector instances?**
- Each component instance gets its own memoization cache
- Prevents cache thrashing when multiple components use same selector with different arguments

---

## Part 11: Performance Deep Dive

### React-Redux Rendering Behavior

**The render chain:**

1. Redux action dispatched
2. Store runs reducer, gets new state reference
3. Store notifies all subscriptions
4. React-Redux Provider's subscription runs
5. Provider notifies child subscriptions (components with `useSelector`)
6. Each `useSelector` runs its `checkForUpdates` function:
   - Compares store state reference
   - Runs selector if state changed
   - Compares selector output with equality function
   - If different, triggers `forceRender()`
7. React re-renders component
8. Component's children re-render (unless memoized with `React.memo`)

### Preventing Unnecessary Renders

**Problem: New object references every time**

```javascript
// BAD: New object on every selector call
const user = useSelector(state => ({
  name: state.user.name,
  email: state.user.email,
}));
// This ALWAYS triggers re-render, even if name and email haven't changed!

// SOLUTION 1: Use shallowEqual
import { shallowEqual } from 'react-redux';
const user = useSelector(
  state => ({
    name: state.user.name,
    email: state.user.email,
  }),
  shallowEqual
);

// SOLUTION 2: Select primitive value
const userName = useSelector(state => state.user.name);
const userEmail = useSelector(state => state.user.email);

// SOLUTION 3: Memoized selector
const selectUserBasicInfo = createSelector(
  state => state.user.name,
  state => state.user.email,
  (name, email) => ({ name, email })
);
const user = useSelector(selectUserBasicInfo);
```

### Batching Updates

React 18 automatically batches updates, but in React 17 and earlier:

```javascript
import { unstable_batchedUpdates } from 'react-dom';

// Multiple dispatches
function handleSaveAll() {
  unstable_batchedUpdates(() => {
    dispatch(saveUser(userData));
    dispatch(saveSettings(settingsData));
    dispatch(updateUI({ saved: true }));
  });
  // React only re-renders once after all three dispatches
}
```

**React-Redux automatically batches** store subscription notifications:

```javascript
// Inside React-Redux Provider
useEffect(() => {
  const { subscription } = contextValue;
  
  subscription.onStateChange = () => {
    // This batches all nested subscription notifications
    unstable_batchedUpdates(() => {
      subscription.notifyNestedSubs();
    });
  };
  
  subscription.trySubscribe();
}, [contextValue]);
```

### Measuring Performance

**Redux DevTools performance monitoring:**

```javascript
const store = configureStore({
  reducer: rootReducer,
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware({
      immutableCheck: { warnAfter: 128 },
      serializableCheck: { warnAfter: 64 },
    }),
  devTools: {
    trace: true, // Enable action stack traces
    traceLimit: 25,
  },
});
```

**Custom performance middleware:**

```javascript
const performanceMiddleware = (store) => (next) => (action) => {
  const start = performance.now();
  const result = next(action);
  const end = performance.now();
  
  const duration = end - start;
  
  if (duration > 16) { // Longer than one frame (60fps)
    console.warn(`Slow action: ${action.type} took ${duration.toFixed(2)}ms`);
  }
  
  return result;
};
```

**React component profiling:**

```javascript
import { Profiler } from 'react';

function onRenderCallback(
  id, // Component identifier
  phase, // "mount" or "update"
  actualDuration, // Time spent rendering
  baseDuration, // Estimated time without memoization
  startTime,
  commitTime,
  interactions // Set of interactions
) {
  if (actualDuration > 16) {
    console.log(`${id} (${phase}) took ${actualDuration.toFixed(2)}ms`);
  }
}

function App() {
  return (
    <Profiler id="App" onRender={onRenderCallback}>
      <TodoList />
    </Profiler>
  );
}
```

---

## Part 12: Testing Redux and React-Redux

### Testing Pure Redux Logic

**Testing reducers:**

```javascript
import { todosSlice } from './todosSlice';

describe('todos reducer', () => {
  it('should handle initial state', () => {
    expect(todosSlice.reducer(undefined, { type: 'unknown' })).toEqual([]);
  });

  it('should handle addTodo', () => {
    const actual = todosSlice.reducer(
      [],
      todosSlice.actions.addTodo({ id: 1, text: 'Run the tests' })
    );
    
    expect(actual).toEqual([{ id: 1, text: 'Run the tests' }]);
  });

  it('should handle toggleTodo', () => {
    const initial = [{ id: 1, text: 'Run the tests', completed: false }];
    
    const actual = todosSlice.reducer(
      initial,
      todosSlice.actions.toggleTodo(1)
    );
    
    expect(actual[0].completed).toBe(true);
  });
});
```

**Testing selectors:**

```javascript
import { selectFilteredTodos } from './selectors';

describe('selectFilteredTodos', () => {
  const state = {
    todos: [
      { id: 1, text: 'Test', completed: false },
      { id: 2, text: 'Build', completed: true },
    ],
    visibilityFilter: 'SHOW_ALL',
  };

  it('should select all todos when filter is SHOW_ALL', () => {
    const result = selectFilteredTodos(state);
    expect(result).toHaveLength(2);
  });

  it('should select only completed todos', () => {
    const result = selectFilteredTodos({
      ...state,
      visibilityFilter: 'SHOW_COMPLETED',
    });
    
    expect(result).toHaveLength(1);
    expect(result[0].completed).toBe(true);
  });

  it('should memoize results', () => {
    const result1 = selectFilteredTodos(state);
    const result2 = selectFilteredTodos(state);
    
    // Same reference = memoization working
    expect(result1).toBe(result2);
  });
});
```

**Testing async thunks:**

```javascript
import { fetchUser } from './userSlice';
import { configureStore } from '@reduxjs/toolkit';

describe('fetchUser thunk', () => {
  let store;

  beforeEach(() => {
    store = configureStore({
      reducer: {
        user: userReducer,
      },
    });
  });

  it('should fetch user successfully', async () => {
    // Mock fetch
    global.fetch = jest.fn(() =>
      Promise.resolve({
        json: () => Promise.resolve({ id: 1, name: 'Alice' }),
      })
    );

    await store.dispatch(fetchUser(1));

    const state = store.getState();
    expect(state.user.entity).toEqual({ id: 1, name: 'Alice' });
    expect(state.user.loading).toBe(false);
  });

  it('should handle fetch error', async () => {
    global.fetch = jest.fn(() => Promise.reject(new Error('Network error')));

    await store.dispatch(fetchUser(1));

    const state = store.getState();
    expect(state.user.error).toBe('Network error');
    expect(state.user.loading).toBe(false);
  });
});
```

### Testing React-Redux Connected Components

**Using React Testing Library:**

```javascript
import { render, screen } from '@testing-library/react';
import { Provider } from 'react-redux';
import { configureStore } from '@reduxjs/toolkit';
import TodoList from './TodoList';
import { todosSlice } from './todosSlice';

function renderWithRedux(
  ui,
  {
    preloadedState = {},
    store = configureStore({
      reducer: { todos: todosSlice.reducer },
      preloadedState,
    }),
    ...renderOptions
  } = {}
) {
  function Wrapper({ children }) {
    return <Provider store={store}>{children}</Provider>;
  }
  
  return { store, ...render(ui, { wrapper: Wrapper, ...renderOptions }) };
}

describe('TodoList', () => {
  it('should render todos from store', () => {
    const initialState = {
      todos: [
        { id: 1, text: 'Test Redux', completed: false },
        { id: 2, text: 'Write docs', completed: true },
      ],
    };

    renderWithRedux(<TodoList />, { preloadedState: initialState });

    expect(screen.getByText('Test Redux')).toBeInTheDocument();
    expect(screen.getByText('Write docs')).toBeInTheDocument();
  });

  it('should dispatch addTodo action', () => {
    const { store } = renderWithRedux(<TodoList />);
    
    const input = screen.getByPlaceholderText('Add todo');
    const button = screen.getByText('Add');

    userEvent.type(input, 'New todo');
    userEvent.click(button);

    const state = store.getState();
    expect(state.todos).toContainEqual(
      expect.objectContaining({ text: 'New todo' })
    );
  });
});
```

**Testing hooks in isolation:**

```javascript
import { renderHook } from '@testing-library/react-hooks';
import { Provider } from 'react-redux';
import { useSelector } from 'react-redux';
import { configureStore } from '@reduxjs/toolkit';

describe('useSelector with selectFilteredTodos', () => {
  it('should select filtered todos', () => {
    const store = configureStore({
      reducer: {
        todos: todosReducer,
        filter: filterReducer,
      },
      preloadedState: {
        todos: [{ id: 1, text: 'Test', completed: true }],
        filter: 'SHOW_COMPLETED',
      },
    });

    const wrapper = ({ children }) => (
      <Provider store={store}>{children}</Provider>
    );

    const { result } = renderHook(
      () => useSelector(selectFilteredTodos),
      { wrapper }
    );

    expect(result.current).toHaveLength(1);
  });
});
```

---

## Part 13: Common Pitfalls and Anti-Patterns

### Anti-Pattern 1: Mutating State in Reducers (Without Immer)

```javascript
// WRONG: Direct mutation
function todosReducer(state = [], action) {
  switch (action.type) {
    case 'ADD_TODO':
      state.push(action.payload); // MUTATES state!
      return state; // Returns same reference - Redux won't detect change!
    case 'TOGGLE_TODO':
      const todo = state.find(t => t.id === action.payload);
      todo.completed = !todo.completed; // MUTATES nested object!
      return state;
    default:
      return state;
  }
}

// CORRECT: Immutable updates
function todosReducer(state = [], action) {
  switch (action.type) {
    case 'ADD_TODO':
      return [...state, action.payload]; // New array
    case 'TOGGLE_TODO':
      return state.map(todo =>
        todo.id === action.payload
          ? { ...todo, completed: !todo.completed } // New object
          : todo
      );
    default:
      return state;
  }
}
```

**Why mutation breaks Redux:**

```javascript
// Redux's state comparison
function checkStateChanged(oldState, newState) {
  return oldState !== newState; // Reference equality check
}

// With mutation:
const oldState = [{ id: 1, completed: false }];
const newState = oldState;
newState[0].completed = true;

console.log(oldState === newState); // true - SAME REFERENCE!
// Redux thinks nothing changed, components don't re-render
```

### Anti-Pattern 2: Non-Serializable Values in State

```javascript
// WRONG: Storing functions, Promises, class instances
const initialState = {
  user: null,
  fetchUser: () => fetch('/api/user'), // Function - NOT serializable!
  currentRequest: fetch('/api/data'), // Promise - NOT serializable!
  dateObject: new Date(), // Class instance - loses methods on serialization
  mapData: new Map([['key', 'value']]), // Map - NOT serializable!
};

// CORRECT: Only plain serializable data
const initialState = {
  user: null,
  isLoading: false,
  timestamp: Date.now(), // Primitive number
  mapData: { key: 'value' }, // Plain object
};
```

**Why serialization matters:**

1. **Redux DevTools**: Needs to serialize state for time-travel debugging
2. **Persistence**: `redux-persist` requires JSON serialization
3. **Server-side rendering**: State must serialize for hydration
4. **Debugging**: `console.log` shows meaningful data

### Anti-Pattern 3: Overusing Redux

```javascript
// WRONG: Everything in Redux
const store = {
  form: {
    username: '',
    password: '',
    confirmPassword: '',
    // Form state rarely needs to be global
  },
  ui: {
    isModalOpen: false,
    hoveredItemId: null,
    scrollPosition: 0,
    // UI state is usually component-local
  },
};

// CORRECT: Use local state for local concerns
function LoginForm() {
  const [username, setUsername] = useState('');
  const [password, setPassword] = useState('');
  const dispatch = useDispatch();

  const handleSubmit = () => {
    // Only dispatch when data needs to be shared/persisted
    dispatch(loginUser({ username, password }));
  };

  return (
    <form onSubmit={handleSubmit}>
      <input value={username} onChange={e => setUsername(e.target.value)} />
      <input value={password} onChange={e => setPassword(e.target.value)} />
      <button type="submit">Login</button>
    </form>
  );
}
```

**When to use Redux vs local state:**

**Use Redux for:**
- Data shared across multiple components
- Data that needs to persist across route changes
- Complex state logic with multiple actions
- State that needs to be persisted or rehydrated
- State that benefits from time-travel debugging

**Use local state for:**
- Form inputs (unless multi-step or needs persistence)
- UI state (modals, tooltips, hover states)
- Temporary state within a single component tree
- Animation state
- Any state that doesn't need to outlive the component

### Anti-Pattern 4: Dispatching Actions in Render

```javascript
// WRONG: Side effect in render
function UserProfile({ userId }) {
  const dispatch = useDispatch();
  
  // This runs on EVERY render!
  dispatch(fetchUser(userId)); // SIDE EFFECT IN RENDER!
  
  const user = useSelector(state => state.user);
  return <div>{user?.name}</div>;
}

// CORRECT: Use useEffect for side effects
function UserProfile({ userId }) {
  const dispatch = useDispatch();
  const user = useSelector(state => state.user);

  useEffect(() => {
    dispatch(fetchUser(userId));
  }, [dispatch, userId]); // Runs only when userId changes

  return <div>{user?.name}</div>;
}
```

**Why this matters:**
- React may call render multiple times without committing
- Side effects in render can cause duplicate requests
- Breaks React's rendering model (renders should be pure)

### Anti-Pattern 5: Incorrect Selector Memoization

```javascript
// WRONG: Inline object creation in selector
function TodoList() {
  // This selector creates a NEW object every time!
  const filteredTodos = useSelector(state => ({
    todos: state.todos.filter(t => !t.completed),
    count: state.todos.length,
  }));
  
  // Component re-renders on EVERY state change, even unrelated ones!
}

// CORRECT: Use createSelector
const selectTodoData = createSelector(
  state => state.todos,
  todos => ({
    todos: todos.filter(t => !t.completed),
    count: todos.length,
  })
);

function TodoList() {
  const todoData = useSelector(selectTodoData);
  // Only re-renders when todos array changes
}
```

### Anti-Pattern 6: Dispatching Actions in Reducers

```javascript
// WRONG: Side effects in reducer
function userReducer(state, action) {
  switch (action.type) {
    case 'USER_LOGIN_SUCCESS':
      // NEVER dispatch from reducer!
      store.dispatch({ type: 'ANALYTICS_TRACK_LOGIN' });
      
      // NEVER make API calls from reducer!
      fetch('/api/log-login');
      
      return {
        ...state,
        user: action.payload,
        isAuthenticated: true,
      };
  }
}

// CORRECT: Use middleware or thunks
const analyticsMiddleware = store => next => action => {
  if (action.type === 'USER_LOGIN_SUCCESS') {
    // Side effects in middleware
    analytics.track('User Login');
    fetch('/api/log-login');
  }
  return next(action);
};
```

**Why reducers must be pure:**
- Time-travel debugging requires replaying actions
- Server-side rendering requires predictable state
- Testing requires deterministic behavior
- Enables optimizations (memoization, batching)

---

## Part 14: Migration Strategies and Best Practices

### Migrating from Class Components with `connect`

**Old pattern (deprecated but still works):**

```javascript
import { connect } from 'react-redux';

class TodoList extends React.Component {
  render() {
    const { todos, addTodo } = this.props;
    return (
      <div>
        {todos.map(todo => (
          <TodoItem key={todo.id} {...todo} />
        ))}
        <button onClick={() => addTodo('New todo')}>Add</button>
      </div>
    );
  }
}

const mapStateToProps = state => ({
  todos: state.todos,
});

const mapDispatchToProps = dispatch => ({
  addTodo: text => dispatch({ type: 'ADD_TODO', payload: text }),
});

export default connect(mapStateToProps, mapDispatchToProps)(TodoList);
```

**Modern pattern with hooks:**

```javascript
import { useSelector, useDispatch } from 'react-redux';
import { todosSlice } from './todosSlice';

function TodoList() {
  const todos = useSelector(state => state.todos);
  const dispatch = useDispatch();

  const handleAddTodo = () => {
    dispatch(todosSlice.actions.addTodo('New todo'));
  };

  return (
    <div>
      {todos.map(todo => (
        <TodoItem key={todo.id} {...todo} />
      ))}
      <button onClick={handleAddTodo}>Add</button>
    </div>
  );
}

export default TodoList;
```

**How `connect` worked internally:**

```javascript
// Simplified connect implementation
function connect(mapStateToProps, mapDispatchToProps) {
  return function wrapWithConnect(WrappedComponent) {
    function ConnectedComponent(props) {
      const store = useContext(ReactReduxContext).store;
      const [, forceRender] = useReducer(s => s + 1, 0);

      // Map state
      const stateProps = mapStateToProps
        ? mapStateToProps(store.getState(), props)
        : {};

      // Map dispatch
      const dispatchProps = mapDispatchToProps
        ? typeof mapDispatchToProps === 'function'
          ? mapDispatchToProps(store.dispatch, props)
          : bindActionCreators(mapDispatchToProps, store.dispatch)
        : { dispatch: store.dispatch };

      // Merge props
      const mergedProps = { ...props, ...stateProps, ...dispatchProps };

      // Subscribe to store
      useEffect(() => {
        const unsubscribe = store.subscribe(() => {
          const newStateProps = mapStateToProps(store.getState(), props);
          
          // Shallow compare
          if (!shallowEqual(stateProps, newStateProps)) {
            forceRender();
          }
        });

        return unsubscribe;
      }, [store]);

      return <WrappedComponent {...mergedProps} />;
    }

    return ConnectedComponent;
  };
}
```

### Gradual Migration Path

**Step 1: Install Redux Toolkit alongside existing Redux**

```bash
npm install @reduxjs/toolkit
```

**Step 2: Convert one slice at a time**

```javascript
// Old: Hand-written reducer
const ADD_TODO = 'ADD_TODO';
const TOGGLE_TODO = 'TOGGLE_TODO';

export const addTodo = text => ({ type: ADD_TODO, payload: text });
export const toggleTodo = id => ({ type: TOGGLE_TODO, payload: id });

function todosReducer(state = [], action) {
  switch (action.type) {
    case ADD_TODO:
      return [...state, { id: Date.now(), text: action.payload, completed: false }];
    case TOGGLE_TODO:
      return state.map(todo =>
        todo.id === action.payload
          ? { ...todo, completed: !todo.completed }
          : todo
      );
    default:
      return state;
  }
}

// New: RTK slice
import { createSlice } from '@reduxjs/toolkit';

const todosSlice = createSlice({
  name: 'todos',
  initialState: [],
  reducers: {
    addTodo: (state, action) => {
      state.push({
        id: Date.now(),
        text: action.payload,
        completed: false,
      });
    },
    toggleTodo: (state, action) => {
      const todo = state.find(t => t.id === action.payload);
      if (todo) {
        todo.completed = !todo.completed;
      }
    },
  },
});

export const { addTodo, toggleTodo } = todosSlice.actions;
export default todosSlice.reducer;
```

**Step 3: Update store configuration**

```javascript
// Old
import { createStore, combineReducers, applyMiddleware } from 'redux';
import thunk from 'redux-thunk';

const rootReducer = combineReducers({
  todos: todosReducer,
  user: userReducer,
});

const store = createStore(rootReducer, applyMiddleware(thunk));

// New (can coexist with old reducers)
import { configureStore } from '@reduxjs/toolkit';
import todosReducer from './todosSlice'; // New RTK slice
import userReducer from './userReducer'; // Old reducer still works

const store = configureStore({
  reducer: {
    todos: todosReducer, // RTK slice
    user: userReducer,   // Old reducer
  },
  // Thunk included by default, plus dev checks
});
```

**Step 4: Replace `connect` with hooks gradually**

You can mix both patterns:

```javascript
// Some components still use connect
export default connect(mapStateToProps, mapDispatchToProps)(OldComponent);

// New components use hooks
function NewComponent() {
  const data = useSelector(selectData);
  const dispatch = useDispatch();
  // ...
}
```

---

## Part 15: Advanced Architecture Patterns

### Pattern 1: Domain-Driven State Structure

Organize state by business domain, not technical role:

```javascript
// AVOID: Organizing by technical type
{
  entities: { /* all entities */ },
  ui: { /* all UI state */ },
  network: { /* all request state */ }
}

// PREFER: Organizing by domain
{
  products: {
    byId: { /* product entities */ },
    filters: { /* product filtering UI */ },
    isLoading: false,
    error: null
  },
  cart: {
    items: [],
    isCheckingOut: false,
    checkoutError: null
  },
  user: {
    profile: { /* user data */ },
    preferences: { /* settings */ },
    isAuthenticated: false
  }
}
```

**RTK structure:**

```javascript
// features/products/productsSlice.js
export const productsSlice = createSlice({
  name: 'products',
  initialState: {
    byId: {},
    allIds: [],
    filters: { category: null, search: '' },
    isLoading: false,
    error: null,
  },
  reducers: {
    // Product-specific actions
  },
});

// features/cart/cartSlice.js
export const cartSlice = createSlice({
  name: 'cart',
  initialState: {
    items: [],
    isCheckingOut: false,
  },
  reducers: {
    // Cart-specific actions
  },
});

// app/store.js
export const store = configureStore({
  reducer: {
    products: productsSlice.reducer,
    cart: cartSlice.reducer,
    user: userSlice.reducer,
  },
});
```

### Pattern 2: Feature Folder Structure

```
src/
├── app/
│   ├── store.js              # Store configuration
│   └── rootReducer.js        # Root reducer
├── features/
│   ├── products/
│   │   ├── productsSlice.js  # State + reducers
│   │   ├── productsAPI.js    # RTK Query endpoints
│   │   ├── productsSelectors.js
│   │   ├── ProductList.jsx   # Components
│   │   ├── ProductDetail.jsx
│   │   └── index.js          # Public exports
│   ├── cart/
│   │   ├── cartSlice.js
│   │   ├── CartSummary.jsx
│   │   └── index.js
│   └── user/
│       ├── userSlice.js
│       ├── UserProfile.jsx
│       └── index.js
└── common/
    ├── components/           # Shared components
    ├── hooks/                # Shared hooks
    └── utils/                # Utilities
```

### Pattern 3: Cross-Slice Logic with extraReducers

Sometimes one slice needs to respond to actions from another slice:

```javascript
// cartSlice.js
export const cartSlice = createSlice({
  name: 'cart',
  initialState: { items: [] },
  reducers: {
    addToCart: (state, action) => {
      state.items.push(action.payload);
    },
    clearCart: (state) => {
      state.items = [];
    },
  },
});

// orderSlice.js
import { cartSlice } from '../cart/cartSlice';

export const orderSlice = createSlice({
  name: 'orders',
  initialState: { list: [] },
  reducers: {
    orderCreated: (state, action) => {
      state.list.push(action.payload);
    },
  },
  extraReducers: (builder) => {
    // Listen to cart actions
    builder.addCase(cartSlice.actions.clearCart, (state) => {
      console.log('Cart was cleared, might want to log this');
    });
  },
});
```

**Using `extraReducers` with async thunks:**

```javascript
// userSlice.js
export const logout = createAsyncThunk('user/logout', async () => {
  await api.logout();
  return null;
});

export const userSlice = createSlice({
  name: 'user',
  initialState: { profile: null, isAuthenticated: false },
  reducers: {},
  extraReducers: (builder) => {
    builder.addCase(logout.fulfilled, (state) => {
      state.profile = null;
      state.isAuthenticated = false;
    });
  },
});

// cartSlice.js
import { logout } from '../user/userSlice';

export const cartSlice = createSlice({
  name: 'cart',
  initialState: { items: [] },
  reducers: {
    addToCart: (state, action) => {
      state.items.push(action.payload);
    },
  },
  extraReducers: (builder) => {
    // Clear cart when user logs out
    builder.addCase(logout.fulfilled, (state) => {
      state.items = [];
    });
  },
});
```

### Pattern 4: Listener Middleware for Side Effects

Redux Toolkit 1.8+ introduced `createListenerMiddleware`:

```javascript
import { createListenerMiddleware } from '@reduxjs/toolkit';

const listenerMiddleware = createListenerMiddleware();

// Listen for specific actions and run side effects
listenerMiddleware.startListening({
  actionCreator: cartSlice.actions.addToCart,
  effect: async (action, listenerApi) => {
    // Access full API
    const state = listenerApi.getState();
    const dispatch = listenerApi.dispatch;

    // Analytics
    analytics.track('Item Added to Cart', {
      productId: action.payload.id,
      cartTotal: state.cart.items.length,
    });

    // Save to localStorage
    localStorage.setItem('cart', JSON.stringify(state.cart.items));
  },
});

// Pattern matching
listenerMiddleware.startListening({
  predicate: (action, currentState, previousState) => {
    // Listen to any action that changes cart
    return currentState.cart.items.length !== previousState.cart.items.length;
  },
  effect: async (action, listenerApi) => {
    console.log('Cart length changed!');
  },
});

// Add to store
const store = configureStore({
  reducer: rootReducer,
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware().prepend(listenerMiddleware.middleware),
});
```

**Advantages over traditional middleware:**
- Type-safe action matching
- Easier to test (can add/remove listeners dynamically)
- Access to full store API
- Built-in debouncing/throttling support
- Can cancel listeners

---

## Part 16: Redux in Different Environments

### Server-Side Rendering (SSR) with Next.js

```javascript
// store.js
import { configureStore } from '@reduxjs/toolkit';
import { productsApi } from './services/products';

export const makeStore = () =>
  configureStore({
    reducer: {
      [productsApi.reducerPath]: productsApi.reducer,
      // other reducers
    },
    middleware: (getDefaultMiddleware) =>
      getDefaultMiddleware().concat(productsApi.middleware),
  });

// _app.js (Next.js App wrapper)
import { Provider } from 'react-redux';
import { makeStore } from '../store';

function MyApp({ Component, pageProps }) {
  // Create store per request (SSR)
  const store = makeStore();

  return (
    <Provider store={store}>
      <Component {...pageProps} />
    </Provider>
  );
}

// pages/products.js
export async function getServerSideProps() {
  const store = makeStore();

  // Pre-fetch data on server
  await store.dispatch(
    productsApi.endpoints.getProducts.initiate()
  );

  return {
    props: {
      initialReduxState: store.getState(),
    },
  };
}
```

**Key SSR considerations:**

1. **Create fresh store per request**: Prevents state leaking between users
2. **Serialize state carefully**: No functions, class instances, or circular refs
3. **Hydrate client-side**: Match server-rendered state exactly
4. **Handle async data**: Pre-fetch critical data server-side

### React Native

Redux works identically in React Native, but storage differs:

```javascript
// Use AsyncStorage instead of localStorage
import AsyncStorage from '@react-native-async-storage/async-storage';
import { persistStore, persistReducer } from 'redux-persist';

const persistConfig = {
  key: 'root',
  storage: AsyncStorage, // Mobile storage
  whitelist: ['user', 'settings'],
};

const persistedReducer = persistReducer(persistConfig, rootReducer);

export const store = configureStore({
  reducer: persistedReducer,
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware({
      serializableCheck: {
        ignoredActions: ['persist/PERSIST', 'persist/REHYDRATE'],
      },
    }),
});

export const persistor = persistStore(store);
```

---

## Part 17: Debugging and DevTools

### Redux DevTools Deep Dive

**Time-travel debugging:**

```javascript
// Install Redux DevTools Extension
const store = configureStore({
  reducer: rootReducer,
  devTools: {
    name: 'My App',
    trace: true, // Show action stack traces
    traceLimit: 25,
    actionSanitizer: (action) => {
      // Hide sensitive data in DevTools
      if (action.type === 'user/loginSuccess') {
        return {
          ...action,
          payload: {
            ...action.payload,
            password: '***HIDDEN***',
          },
        };
      }
      return action;
    },
    stateSanitizer: (state) => {
      // Hide sensitive state
      return {
        ...state,
        user: {
          ...state.user,
          apiKey: state.user.apiKey ? '***HIDDEN***' : null,
        },
      };
    },
  },
});
```

**Remote Redux DevTools:**

```javascript
// For React Native or remote debugging
import { composeWithDevTools } from 'redux-devtools-extension';

const composeEnhancers = composeWithDevTools({
  realtime: true,
  hostname: 'localhost',
  port: 8000,
});

const store = createStore(rootReducer, composeEnhancers(applyMiddleware(...)));
```

### Custom Dev Middleware

```javascript
const loggerMiddleware = (store) => (next) => (action) => {
  console.group(action.type);
  console.info('dispatching', action);
  
  const result = next(action);
  
  console.log('next state', store.getState());
  console.groupEnd();
  
  return result;
};

// Conditional logging (only in dev)
const conditionalLogger = process.env.NODE_ENV === 'development'
  ? loggerMiddleware
  : (store) => (next) => (action) => next(action);
```

---

## Conclusion: The Philosophy Behind React-Redux

React-Redux bridges two powerful paradigms:

1. **Redux's unidirectional data flow**: Predictable state management
2. **React's component model**: Declarative UI based on state

The library's design reflects deep understanding of both systems:

- **Subscription management**: Ensures top-down rendering consistency
- **Selector optimization**: Prevents unnecessary re-renders
- **Context + hooks**: Modern, ergonomic API
- **Performance by default**: Smart equality checks and memoization

**The mental model:**
- Store is single source of truth
- Actions describe "what happened"
- Reducers specify "how state changes"
- Selectors derive data efficiently
- Components subscribe to precisely what they need

**When React-Redux shines:**
- Large applications with complex state
- Multiple components need same data
- State logic is complex and benefits from Redux DevTools
- Team needs predictable, testable architecture

**When simpler alternatives work:**
- Small apps with minimal shared state
- State is truly local to component trees
- No need for time-travel debugging or middleware
- Context + useReducer suffices

This exploration covered the complete architecture—from low-level store implementation to high-level patterns. Understanding these internals empowers you to:

- Debug effectively when issues arise
- Optimize performance intelligently
- Make informed architectural decisions
- Extend Redux with custom solutions

The beauty of React-Redux is how it abstracts complexity while remaining transparent—every piece serves a purpose, and understanding that purpose makes you a better engineer.