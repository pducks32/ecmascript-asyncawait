# Async Functions for  ECMAScript

The introduction of Promises and Generators in ECMAScript presents an opportunity to dramatically improve the language-level model for writing asynchronous code in ECMAScript.  

A similar proposal was made with [Defered Functions](http://wiki.ecmascript.org/doku.php?id=strawman:deferred_functions) during ES6 discussions.  The proposal here supports the same use cases, using similar or the same syntax, but directly building upon generators and promises instead of defining custom mechanisms.

## Status of this Proposal

This proposal was accepted into stage 1 ("Proposal") of the ECMASCript 7 [spec  process](https://docs.google.com/document/d/1QbEE0BsO4lvl7NFTn5WXWeiEIBfaVUF7Dk0hpPpPDzU) in January 2014.  See discussion [here](http://esdiscuss.org/notes/2014-01-30#async-await).

This proposal is implemented in a [branch of regenerator](https://github.com/facebook/regenerator/pull/101) on top of Esprima, which can compile ES5 code containing `async` and `await` down to vanilla ES5 to run in existing browsers and runtimes.

This repo contains a complete example using a large number of the features of the proposal.  To run this example:

```Shell
npm install		
regenerator -r server.asyncawait.js | node
```


## Example

Take the following example, first written using Promises.  This code chains a set of animations on an element, stopping when there is an exception in an animation, and returning the value produced by the final successfully executed animation.

```JavaScript
function chainAnimationsPromise(elem, animations) {
    var ret = null;
    var p = currentPromise;
    for(var anim of animations) {
        p = p.then(function(val) {
            ret = val;
            return anim(elem);
        })
    }
    return p.catch(function(e) {
        /* ignore and keep going */
    }).then(function() {
        return ret;
    });
}
```

Already with promises, the code is much improved from a straight callback style, where this sort of looping and exception handling is challenging.

[Task.js](http://taskjs.org/) and similar libraries offer a way to use generators to further simplify the code maintaining the same meaning:

```JavaScript
function chainAnimationsGenerator(elem, animations) {
    return spawn(function*() {
        var ret = null;
        try {
            for(var anim of animations) {
                ret = yield anim(elem);
            }
        } catch(e) { /* ignore and keep going */ }
        return ret;
    });
}
```

This is a marked improvement.  All of the promise boilerplate above and beyond the semantic content of the code is removed, and the body of the inner function represents user intent.  However, there is an outer layer of boilerplate to wrap the code in an additional generator function and pass it to a library to convert to a promise.  This layer needs to be repeated in every function that uses this mechanism to produce a promise.  This is so common in typical async Javascript code, that there is value in removing the need for the remaining boilerplate.

With async functions, all the remaining boiler plate is removed, leaving only the semantically meaningful code in the program text:

```JavaScript
async function chainAnimationsAsync(elem, animations) {
    var ret = null;
    try {
        for(var anim of animations) {
            ret = await anim(elem);
        }
    } catch(e) { /* ignore and keep going */ }
    return ret;
}
```

This is morally similar to generators, which are a function form that produces a Generator object.  This new async function form produces a Promise object.

## Details

Async functions are a thin sugar over generators and a `spawn` function which converts generators into promise objects.  The internal generator object is never exposed directly to user code, so the rewrite below can be optimized significantly.

### Rewrite

```
async function <name>?<argumentlist><body>

=>

function <name>?<argumentlist>{ return spawn(function*() <body>); }
```

### Spawning

The `spawn` used in the above desugaring is a call to the following algorithm.  This algorithm does not need to be exposed directly as an API to user code, it is part of the semantics of async functions.

```JavaScript
function spawn(genF) {
    return new Promise(function(resolve, reject) {
        var gen = genF();
        function step(nextF) {
            var next;
            try {
                next = nextF();
            } catch(e) {
                // finished with failure, reject the promise
                reject(e); 
                return;
            }
            if(next.done) {
                // finished with success, resolve the promise
                resolve(next.value);
                return;
            } 
            // not finished, chain off the yielded promise and `step` again
            Promise.cast(next.value).then(function(v) {
                step(function() { return gen.next(v); });      
            }, function(e) {
                step(function() { return gen.throw(e); });
            });
        }
        step(function() { return gen.next(undefined); });
    });
}
```

### Syntax

The set of syntax forms are the same as for generators.

```bnf
AsyncFunctionDeclaration :
    async [no LineTerminator here] function BindingIdentifier ( FormalParameters ) { FunctionBody }

AsyncFunctionExpression :
    async [no LineTerminator here] function BindingIdentifier? ( FormalParameters ) { FunctionBody }

AsyncMethod :
    async PropertyName (StrictFormalParameters)  { FunctionBody }

AsyncArrowFunction :
    async [no LineTerminator here] ArrowParameters [no LineTerminator here] => ConciseBody

Declaration :
    ...
    AsyncFunctionDeclaration

PrimaryExpression :
    ...
    AsyncFunctionExpression

MethodDefinition :
    ...
    AsyncMethod

AssignmentExpression :
    ...
    AsyncArrowFunction

UnaryExpression :
    ...
    await [Lexical goal InputElementRegExp] UnaryExpression


Note:  await would only be legal inside an Async body.  
       This could use similar formalism to ES6 parameterized grammar.
```

### await* and parallelism

In generators, both `yield` and `yield*` can be used.  In async functions, only `await` is allowed.  The direct analogue of `yield*` does not make sense in async functions because it would need to repeatedly await the inner operation, but does not know what value to pass into each await (for `yield*`, it just passes in undefined because iterators do not accept incoming values).

It has been suggested that the syntax could be reused for different semantics - sugar for Promise.all.  This would accept a value that is an array of Promises, and would (asynchronously) return an array of values returned by the promises.  This is expected to be one of the most common Promise-related oprerations that would not yet have syntax sugar after the core of this proposal is available. 

This would allow, for example:

```JavaScript
async function getData() {
  var items = await fetchAsync('http://example.com/users');
  return await* items.map(async(item) => {
    return {
      title: item.title, 
      img: (await fetchAsync(item.userDataUrl)).img
    }
  });
}
```

### Awaiting Non-Promise

When the value passed to `await` is a Promise, the completion of the async function is scheduled on completion of the Promise.  For non-promises, behaviour aligns with Promise conversation rules according to the proposed semantic polyfill.

### Surface syntax
Instead of `async function`/`await`, the following are options:
- `function^`/`await`
- `function!`/`yield`
- `function!`/`await`
- `function^`/`yield`

### Notes on Types
For generators, an `Iterable<T>` is always returned, and the type of each yield argument must be `T`.  Return should not be passed any argument.

For async functions, a `Promise<T>` is returned, and the type of return expressions must be `T`.  Yield's arguments are `any`.
