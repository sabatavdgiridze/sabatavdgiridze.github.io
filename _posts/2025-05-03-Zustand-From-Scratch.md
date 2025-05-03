---
layout: post
title: "How Zustand Library Works"
---
Our team uses Spring Boot for the back-end and mainly Angular for its front-end. However, we had an opportunity to write a new internal tool in React, and that experience made me appreciate the advantages of dynamic typing and functional language features in JavaScript that have no true substitutes in statically typed languages like Java. To handle the state management, we searched for something both easy to use and modern and ultimately chose Zustand library. I am very interested in understanding how things work under the hood, so I searched for the very first published version of the library, tried to understand it and in the way learn much both about the language itself but also React libary. What follows is my write-up both for myself, but also for others want to know more about how such libraries are written.

Firstly, let's forget about React and write a simple pub/sub library that gives the users the following contract:

- We have to write a function that accepts two methods (magically implemented and provided by the library, for now), *getState* and *merge*. They are justified in the following way: to modify the state, we have to both know the current state, hence *getState*, but also specify the changes, hence *merge*, which takes a new object literal and merges it into the current state using the spread operator. So, we are expected to write a method like this:
  ```js
  const counterStore = (getState, merge) => {
  	return {
  		operations: 0,
  		count: 0,
  		add() {
  			merge({operations: getState().operations + 1, count: getState().count + 1});
  		},
  		subtract() {
  			merge({operations: getState().operations + 1, count: getState().count - 1});
  		}
  	};
  }
  ```
