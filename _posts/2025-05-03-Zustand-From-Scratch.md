---
layout: post
title: "How Zustand Library Works"
---
Our team uses Spring Boot for the back-end and mainly Angular for its front-end. However, we had an opportunity to write a new internal tool in React, and that experience made me appreciate the advantages of dynamic typing and functional language features in JavaScript that have no true substitutes in statically typed languages like Java. To handle the state management, we searched for something both easy to use and modern and ultimately chose Zustand library. I am very interested in understanding how things work under the hood, so I searched for the very first published version of the library, tried to understand it and in the way learn much both about the language itself but also React libary. What follows is my write-up both for myself, but also for others want to know more about how such libraries are written.

Firstly, let's forget about React and write a simple pub/sub library that gives the users the following contract:

- We have to write a function that accepts two methods (magically implemented and provided by the library, for now), *getState* and *merge*. They are justified in the following way: to modify the state, we have to both know the current state, hence *getState*, but also specify the changes, hence *merge*, which takes a new object literal and merges it into the current state using the spread operator. So, we are expected to write a method like this:
  ```js
  const counterStore = (getState, merge) => ({
    operations: 0,
    count: 0,
    add() {
      const { operations, count } = getState();
      merge({ operations: operations + 1, count: count + 1 });
    },
    subtract() {
      const { operations, count } = getState();
      merge({ operations: operations + 1, count: count - 1 });
    }
  });
  ```
- The library should accept such description and provide the implementations of *getState* and *merge*. The letter should also notify all the listeners that something has changed and therefore we have to provide the functionality for both subscribing and unsubscribing to those changes. This very simple library could be written like this:
  ```js
  function createStore(initFn) {
    let listeners = [];
    let state;
  
    const getState = () => state;
    const merge = patch => {
      state = { ...state, ...patch };
      listeners.forEach(fn => fn(state));
    };
  
    // initialize your state/store object
    state = initFn(getState, merge);
  
    return listener => {
        listeners.push(listener);
        // return an unsubscribe handle
        return () => {
          listeners = listeners.filter(l => l !== listener);
        };
    };
  }
  ```


We now have basic ingredients: a store for internal state, a way to update/patch that state and a listener system. Zustand build on that foundation and adapts it for React, using Hooks. Let's go and do just that. The first version we will accomplish will contain a subtle bug, which we will then try to fix. That bug is connected with closures and how react store states and has lot's of teaching purpose. These are steps we are going to take:

- a given React component might not need all the data in the given store, so we give the user possibility to specify that selector. Also we might need to only listen to the changes in the story only if the specific dependencies are changed. This way we gain in efficiency. To sum up, our Hook should accept two parameters, *selector* function and *dependencies* list.
- using **useState** hook and *selector* function provided, we describe a resulting state slice for the component.
- using **useEffect**, we create a listener to the store that gets notified when something changes and updates the component state accordingly. Inside **useEffect** hook, we have to also clean up the subscription

Those steps can be incorporated in the following way to improve the library. It should now support React:
```js
  function createStore(initFn) {
    let listeners = [];
    let state;
  
    const getState = () => state;
    const merge = patch => {
      state = { ...state, ...patch };
      listeners.forEach(fn => fn(state));
    };
  
    // initialize your state/store object
    state = initFn(getState, merge);
  
    return (selector, dependencies) => {
    	// to make changes in selected slice,
      // we have to use provided methods in the store we wrote ourselves.
      // Hence const below
    	const selected = selector(state)
      // Here, selected could be a method.
      // The way React works, if it sees the method inside useState hook
      // it evaluates and its result becomes the value of the state.
      // We don't want that. Hence () => selected
    	const [slice, setSlice] = useState(() => selected)
    	
    	useEffect(() => {
    		const listener = () => {
    			const newSlice = selector(state);
    			setSlice(() => newSlice);
    		}
    		
    		listeners.push(listener);
    		// unsubscribe method given to useEffect
    		return () => {
    			listeners = listeners.filter(l => l !== listener)
    		}
    	}, dependencies || [selector])
    	return selected;
    }
  }
```

