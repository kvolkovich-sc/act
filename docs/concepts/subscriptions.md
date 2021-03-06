# Subscriptions

Subscriptions are a way to get values in your application from sources other
than regular DOM node events (that you define directly in your views), like
window events, timers, ajax calls and web sockets.

They exist in a pretty [similar way in Elm](http://elm-lang.org/blog/farewell-to-frp)
and they are analogous to creating any event source in [Cycle.js](http://cycle.js.org/),
just the workflow is very different.

Before understanding subscriptions, you should first take a look at
[actions](/actions.md) and [signals](/signals.md). Once you do that you will
already know a bit about signals and their sources and processes. And
subscriptions are nothing but signals. So if we wanna listen to the window's
scroll, we should use the `fromEvent` source. Creating a source goes like this:

```js
const source = fromEvent(window, 'scroll')
```

Note all DOM events have `start` and `stop` method, so when you pass then to your
process, you have to call `start`. It exists because for some events, Act needs to
stop the source, a DOM element that is removed from the DOM, for instance, so Act
will call the method `stop` behind the scenes.

Now, there is a built in helper for processing the scroll position, called,
conveniently, `scroll`. So you can just attach your source to it:

```js
scroll(source.start())
```

When you create subscription you don't need to call the returning function
yourself, you just pass it to Act and it will manage. The way to do that is to
pass to Act's `main` method an object with the type of the action you want to
trigger as the key. Here's the full code:

```js
const subscriptions = {
  scroll: scroll(fromEvent(window, 'scroll').start())
}

main(view, { model, reducer, subscriptions })
```

With this code, Act will get the scroll position and send it to your reducer
with an action like `{ type: 'scroll', payload: 123 }` every time the scroll
position changes.

In real life, you'll probably only case about certain specific scroll
positions. To accomplish that you can simply put this logic in your reducer,
but the encouraged way is to further compose your scroll process:

```js
const moved = pipe(
  scroll,
  map((scrollPosition) => scrollPosition > 0)
  distinct
)
```

So in this process we first get the scroll position and turn it into a boolean,
if the user has scrolled (>0). We finally call `distinct`, else we would get
the `true` value everytime the user scrolled any pixel. `distinct` will filter
our values to send them through only when they are different than the previous
one (false becomes true and vice versa).

Another common usage here is to to throtlle (a.k.a. debouce) this event, so you
don't get too many rerenders which is memory and time consuming. In order to
emit the value only once in every second, just do:

```js
const throttledScroll = pipe(
  throttle(1000),
  scroll
)
```

The cool thing about this is that these functions can be used not only for
scroll but for any sort of subscription you need.

One final and relevant comment is that Ramda's functions are pretty handy for
creating your own processes. Take a look at some ideas:

| Ex | Transforms |
|--------|---------|
| map(head) | [1, 2, 3] => 1<br>'abc' => 'a'|
| map(tail) | [1, 2, 3] => [2, 3]<br>'abc' => 'bc'|
| map(prop('name')) | {name: 'Spinoza'} => 'Spinoza'<br />{} => undefined |
| map(propOr('Descartes', 'name')) | {name: 'Spinoza'} => 'Spinoza'<br />{} => 'Descartes' |
| filter(equals(5)) | 5 => 5<br /> 6 => ()|
| filter(propEq('id', 7)) | {id: 7} => {id: 7}<br />{id: 8} => ()|
| filter(contains(100)) | [99, 100] => [99, 100]<br /> [99, 101] => ()|
| fold(add, 0) | [1,2,3] => 6 |

### Subscription helpers

Many use cases are too common for people to have to reinvent the wheel every time.
If you followed the examples above, take a look at the `subscriptions` folder, and
you'll find plenty of ready made helpers. Here are two of them:

```js
const subscriptions = { breakpoint, scroll }
```

The `breakpoint` subscription will give you a breakpoint for the current screen
width, every time it changes. You can provide custom breakpoint config using
`brekpointWith`. And `scroll` will give you the current scroll position, throttled to
a reasonable amount of time. To know more check the documentation on subscriptions.

Note that the examples above used the `scroll` helper from processes, so you
can compose your own, and here `scroll` comes from the subscription helpers, a
more high level library that abstracts the whole signal - source and process.

### Subscriptions with side effects

Not all subscriptions are supposed to change our state. Some may only execute
some side effect, or even do both. In this case, you can use the following
syntax:

```
const subscriptions = [
  [sideEffect, subscription]
]
```

This will call `sideEffect` with the history and the subscription value every
time there's a new value produced. Note this is analogous to actions with
signals and side effects, even the syntax is the same.

If you want a quick way to know when to use each, look into the
[diagrams](../real_world/diagrams.md) doc.
