# ff.js - function-based state manager

**experimental**

> “We keep moving forward, opening new doors, and doing new things
> because we’re curious and curiosity keeps leading us down new
> paths.” Walt Disney

![seemingly fractal](https://raw.githubusercontent.com/amirouche/ff.js/master/fabian-fauth-RiWvn39cZSQ-unsplash.jpg)

## Motivation

- Keep It Simply Simple,

- Avoid classes when possible,

- Do not support Server-Side-Rendering,

- Implement (fractal?) **M**odel-**V**iew-**C**ontroller pattern,

- Avoid reduxjs actions indirection, because I did not find a
  compeling reason to define a giant enumeration,

- Neither does the various (hackish?) way to define the reducer.

## History

`ff.js` started as learning project to understand how
[redux](https://github.com/reduxjs/react-redux/) and
[elm](https://elm-lang.org/) work with [scheme programming
language](http://scheme-lang.com/). Basicaly, it was a note taking
exercices in the form of lot of parenthesis. I started with
[biwascheme](https://www.biwascheme.org/), an interpreter of scheme
written in JavaScript.  Because [browsers don't implement tail call
optimisation (TCO)](https://kangax.github.io/compat-table/es6/), yet,
I was hiting recursion depth limit. Nonetheless, it was possible to
make the classic
[todomvc](https://github.com/amirouche/scheme-todomvc) exercice.

The story about `ff.js` in Scheme continue
[here](http://scheme-lang.com/cons/).

Then I challenged myself with the task to port that code to
JavaScript.  To implement asynchronous requests without callbacks my
scheme code was relying on
[call/cc](https://en.wikipedia.org/wiki/Call-with-current-continuation)
which JavaScript does not have. Instead, modern browsers implement
[`async` / `await` which is readily available in most modern
browsers](https://caniuse.com/#feat=async-functions).  The base
library ported to JavaScript, I discovered through various tests a
race condition that was happening when forms were automatically filled
by the browser. My design being somewhat like reduxjs I looked at how
reduxjs solved the issue and re-discovered that my controllers were
asynchronous (hence subject to race conditions because of concurrency)
whereas the equivalent in redux (called reducers) were synchronous. I
fixed that bug using what I call a `transformer` that is
event-handlers (namely controllers) return a synchronous function that
should return the new state. See how redux workaround that issue
https://redux.js.org/advanced/async-actions#async-action-creators

I know there is still a source of race condition when the code has
concurrency involved (for instance via `setTimeout`). Right now,
`ff.js` does not make it easy to do such things hence avoids those
race conditions.

The result 350 lines of JavaScript that make a state manager for
reactjs that use lot of functions (that might return (async?)
functions).

## Getting started

This assumes you have nodejs and npm installed. If it's not the case
follow [nvm tutorial](https://github.com/creationix/nvm#installation).

1. Install `create-react-app` with `npm install -g create-react-app`
1. Start the project with the following command: `npm init react-app my-app`
1. Change directory `cd ./my-app/`
1. Install dependencies of `ff.js` with `npm install seamless-immutable`
1. Change directory `cd src`
1. Download `ff.js` with `wget https://github.com/amirouche/ff.js/raw/master/ff.js`
1. Change back directory `cd ..`
1. Run the app `npm start`

## Documentation

### Application flow

The application flow is defined as the pattern that the code follows
during the life of the application. There is an entry point called
**route init** then it goes through a some kind of loop a certain
number of times and then it exits that loop restarting from the entry
point probably another **route init**. It can be represented as
follow:

![ff single route diagram](ff-single-route-flow.png)

Multiple turn in the same loop means that the user interact with the
application on the same route ie. the same URL's path.

Changing route, means the application flow will move to a new loop.
That happens in the controller. So the overall life of the application
look something like the following:

![ff multiple route diagram](ff-multiple-route-flow.png)

Maybe those diagrams are helpful, but they are only a second hand view
on what the code actually looks like. Indeed, the diagram do not tell
that there is multiple controller functions that each controller is
attached to a DOM event via ReactJS. Last but not least, the diagram
fail to give a clue about what data is passed around.

An immutable application state called `model` is passed between the
various (maybe async) functions.  `render` through view functions have
the responsability to return reactjs components based on the current
`model`. Controllers have the responsability to *transform* that
`model` into a new model that will be used to re-render the
application. Hence the loop thing. When you want to break the loop
aka. change route aka. change path, you can call `redirect` in the
controller.

The following paragraph try to help with that.

### `Router` class

The `Router` class allows to configure the different routes that the
application will use.  That is also the first object you will use when
defining an application. It has a single public method called
`Router.append`.

TODO: `Router.append` could be a function :8ball:

### `Router.append(pattern, init, view)`

`Router.append` allows to add a route to the application.

### `Router`'s `pattern`

`Router.append` takes a `pattern` as first argument. A pattern is an
URL path that might contains parameters enclosed in curly
braces. Parameters will match any component of a path.

**E.g.**:

Without parameters:

```javascript
router = new ff.Router()
router.append('/home/', init, view)
```

With a parameter:

```javascript
router = new ff.Router()
router.append('/post/{slug}/', init, view)
```

### `Router`'s `init`

A route `init` function has the responsability to setup ready the
application's `model` to render the current route. It's an
asynchronous function that returns a transformer:

- `init` functions are asynchronous it means that they are defined
  with `async function` prelude and that they can fetch data from a
  server, which might come handy.

- `init` functions return a *transformer*. A *transformer* is a pure
  function of the `app` and `model` that must return the new `model`.

What you must remember is that `init` functions are executed in two
steps:

1. The first step is asynchronous where the application can fetch
   data from a server,

2. And a second step that is pure (hence synchronous) that must return
   the new application state namely `model`.

**E.g.**:

To take an example from `ff.js` itself here is the definition of `ff.routeClean`:

```javascript
let routeClean = async function(app, model) {
    return (app, model) => clean(model);
}
```

What you can take away from this example:

1. Route `init` functions are `async` function
2. They take as arguments `app` and `model` they look like `async
   function(app, model)`
3. They must return a pure function of `app` and `model`, something
   like `async function(app, model) { return (app, model) => model}`

A more realistic example is a route that is defined with a parameter:

```javascript
router.append('/post/{slug}/', init, view)
```

In that case, the `init` function will take an extra argument called
`params` that is a JavaScript Object where every parameters is bound
to its value in the current URL. Here is an example use of that:

```javascript
let init = async function(app, model, params) {
    let post = await fetchPost(params.slug);
    return (app, model) => model.set('post', ff.Immutable(post));
}
```

### `Router`'s `view`

A route's `view` must return a reactjs component. It takes as
arguments the current application's `model` and `makeController`
aliased `mc`.

**E.g.**::

Here is very simple view function:

```javascript
let view = function(model, mc) {
    return <h1>Hello, world!</h1>;
}
```

`model` is not very interesting it is the usual stuff.

`makeController` aka. `mc` is more interesting.  Simply said, `mc`
will bind event handlers to the current application so that they
execute in the context of the application. That is `mc` will pass (or
if you prefer *inject* via a closure) the current application `model`
to the event handler.

Event handlers must return a *transformer*.

At the end of the day, you must:

1. call `mc` on every event handler **e.g.** `<button
   onClick={mc(onClick)}>increment</button>`
2. Remember the correct imbrication of `async function` and non-`async
   function` and their arguments.

**E.g.**:

Here is simple example for a clicker game:

```javascript
let onClick = function (app, model) {  // first `app` and `model` as argument,
                                       // not async. That is the closure added by mc.

    return async function(event) {  // then the *event handler*
                                    // as an `async function`
                                    // with DOM `event` as argument.
                                    // Do not forget the RETURN keyword.

        return function (app, model) {  // at last the *transformer*
                                        // pure synchronous function
                                        // with `app` and `model` as arguments

            return model.set('counter', model.counter + 1);
        }
    }
}
```

Here is more realistic example:

```javascript
let onCreate = function(app, model) {
    return async function(event) {
        let data = {
            title: model["title"],
            description: model["description"],
        }
        let response = await ff.post("/api/project/new/", data);
        if(response.status === 200) {
            let json = await response.json();
            let uid = json['uid']
            return await ff.redirect(app, ff.clean(model), "/project/" + uid + "/");
        } else {
            return (app, model) => ff.clean(model);
        }
    }
}
```

## In the wild

- [lasthope](https://github.com/amirouche/lastho.pe) conversational
  search engine with a python backend.
