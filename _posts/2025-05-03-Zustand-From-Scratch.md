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
