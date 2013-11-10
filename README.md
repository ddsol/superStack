SuperStack node.js continuation stacks.
=======================================

Rationale
---------

In node.js we do a lot of callback hell things. We have all kinds of ways of making that work for us.

But when an error happens:

```
Error: Oh noes!
    at null._onTimeout (C:\superStack\test.js:28:9)
    at Timer.listOnTimeout (timers.js:107:15)
```

Okay. Not very helpful. Who called setTimeout? You may be able to find out by looking at the code at the call site of the last function. Then again, you may not.

In node.js, we use callbacks instead of blocking waits. Ideally, instead of

```JavaScript
...before...
setTimeout(function(){
  ...after...
},200);
```

we would like

```JavaScript
...before...
await delay(200);
...after...
```

but we can't do that (yet). With gen-run and such, you can get a long way.

Solution
--------

SuperStack can help with the error problem:

```JavaScript
var w=require('superstack'); //get the wrapper

w.deepTrace=true;
Error.traceKind='tree';

function sleep(msecs,cb){
  setTimeout(w(cb,1,'sleep'),msecs);
}

sleep(100,function wake1(){
  sleep(200,function wake2(){
    throw new Error('Oh noes!');
  });
})
```

Result:

```
Error: Oh noes!
    at wake2 (C:\superStack\test.js:28:11)
    via sleep
    at wake1 (C:\\superStack\test.js:27:3)
    via sleep
    at Object.<anonymous> (C:\superStack\test.js:26:1)
    at Module._compile (module.js:449:26)
    at Object.Module._extensions..js (module.js:467:10)
    at Module.load (module.js:349:32)
    at Function.Module._load (module.js:305:12)
    at Function.Module.runMain (module.js:490:10)
    at startup (node.js:121:16)
    at node.js:761:3
```

Now we have a complete stack trace.

Really, what's going on is that the stack gets saved somewhere. Then when the callback is called, it's retrieved. Also, anything before the current point in the stack is stored on the stack as a stack branch.
When an error occurs, it's caught and augmented with the new stuff. It only works in V8. Don't use it in the browser unless you know it's V8.

Note that it's possible to branch the stack in weird ways:

```JavaScript
function sleep(msecs,cb){
  setTimeout(w(cb,0,'sleep'),msecs);
}

function sleep2(msecs,cb){
  setTimeout(function(){
    sleep(msecs,cb);
  },msecs);
}

sleep2(100,w(function(){
  throw new Error('Oh noes!');
}));
```

Result:

```
Error: Oh noes!
    at C:\superStack\test.js:17:9
    via timer event
      via timer event
      at timeout handler (C:\superStack\test.js:12:5)
    at Object.<anonymous> (C:\superStack\test.js:16:12)
    at Module._compile (module.js:449:26)
    at Object.Module._extensions..js (module.js:467:10)
    at Module.load (module.js:349:32)
    at Function.Module._load (module.js:305:12)
    at Function.Module.runMain (module.js:490:10)
    at startup (node.js:121:16)
    at node.js:761:3
```

That is because here both branches are tracked though callback wrappers, and each contributes a virtual stack.

Synopsis
--------

```JavaScript
var wrapCb=require('superstack');

//Whenever you write code that 'spans the event loop', wrap it:
function tick(callback){
  process.nextTick(w(callback));
}

//Anyone calling 'tick' will use the superStack

//To turn off enhanced superStack (defaults to true):
Error.deepTrace=false;

//To select tree traces:
Error.traceKind='tree';

//To select normal traces:
Error.traceKind='normal';

//To select only continuable traces:
Error.traceKind='continuable';

//You can clean error traces of internal API calls. Of course, only do this with well tested code where the traces no longer shows relevant information.
//The parameter is an array, and you can remove items with stackArray.slice(index,count); You cannot return a new array (only in-place changes).
wrapCb.clean=function(stackArray){
  for (var i=stackArray.length-1;i>=0;i--){ //Make sure to go backwards: We're deleting items.
    if (/mylib.js/.exec(stackArray[i])) stackArray.slice(i,1);
  }
}

//usage:

//Note that named function expressions show up better in the stack. There are issues with named function expressions, but not in v8.
process.nextTick(wrapCb(function tickHandler(){
  throw new Error('Oh noes!');
}));

//Also works with continuables (error is first param, callback is last param):
require('fs').stat("/doesn't exist",wrapCb(function statCb(err,item){
  if (err) return console.log(err.stack);
  console.log('Got:',item);
}));
```

I assume here you're importing superstack.js as wrapCb. You can use any name you like, of course, but it wraps callbacks, so a name that relects that seems appropriate.

The parameters for wrapCb are:
wrapCb(callback,context,preRemovalDepth,postRemovalDepth,viaName);

* callback: The callback you want to wrap.
* context [optional]: The 'this' parameter passed to the callback. If you leave it out, the normal 'this' parameter will be passed on. Cannot be a number or a string.
* preRemovalDepth [optional]: How many items of the end of the stack at wrap-time to leave off the top of the trace. Internal details of api functions can be left out this way.
* postRemovalDepth [optional]: How many items of the end of the stack at callback-time to leave off the top of the trace. Internal details of api functions can be left out this way. You cannot omit preRemovalDepth while specifying this. How would superstack know the difference?
* viaName [optional]: The name reported in the stack as the 'via' route of how we jumped the event loop. If left out, some heuristic will try to name it (e.g. "via network event")

Keep in mind that the stack is captured at the time of the wrapCb call.
This means you can't pre-wrap your callbacks, unless you want to leave out internal details.

Please note that this is a work in progress and that weird output is certainly possible. Also, I cannot guarantee that it won't produce memory leaks. Apparently, holding on to errors can drive the GC crazy. The errors contain references to actual execution contexts and these in turn can reference the errors. When this happens, the GC has trouble detecting that there are no more references and leaves it in memory. I don't know if superstack is affected. If it is, let me know, and I'll see if there's something that can be done (explicit delete somewhere, maybe?)

Stability Status
----------------

Besides the fact that it may melt your CPU, randomly change data, make obscene calls to your mother-in-law and in general is very unstable, superStack is completely *ready for all production environments*.

License
-------

SuperStack  copyright (c) 2013 by ddsol

Permission is hereby granted, free of charge, to any person obtaining a copy of
this software and associated documentation files (the "Software"), to deal in
the Software without restriction, including without limitation the rights to
use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies
of the Software, and to permit persons to whom the Software is furnished to do
so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.

In other words, do what you want with it, but don't blame me if stuff starts exploding.
