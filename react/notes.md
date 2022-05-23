#### sources:

[Dan Abramov's blog: Overreacted](https://overreacted.io/a-complete-guide-to-useeffect/)

# Dependency Triage

_Exhaustive deps warning ... are you lying about your dependencies?_

### Why you should fix this

> "if you specify deps, **all values from inside your component that are used by the effect must be there.** Including props, state, functions — anything in your component. [Though,] sometimes when you do that, it causes a problem. For example, maybe you see an infinite refetching loop, or a socket is recreated too often. **The solution to that problem is _NOT_ to remove a dependency.**" - Dan Abramov

When you omit values from your effect's dependency array you open the door to lifecycle bugs. React needs to know when things change, it uses that knowledge to trigger re-renders and cleanups. Don't lie about your dependencies, fix them!

## Props/State Triage

> "Instead of reading the state inside an effect, dispatch an action that encodes the information about what happened. This allows our effect to stay decoupled from the state. Our effect doesn’t care _how_ we update the state, it just tells us about what happened." - Dan Abramov

### How you can fix this

option 1: **Include _all_ of the values inside the component that are used inside the effect.** -- ideal
option 2: **Reduce the amount of dependencies.** -- using `useReducer` or "functional updater form."

### Using `useReducer`

- This hook lets you decouple expressing the "actions" that happen in your component from _how_ the component state updates in response to them. You can now abstract the stateful values into the reducer and simply leave `dispatch` (optional since React knows it is static) within the `useEffect` deps.

  - Instead of reading the state _inside_ an effect, it dispatches an _action_ that encodes the information about what _happened_. This allows our effect to stay **decoupled** from the reducer's state. Our effect doesn't care _how_ we updated the state, it just tells us about what _happened_ and **the reducer centralizes the update logic.**

### Using the "Functional updater form"

> `setCount(c => c + 1)`

- Your component has the information derived from the previous render, you can leverage that to update that state, albeit in a pretty limited manner.
- You're essentially telling React to increment the state, -- whatever it is right now. It doesn't need to know the current `count` in state, React _already_ knows it, just increment it whatever it may be.

#### 🚀 Examples:

<details>
    <summary>useReducer</summary>

#### Before (with state)

```jsx
useEffect(() => {
  const id = setInterval(() => {
    setCount((c) => c + step);
  }, 1000);
  return () => clearInterval(id);
}, [step]);
```

#### After

```jsx
// reducer file
const initialState = {
  count: 0,
  step: 1,
};

function reducer(state, action) {
  const { count, step } = state;
  if (action.type === "tick") {
    return { count: count + step, step };
  } else if (action.type === "step") {
    return { count, step: action.step };
  } else {
    throw new Error();
  }
}

// ====

// component file
const [state, dispatch] = useReducer(reducer, initialState);
const { count, step } = state;

useEffect(() => {
  const id = setInterval(() => {
    dispatch({ type: "tick" }); // Instead of setCount(c => c + step);
  }, 1000);
  return () => clearInterval(id);
}, [dispatch]);
```

</details>

<details>
    <summary>Functional updater form</summary>

#### Before (with state)

```jsx
useEffect(() => {
  const id = setInterval(() => {
    setCount(count + 1);
  }, 1000);
  return () => clearInterval(id);
}, [count]);
```

#### After

```jsx
useEffect(() => {
  const id = setInterval(() => {
    setCount((c) => c + 1);
  }, 1000);
  return () => clearInterval(id);
  // no deps!
}, []);
```

</details>

## Functions Triage

> "A function inside a component changes between _every_ render." - Dan Abramov

You don't want to lie about your deps and you also want to avoid bugs. What steps can you take to prevent unnecessary re-renders when a `useEffect` has a function as a dependency?

### Function is used within _one_ Effect

- If used in one `useEffect` hook, just include the function within the `useEffect`'s scope

### Function is used within _multiple_ Effects

- ⚠️ **Hoist it outside of the component** ⚠️
  - **ONLY:** If the function does not use _anything_ from the component's scope: props, state, etc.
  - Does not need to be specified in the dependency array because it's not in the render scope and can't be effected by the data flow. It can't _accidentally_ depend on props or state.
- **Wrap it with a `useCallback` hook within the component.**
  - This adds another layer of dependency checks and won't trigger the `useEffect` deps unless the `useCallback`'s dep itself is triggered by a change.
  - If you need to use props or state in the component, simply add them as dependencies within this `useCallback`.
  - A function wrapped in `useCallback` in a parent component, when passed into a child as a prop, the child will remain synchronized with the parent and won't re-render unless the `useCallback` dep values change.

#### 🚀 Examples:

<details>
    <summary>Scoped</summary>

```jsx
function SearchResults() {
  const [query, setQuery] = useState("react");

  useEffect(() => {
    // yay, I live inside the useEffect and 'query' is a dep!
    function getFetchUrl() {
      return "https://hn.algolia.com/api/v1/search?query=" + query;
    }

    async function fetchData() {
      const result = await axios(getFetchUrl());
      setData(result.data);
    }

    fetchData();
  }, [query]); // ✅ Deps are OK

  // ...
}
```

</details>

<details>
    <summary>Hoisting</summary>

#### Before

```jsx
// component
function SearchResults() {
  // 🔴 Re-triggers all effects on every render
  function getFetchUrl(query) {
    return "https://hn.algolia.com/api/v1/search?query=" + query;
  }

  useEffect(() => {
    const url = getFetchUrl("react");
    // ... Fetch data and do something ...
  }, [getFetchUrl]); // 🚧 Deps are correct but they change too often

  useEffect(() => {
    const url = getFetchUrl("redux");
    // ... Fetch data and do something ...
  }, [getFetchUrl]); // 🚧 Deps are correct but they change too often

  // ...
}
```

#### After

```jsx
// ✅ Not affected by the data flow
function getFetchUrl(query) {
  return "https://hn.algolia.com/api/v1/search?query=" + query;
}

// component
function SearchResults() {
  useEffect(() => {
    const url = getFetchUrl("react");
    // ... Fetch data and do something ...
  }, []); // ✅ Deps are OK

  useEffect(() => {
    const url = getFetchUrl("redux");
    // ... Fetch data and do something ...
  }, []); // ✅ Deps are OK

  // ...
}
```

</details>

<details>
    <summary>useCallBack</summary>

#### Not using component state/props

```jsx
function SearchResults() {
  // ✅ Preserves identity when its own deps are the same
  const getFetchUrl = useCallback((query) => {
    return "https://hn.algolia.com/api/v1/search?query=" + query;
  }, []); // ✅ Callback deps are OK

  useEffect(() => {
    const url = getFetchUrl("react");
    // ... Fetch data and do something ...
  }, [getFetchUrl]); // ✅ Effect deps are OK

  useEffect(() => {
    const url = getFetchUrl("redux");
    // ... Fetch data and do something ...
  }, [getFetchUrl]); // ✅ Effect deps are OK

  // ...
}
```

#### Using component state/props

```jsx
function SearchResults() {
  // We now have state that will be used in the getFetchUrl function
  const [query, setQuery] = useState("react");

  // ✅ Preserves identity until query changes
  const getFetchUrl = useCallback(() => {
    return "https://hn.algolia.com/api/v1/search?query=" + query;
  }, [query]); // ✅ Callback deps are OK

  useEffect(() => {
    const url = getFetchUrl();
    // ... Fetch data and do something ...
  }, [getFetchUrl]); // ✅ Effect deps are OK

  // ...
}
```

</details>

---

### Questions

#### Swimming against the Tide

1. Where does **every** function inside of a component render get its props and state? What is this an example of?
1. If you want to read the _latest_, rather than the captured value, inside a callback defined within an effect, what is the easiest way to accomplish this?
1. Why is using the `useRef` considered "swimming against the tide" in React?
1. What's a risk with using `useRef`?
1. What's a key difference between React classes and hooks in regards to rendering timeout callbacks? In this regard, how can you replicate class behaviors with hooks?

#### Cleaning up effects

1. When a component renders, what information does it receive?
1. How can a component cleanup previous code, wouldn't it only have the current props and data to work with?
1. When we cleanup an effect, does the cleanup happen in the current render or within the subsequent render? Why?
1. Does React run effects before or after letting the browser paint? Does behavior also include cleanups?
1. What is a typical use case for effect cleanups?
1. quiz: If a component effect has a prop of 10 and the same effect has a cleanup for cleaning up that prop. When the component re-renders with a new prop of 20, what is the rendering cycle of the new render?

### Answers

#### Swimming against the Tide

1. **Every** function inside the component render (including event handlers, effects, timeouts or API calls inside them) captures the props and state of the render call that defined it.
1. `useRef` allows you to capture the latest value in a (timeout) callback as opposed to the value at the time of invocation.
1. When you're trying to use _future_ props or state from a function in a _past_ render, while sometimes necessary, the code will look less "clean" due to breaking out of the paradigm.
1. "Unlike with captured props and state, you don’t have any guarantees that reading `latestCount.current` would give you the same value in any particular callback. By definition, you can mutate it any time. This is why it’s not a default, and you have to opt into that."
1. With classes, if you console log a stateful value or prop with a timeout, the logs will all display the _latest_ value. With Hooks, each log will display the value present during its specific render. If you want to replicate the class behavior, the `useRef` hook used inside the effect will display the latest value.

#### Cleaning up effects

1. "Every function inside the component render (including event handlers, effects, timeouts or API calls inside them) **captures the props and state of the render call that defined it.**"
1. **React hooks** (not classes) use closures [see above answer], when the component renders, it has the closed over values form the previous render. For optimization purposes, the process of cleaning up the previous render's closed over values occurs _after_ the browser paints.
1. The cleanup for an effect occurs after the following render paints. This makes your app _faster_ as most effects don't need to block screen updates.
1. React **only** runs effects **after** letting the browser paint. Effect cleanup is also delayed.
1. Typically, effect cleanups are used to "undo" an effect, ie closing a subscription event.
1. quiz: React renders the UI for 20. The browser paints (we see 20 on the screen). React cleans up the effect for 10. React runs the effect for 20. Notice the rendering and UI happen before the effect processes fire.