To make this more efficient, we only re-render the component when the new slice differs from the previous one. We do this with a shallow diff—comparing only top-level keys. That means their values must reference the **same** object in memory for each matching key in two object literals. It’s stricter than necessary (you could have the same data in different memory locations), but deep diffs are expensive. This strikes a practical balance between accuracy and performance. The shallow diff is implemented in Zustand library in a very slick way:
```js
if (typeof slice === 'object' and !Array.isArray(selected)) {
  newSlice = Object.entries(slice).reduce(
    (acc, [key, value]) => (slice[key] !== value ? { ...acc, [key]: value} : acc),
    slice
  )
}
if (slice !== newSlice) {
  setSlice(() => newSlice);
}
```
The idea is simple: we use the previous slice as our base, then for each top-level key we check whether the new value points to the **same** object in memory. As soon as one doesn’t match, we spread the base into a fresh object—giving us a new reference—so that the final slice !== selected check will catch the change. Very clever if you ask me! So the almost final version is this (it has a subtle bug, can you notice it?)

```js
  function createStore(initFn) {
    let listeners = [];
    let state;
  
    const getState = () => state;
    const merge = patch => {
      state = { ...state, ...patch };
      listeners.forEach(fn => fn(state));
    };
  
    // initialize your state/store object
    state = initFn(getState, merge);
  
    return (selector, dependencies) => {
    	const selected = selector(state)
    	const [slice, setSlice] = useState(() => selected)
    	
    	useEffect(() => {
          const listener = () => {
            let newSlice = selector(state);
            if (typeof newSlice === 'object' && !Array.isArray(newSlice)) {
              newSlice = Object.entries(newSlice).reduce(
              (acc, [key, value]) => (slice[key] !== value ? { ...acc, [key]: value} : acc),
                slice
              )
            }
            if (slice !== newSlice) {
              setSlice(() => newSlice);
            }
          }
      		
          listeners.push(listener);
          return () => {
            listeners = listeners.filter(l => l !== listener)
          }
        }, dependencies || [selector])
      	return selected;
    }
  }
```

Now, what is the bug we have been talking about? The listener we changed looks at the previous slice and re-renders the component only if the new slice is different. However, it still points to the slice which was up-to-date only when the useEffect hook was first run. As states in React are immutable, the data this particular slice was looking at was never updated even though the state changed. To remedy this situation, we add a new useState hook that checks if the slice is ever changed and accordingly updates the also added reference created using **useRef**. This combination is pretty common in development and solves stale closure problems in React. So the final code has the following form:

```js
  function createStore(initFn) {
    let listeners = [];
    let state;
  
    const getState = () => state;
    const merge = patch => {
      state = { ...state, ...patch };
      listeners.forEach(fn => fn(state));
    };
  
    // initialize your state/store object
    state = initFn(getState, merge);
  
    return (selector, dependencies) => {
      const selected = selector(state)
      const [slice, setSlice] = useState(() => selected)
      const sliceRef = useRef()
      useEffect(() => (sliceRef.current = slice), [slice]);
    	useEffect(() => {
          const listener = () => {
            let newSlice = selector(state);
            if (typeof newSlice === 'object' && !Array.isArray(newSlice)) {
              newSlice = Object.entries(newSlice).reduce(
              (acc, [key, value]) => (sliceRef.current[key] !== value ? { ...acc, [key]: value} : acc),
                sliceRef.current
              )
            }
            if (sliceRef.current !== newSlice) {
              setSlice(() => newSlice);
            }
          }
      		
          listeners.push(listener);
          return () => {
            listeners = listeners.filter(l => l !== listener)
          }
        }, dependencies || [selector])
      	return selected;
    }
  }
```
