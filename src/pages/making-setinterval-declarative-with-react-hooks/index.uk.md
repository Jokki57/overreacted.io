---
title: Створення декларативного setInterval разом з React Hooks
date: '2019-02-04'
spoiler: Як я навчився не хвилюватися та любити рєфи (refs).
---

Якщо ви працювали з [React Hooks](https://reactjs.org/docs/hooks-intro.html) більш ніж декілька годин, скоріш за все ви зіткнулися з інтригуючою проблемою: використання `setInterval` просто [не працює](https://stackoverflow.com/questions/53024496/state-not-updating-when-using-react-state-hook-within-setinterval) як ви того очікували.

Райан Флоренц [сказав](https://mobile.twitter.com/ryanflorence/status/1088606583637061634):

>У мене було багато людей вказуючих на те, що setInterval з хуками наче якесь яйце на обличчі React (🤷‍♂️ – прим. перекладача)

Чесно кажучи, я вважаю, ці люди у чомусь праві. Спочатку це здається заплутаним.

Але я також зрозумів, що це не є недоліком хуків, а скоріше розбіжність [моделі програмування React](/react-as-a-ui-runtime/) та `setInterval`. Хуки, які більше наближенні до моделі програмування React, аніж класи, роблять цю розбіжність більш помітною.

**_Є_ спосіб змусити їх працювати гарно між собою, але це трохи не так інтуїтивно.**

У цьому пості ми роздивимось _що_ необхідно зробити що інтервали та хуки почали ладити між собою, _чому_ це рішення має сенс, та які *нові* можливості це може дати вам.

-----

**Примітка: цей пост зфокусований на _крайовому випадку_. Навіть якщо API спрощує сотні юзкейсів, обговорення все одно буде сфокусоване на більш складному кейсі.**

Якщо Hooks для вас нові та ви не розумієте про що вся ця метушня, краще читайте [це знайомство](https://medium.com/@dan_abramov/making-sense-of-react-hooks-fdbde8803889) та [документацію](https://reactjs.org/docs/hooks-intro.html). Цей пост передбачає, що ви працювали з хуками хоча б більше годині.

---

## Лише Покажіть Мені Код

Без зайвого галасу, ось лічильник, який збільшується кожної секунди:

```jsx{6-9}
import React, { useState, useEffect, useRef } from 'react';

function Counter() {
  let [count, setCount] = useState(0);

  useInterval(() => {
    // Your custom logic here
    setCount(count + 1);
  }, 1000);

  return <h1>{count}</h1>;
}
```

*(Ось [CodeSandbox демо](https://codesandbox.io/s/105x531vkq).)*

Цей `useInterval` не є вбудованним React-хуком; це [кастомний хук](https://reactjs.org/docs/hooks-custom.html) який я написав:

```jsx
import React, { useState, useEffect, useRef } from 'react';

function useInterval(callback, delay) {
  const savedCallback = useRef();

  // Remember the latest callback.
  useEffect(() => {
    savedCallback.current = callback;
  });

  // Set up the interval.
  useEffect(() => {
    function tick() {
      savedCallback.current();
    }
    if (delay !== null) {
      let id = setInterval(tick, delay);
      return () => clearInterval(id);
    }
  }, [delay]);
}
```

*(Here’s a [CodeSandbox demo](https://codesandbox.io/s/105x531vkq) in case you missed it earlier.)*

**My `useInterval` Hook sets up an interval and clears it after unmounting.** It’s a combo of `setInterval` and `clearInterval` tied to the component lifecycle.

Feel free to copy paste it in your project or put it on npm.

**If you don’t care how this works, you can stop reading now! The rest of the blog post is for folks who are ready to take a deep dive into React Hooks.**

---

## Wait What?! 🤔

I know what you’re thinking:

>Dan, this code doesn’t make any sense. What happened to “Just JavaScript”? Admit that React has jumped the shark with Hooks!

**I thought this too but I changed my mind, and I’m going to change yours.** Before explaining why this code makes sense, I want to show off what it can do.

---

## Why `useInterval()` Is a Better API


To remind you, my `useInterval` Hook accepts a function and a delay:

```jsx
  useInterval(() => {
    // ...
  }, 1000);
```

This looks a lot like `setInterval`:

```jsx
  setInterval(() => {
    // ...
  }, 1000);
```

**So why not just use `setInterval` directly?**

This may not be obvious at first, but the difference between the `setInterval` you know and my `useInterval` Hook is that **its arguments are “dynamic”**.

I’ll illustrate this point with a concrete example.

---

Let’s say we want the interval delay to be adjustable:

![Counter with an input that adjusts the interval delay](./counter_delay.gif)

While you wouldn’t necessarily control the delay with an *input*, adjusting it dynamically can be useful — for example, to poll for some AJAX updates less often while the user has switched to a different tab.

So how would you do this with `setInterval` in a class? I ended up with this:

```jsx{7-26}
class Counter extends React.Component {
  state = {
    count: 0,
    delay: 1000,
  };

  componentDidMount() {
    this.interval = setInterval(this.tick, this.state.delay);
  }

  componentDidUpdate(prevProps, prevState) {
    if (prevState.delay !== this.state.delay) {
      clearInterval(this.interval);
      this.interval = setInterval(this.tick, this.state.delay);
    }
  }

  componentWillUnmount() {
    clearInterval(this.interval);
  }

  tick = () => {
    this.setState({
      count: this.state.count + 1
    });
  }

  handleDelayChange = (e) => {
    this.setState({ delay: Number(e.target.value) });
  }

  render() {
    return (
      <>
        <h1>{this.state.count}</h1>
        <input value={this.state.delay} onChange={this.handleDelayChange} />
      </>
    );
  }
}
```

*(Here’s a [CodeSandbox demo](https://codesandbox.io/s/mz20m600mp).)*

This is not too bad!

What’s the Hook version looking like?

<font size="50">🥁🥁🥁</font>

```jsx{5-8}
function Counter() {
  let [count, setCount] = useState(0);
  let [delay, setDelay] = useState(1000);

  useInterval(() => {
    // Your custom logic here
    setCount(count + 1);
  }, delay);

  function handleDelayChange(e) {
    setDelay(Number(e.target.value));
  }

  return (
    <>
      <h1>{count}</h1>
      <input value={delay} onChange={handleDelayChange} />
    </>
  );
}
```

*(Here’s a [CodeSandbox demo](https://codesandbox.io/s/329jy81rlm).)*

Yeah, *that’s all it takes*.

Unlike the class version, there is no complexity gap for “upgrading” the `useInterval` Hook example to have a dynamically adjusted delay:

```jsx{4,9}
  // Constant delay
  useInterval(() => {
    setCount(count + 1);
  }, 1000);

  // Adjustable delay
  useInterval(() => {
    setCount(count + 1);
  }, delay);
```

When `useInterval` Hook sees a different delay, it sets up the interval again.

**Instead of writing code to *set* and *clear* the interval, I can *declare* an interval with a particular delay — and our `useInterval` Hook makes it happen.**

What if I want to temporarily *pause* my interval? I can do this with state too:

```jsx{6}
  const [delay, setDelay] = useState(1000);
  const [isRunning, setIsRunning] = useState(true);

  useInterval(() => {
    setCount(count + 1);
  }, isRunning ? delay : null);
```

*(Here is a [demo](https://codesandbox.io/s/l240mp2pm7)!)*

This is what gets me excited about Hooks and React all over again. We can wrap the existing imperative APIs and create declarative APIs expressing our intent more closely. Just like with rendering, we can **describe the process at all points in time simultaneously** instead of carefully issuing commands to manipulate it.

---

I hope by this you’re sold on `useInterval()` Hook being a nicer API — at least when we’re doing it from a component.

**But why is using `setInterval()` and `clearInterval()` annoying with Hooks?** Let’s go back to our counter example and try to implement it manually.

---

## First Attempt

I’ll start with a simple example that just renders the initial state:

```jsx
function Counter() {
  const [count, setCount] = useState(0);
  return <h1>{count}</h1>;
}
```

Now I want an interval that increments it every second. It’s a [side effect that needs cleanup](https://reactjs.org/docs/hooks-effect.html#effects-with-cleanup) so I’m going to `useEffect()` and return the cleanup function:

```jsx{4-9}
function Counter() {
  let [count, setCount] = useState(0);

  useEffect(() => {
    let id = setInterval(() => {
      setCount(count + 1);
    }, 1000);
    return () => clearInterval(id);
  });

  return <h1>{count}</h1>;
}
```

*(See the [CodeSandbox demo](https://codesandbox.io/s/7wlxk1k87j).)*

Seems easy enough? This kind of works.

**However, this code has a strange behavior.**

React by default re-applies effects after every render. This is intentional and helps avoid [a whole class of bugs](https://reactjs.org/docs/hooks-effect.html#explanation-why-effects-run-on-each-update) that are present in React class components.

This is usually good because many subscription APIs can happily remove the old and add a new listener at any time. However, `setInterval` isn’t one of them. When we run `clearInterval` and `setInterval`, their timing shifts. If we re-render and re-apply effects too often, the interval never gets a chance to fire!

We can see the bug by re-rendering our component within a *smaller* interval:

```jsx
setInterval(() => {
  // Re-renders and re-applies Counter's effects
  // which in turn causes it to clearInterval()
  // and setInterval() before that interval fires.
  ReactDOM.render(<Counter />, rootElement);
}, 100);
```

*(See a [demo](https://codesandbox.io/s/9j86r218y4) of this bug.)*

---

## Second Attempt

You might know that `useEffect()` lets us [*opt out*](https://reactjs.org/docs/hooks-effect.html#tip-optimizing-performance-by-skipping-effects) of re-applying effects. You can specify a dependency array as a second argument, and React will only re-run the effect if something in that array changes:

```jsx{3}
useEffect(() => {
  document.title = `You clicked ${count} times`;
}, [count]);
```

When we want to *only* run the effect on mount and cleanup on unmount, we can pass an empty `[]` array of dependencies.

However, this is a common source of mistakes if you’re not very familiar with JavaScript closures. We’re going to make this mistake right now! (We’re also building a [lint rule](https://github.com/facebook/react/pull/14636) to surface these bugs early but it’s not quite ready yet.)

In the first attempt, our problem was that re-running the effects caused our timer to get cleared too early. We can try to fix it by never re-running them:

```jsx{9}
function Counter() {
  let [count, setCount] = useState(0);

  useEffect(() => {
    let id = setInterval(() => {
      setCount(count + 1);
    }, 1000);
    return () => clearInterval(id);
  }, []);

  return <h1>{count}</h1>;
}
```

However, now our counter updates to 1 and stays there. ([See the bug in action](https://codesandbox.io/s/jj0mk6y683).)

What happened?!

**The problem is that `useEffect` captures the `count` from the first render.** It is equal to `0`. We never re-apply the effect so the closure in `setInterval` always references the `count` from the first render, and `count + 1` is always `1`. Oops!

**I can hear your teeth grinding. Hooks are so annoying, right?**

[One way](https://codesandbox.io/s/j379jxrzjy) to fix it is to replace `setCount(count + 1)` with the “updater” form like `setCount(c => c + 1)`. It can always reads fresh state for that variable. But this doesn’t help you read the fresh props, for example.

[Another fix](https://codesandbox.io/s/00o9o95jyv) is to [`useReducer()`](https://reactjs.org/docs/hooks-reference.html#usereducer). This approach gives you more flexibility. Inside the reducer, you have the access both to current state and fresh props. The `dispatch` function itself never changes so you can pump data into it from any closure. One limitation of `useReducer()` is that you can’t yet emit side effects in it. (However, you could return new state — triggering some effect.)

**But why is it getting so convoluted?**

---

## The Impedance Mismatch

This term is sometimes thrown around, and [Phil Haack](https://haacked.com/archive/2004/06/15/impedance-mismatch.aspx/) explains it like this:

>One might say Databases are from Mars and Objects are from Venus. Databases do not map naturally to object models. It’s a lot like trying to push the north poles of two magnets together.

Our “impedance mismatch” is not between Databases and Objects. It is between the React programming model and the imperative `setInterval` API.

**A React component may be mounted for a while and go through many different states, but its render result describes *all of them at once.***

```jsx
  // Describes every render
  return <h1>{count}</h1>
```

Hooks let us apply the same declarative approach to effects:

```jsx{4}
  // Describes every interval state
  useInterval(() => {
    setCount(count + 1);
  }, isRunning ? delay : null);
```

We don’t *set* the interval, but specify *whether* it is set and with what delay. Our Hook makes it happen. A continuous process is described in discrete terms.

**By contrast, `setInterval` does not describe a process in time — once you set the interval, you can’t change anything about it except clearing it.**

That’s the mismatch between the React model and the `setInterval` API.

---

Props and state of React components can change. React will re-render them and “forget” everything about the previous render result. It becomes irrelevant.

The `useEffect()` Hook “forgets” the previous render too. It cleans up the last effect and sets up the next effect. The next effect closes over fresh props and state. This is why our [first attempt](https://codesandbox.io/s/7wlxk1k87j) worked for simple cases.

**But `setInterval()` does not “forget”.** It will forever reference the old props and state until you replace it — which you can’t do without resetting the time.

Or wait, can you?

---

## Refs to the Rescue!

The problem boils down to this:

* We do `setInterval(callback1, delay)` with `callback1` from first render.
* We have `callback2` from next render that closes over fresh props and state.
* But we can’t replace an already existing interval without resetting the time!

**So what if we didn’t replace the interval at all, and instead introduced a mutable `savedCallback` variable pointing to the *latest* interval callback?**

Now we can see the solution:

* We `setInterval(fn, delay)` where `fn` calls `savedCallback`.
* Set `savedCallback` to `callback1` after the first render.
* Set `savedCallback` to `callback2` after the next render.
* ???
* PROFIT

This mutable `savedCallback` needs to “persist” across the re-renders. So it can’t be a regular variable. We want something more like an instance field.

[As we can learn from the Hooks FAQ,](https://reactjs.org/docs/hooks-faq.html#is-there-something-like-instance-variables) `useRef()` gives us exactly that:

```jsx
  const savedCallback = useRef();
  // { current: null }
```

*(You might be familiar with [DOM refs](https://reactjs.org/docs/refs-and-the-dom.html) in React. Hooks use the same concept for holding any mutable values. A ref is like a “box” into which you can put anything.)*

`useRef()` returns a plain object with a mutable `current` property that’s shared between renders. We can save the *latest* interval callback into it:

```jsx{8}
  function callback() {
    // Can read fresh props, state, etc.
    setCount(count + 1);
  }

  // After every render, save the latest callback into our ref.
  useEffect(() => {
    savedCallback.current = callback;
  });
```

And then we can read and call it from inside our interval:

```jsx{3,8}
  useEffect(() => {
    function tick() {
      savedCallback.current();
    }

    let id = setInterval(tick, 1000);
    return () => clearInterval(id);
  }, []);
```

Thanks to `[]`, our effect never re-executes, and the interval doesn’t get reset. However, thanks to the `savedCallback` ref, we can always read the callback that we set after the last render, and call it from the interval tick.

Here’s a complete working solution:

```js{10,15}
function Counter() {
  const [count, setCount] = useState(0);
  const savedCallback = useRef();

  function callback() {
    setCount(count + 1);
  }

  useEffect(() => {
    savedCallback.current = callback;
  });

  useEffect(() => {
    function tick() {
      savedCallback.current();
    }

    let id = setInterval(tick, 1000);
    return () => clearInterval(id);
  }, []);

  return <h1>{count}</h1>;
}
```

*(See the [CodeSandbox demo](https://codesandbox.io/s/3499qqr565).)*

---

## Extracting a Hook

Admittedly, the above code can be disorienting. It’s mind-bending to mix the opposite paradigms. There’s also a potential to make a mess with mutable refs.

**I think Hooks provide lower-level primitives than classes — but their beauty is that they enable us to compose and create better declarative abstractions.**

Ideally, I just want to write this:

```js{4-6}
function Counter() {
  const [count, setCount] = useState(0);

  useInterval(() => {
    setCount(count + 1);
  }, 1000);

  return <h1>{count}</h1>;
}
```

I’ll copy and paste the body of my ref mechanism into a custom Hook:

```jsx
function useInterval(callback) {
  const savedCallback = useRef();

  useEffect(() => {
    savedCallback.current = callback;
  });

  useEffect(() => {
    function tick() {
      savedCallback.current();
    }

    let id = setInterval(tick, 1000);
    return () => clearInterval(id);
  }, []);
}
```

Currently, the `1000` delay is hardcoded. I want to make it an argument:

```jsx
function useInterval(callback, delay) {
```

I will use it when I set up the interval:

```jsx
    let id = setInterval(tick, delay);
```

 Now that the `delay` can change between renders, I need to declare it in the dependencies of my interval effect:

```js{8}
  useEffect(() => {
    function tick() {
      savedCallback.current();
    }

    let id = setInterval(tick, delay);
    return () => clearInterval(id);
  }, [delay]);
```

Wait, didn’t we want to avoid resetting the interval effect, and specifically passed `[]` to avoid it? Not quite. We only wanted to avoid resetting it when the *callback* changes. But when the `delay` changes, we *want* to restart the timer!

Let’s check if our code works:

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  useInterval(() => {
    setCount(count + 1);
  }, 1000);

  return <h1>{count}</h1>;
}

function useInterval(callback, delay) {
  const savedCallback = useRef();

  useEffect(() => {
    savedCallback.current = callback;
  });

  useEffect(() => {
    function tick() {
      savedCallback.current();
    }

    let id = setInterval(tick, delay);
    return () => clearInterval(id);
  }, [delay]);
}
```

*(Try it on [CodeSandbox](https://codesandbox.io/s/xvyl15375w).)*

It does! We can now `useInterval()` in any component and not think too much about its implementation details.

## Bonus: Pausing the Interval

Say we want to be able to pause our interval by passing `null` as the `delay`:

```jsx{6}
  const [delay, setDelay] = useState(1000);
  const [isRunning, setIsRunning] = useState(true);

  useInterval(() => {
    setCount(count + 1);
  }, isRunning ? delay : null);
```

How do we implement this? The answer is: by not setting up an interval.

```js{6}
  useEffect(() => {
    function tick() {
      savedCallback.current();
    }

    if (delay !== null) {
      let id = setInterval(tick, delay);
      return () => clearInterval(id);
    }
  }, [delay]);
```

*(See the [CodeSandbox demo](https://codesandbox.io/s/l240mp2pm7).)*

That’s it. This code handles all possible transitions: a change of a delay, pausing, or resuming an interval. The `useEffect()` API asks us to spend more upfront effort to describe the setup and cleanup — but adding new cases is easy.

## Bonus: Fun Demo

This `useInterval()` Hook is really fun to play with. When the side effects are declarative, it’s much easier to orchestrate complex behaviors together.

**For example, we can have a `delay` of one interval be controlled by another:**

![Counter that automatically speeds up](./counter_inception.gif)

```js{10-15}
function Counter() {
  const [delay, setDelay] = useState(1000);
  const [count, setCount] = useState(0);

  // Increment the counter.
  useInterval(() => {
    setCount(count + 1);
  }, delay);

  // Make it faster every second!
  useInterval(() => {
    if (delay > 10) {
      setDelay(delay / 2);
    }
  }, 1000);

  function handleReset() {
    setDelay(1000);
  }

  return (
    <>
      <h1>Counter: {count}</h1>
      <h4>Delay: {delay}</h4>
      <button onClick={handleReset}>
        Reset delay
      </button>
    </>
  );
}
```

*(See the [CodeSandbox demo](https://codesandbox.io/s/znr418qp13)!)*

## Closing Thoughts

Hooks take some getting used to — and *especially* at the boundary of imperative and declarative code. You can create powerful declarative abstractions with them like [React Spring](http://react-spring.surge.sh/hooks) but they can definitely get on your nerves sometimes.

This is an early time for Hooks, and there are definitely still patterns we need to work out and compare. Don’t rush to adopt Hooks if you’re used to following well-known “best practices”. There’s still a lot to try and discover.

I hope this post helps you understand the common pitfalls related to using APIs like `setInterval()` with Hooks, the patterns that can help you overcome them, and the sweet fruit of creating more expressive declarative APIs on top of them.
