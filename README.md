# Rowan

A lightweight async middleware library.  

![584023-200](https://cloud.githubusercontent.com/assets/3584509/21929203/1ffa1db6-d987-11e6-8e07-77a6131097af.png)

[![NPM version][npm-image]][npm-url]
[![NPM downloads][npm-downloads]][npm-url]
[![Travis Status][travis-image]][travis-url]
[![codecov](https://codecov.io/gh/MeirionHughes/@taktikorg/mollitia-dolore/branch/master/graph/badge.svg)](https://codecov.io/gh/MeirionHughes/@taktikorg/mollitia-dolore)

## Usage

Rowan can be used to build asynchronous middleware-style control-flow and error-handling, with particular focus on providing a rich typescript experience. 

Create an instance of the Rowan class (or derivation) and call `use` with a middleware function

```ts
import {Rowan} from '@taktikorg/mollitia-dolore';

// Create a (derived) app
const app = new Rowan();

// Add middleware and handlers
app.use(async (ctx) => {
  console.log(`foo: ${ctx.foo}`);
});

```

Once the middleware is all setup you call `process` and pass along the context. 


```ts
// Use it 
await app.execute({ foo: "bar!" });
```

... which in this example would output to console: 

>foo: bar!

## Processors
Processors are either a `Handler<Ctx>`,  `AutoHandler<Ctx>` or `Middleware<Ctx>` type signature. 

* *Handler* is a *two*-parameter function that will be given the  `ctx` and `next` callback and should return a Promise. You are required to call `next` if you wish processing to continue to the next middleware processors in the chain. 

```ts
app.use(async (ctx, next) => {
  ctx["start"] = Date.now();
  await next();
  ctx["finish"] = Date.now();
});
```


* *AutoHandler* is a *one*-parameter function that will be given the `ctx` object. The next processor in the chain will automatically be called for you, unless you throw an Error. 

```ts
app.use(async (ctx) => {
  ctx.data = JSON.parse(ctx.raw);
});
```

* *Middleware* is a object containing a method `process` that will be called with *two*-parameters:  `ctx` and `next`. It is expected that `process` will return a `Promise<void>`. 

```ts
app.use({
  async process(ctx, next){
    await next();
    consol.log("Complete");
  }
});
```

## Helpers

### after-if

calls next and if the predicate returns true executes its middleware

```ts
let foo = new Rowan();

foo.use(new AfterIf(
  async (ctx) => ctx.valid, [
  async (ctx) => {
    console.log("valid message: ", ctx.msg);
  }
]));

foo.use(async (ctx) => {
  console.log("validate...")
  if (ctx.msg && ctx.msg.length > 5) {
    ctx.valid = true
  }
})

async function main() {
  await foo.process({ msg: "hello" });
  await foo.process({ msg: "hello world" });
}

main().catch(console.log);
```

outputs: 

```
validate...
validate...
valid message:  hello world
```

### after

calls next first, then executes its own middleware afterwards

```ts
let foo = new Rowan();

foo.use(new After([
  async (ctx) => {
    console.log(ctx.output);
  }
]));

foo.use(async (ctx) => {
  console.log("processing...")
  ctx.output = ctx.msg;
});

async function main() {
  await foo.process({ msg: "hello" });
  await foo.process({ msg: "hello world" });
}

main().catch(console.log);
```
outputs: 

```
processing...
hello
processing...
hello world
```

### catch

wraps its _own_ middleware with a try...catch

```ts
foo.use(
  new Catch(
    async (err, ctx) => {
      console.log("caught: ", err.message);
    },
    new Rowan()
      .use(
        async (ctx) => {
          if (ctx != "foo") {
            throw Error("ctx must be 'foo'");
          }
        })
      .use({
        meta: { name: "Moo" },
        async process(ctx, next) {
          console.log("Moo!");
          return next();
        }
      })
  ));

async function main() {
  await foo.process('foo');
  await foo.process('bar');
}

main().catch(console.log);
```

outputs:

```
Moo!
caught: ctx must be 'foo'
```

### if

```ts
let foo = new Rowan<string>();

foo.use(
  new If(
    async (ctx: string) => {
      return ctx.startsWith("foo");
    },
    [async (ctx) => {
      console.log("IF...", ctx);
    }],
    /** terminate if predicate() == true */
    true, 
  )
);

foo.use(async (ctx) => {
  console.log("Else...", ctx);
})

async function main() {  
  await foo.process('foo');
  await foo.process('foobar');
  await foo.process('bar');
}

main().catch(console.log);
```

outputs: 

```
IF... foo
IF... foobar
Else... bar

```

## Tools

### Rowan.hierarchy()

used to build a meta hierarchy from processors that have a `middleware` field defined. 

```ts
let foo = new Rowan(undefined, { name: "FOO" });
let bar = new Rowan();

bar.meta.name = "Bar";

bar.use((ctx, next) => {
  console.log("boo1:", ctx);
  return next();
}, { name: "Boo1" });

bar.use(Object.assign((ctx, next) => {
  console.log("boo2:", ctx);
  return next();
}, { meta: { name: "Boo2" } }));

bar.use({
  meta: { name: "Boo3" },
  middleware: [{
    meta: { name: "Custom" },
    process(x, n) { console.log("Custom:", x); return n() }
  }],
  process: function (ctx, next) {
    console.log("Boo3:", ctx);
    return Rowan.process(this.middleware, ctx, next);
  }
});

foo.use(bar);

console.log(JSON.stringify(Rowan.hierarchy(foo), null, 2));
```
outputs: 

```json
{
  "meta": {
    "name": "FOO"
  },
  "children": [
    {
      "meta": {
        "name": "Bar"
      },
      "children": [
        {
          "meta": {
            "name": "Boo1"
          }
        },
        {
          "meta": {
            "name": "Boo2"
          }
        },
        {
          "meta": {
            "name": "Boo3"
          },
          "children": [
            {
              "meta": {
                "name": "Custom"
              }
            }
          ]
        }
      ]
    }
  ]
}
```
## Advanced

### Rowan.process()

executes and chains a sequence of `Middleware`, setting up the `next` callback for each. 

```ts

async function main(next: ()=>Promise<void>) {
  Rowan.process(
    [{
      async process(ctx, next) {
        console.log("first");
        return next();
      }
    },
    {
      async process(ctx, next) {
        console.log("second");
        return next();
      }
    }],
    {
      msg: "hello"
    },
    //... optional next
    next
  )
}
main(async () => {
  console.log("END")
}).catch(console.log);
```
outputs: 
```
first
second
END
```

### Rowan.convertToMiddleware()

used interally to convert supported `Handler` types into valid `Middleware`. 

```ts
Rowan.convertToMiddleware(async (ctx)=>{}, {name: "foo"});
```
results in: 
```
{ 
  meta: { name: 'foo' }, 
  process: [Function] 
}
```

## Build

```
npm install
npm test
```

there is an `example.ts` that you can run with ts-node

```
ts-node example
```

## Credits
"Rowan" Icon courtesy of [The Noun Project](https://thenounproject.com/), by [ludmil](https://thenounproject.com/Maludk), under [CC 3.0](http://creativecommons.org/licenses/by/3.0/us/)

[npm-url]: https://npmjs.org/package/@taktikorg/mollitia-dolore
[npm-image]: https://img.shields.io/npm/v/@taktikorg/mollitia-dolore.svg
[npm-downloads]: https://img.shields.io/npm/dm/@taktikorg/mollitia-dolore.svg
[travis-url]: https://travis-ci.org/MeirionHughes/@taktikorg/mollitia-dolore
[travis-image]: https://img.shields.io/travis/MeirionHughes/@taktikorg/mollitia-dolore/master.svg