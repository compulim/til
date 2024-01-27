# Today I learnt

Point form for speedy writing. 80% correct at the time of writing. Just remind myself what I did. Timestamp is not accurate as I may recall things and writing something I learn some time ago.

## 2024-01-27

## Reading docs about Azure Container Apps

- Azure Container Apps is more-or-less a managed version of Azure Kubernetes Service
   - [Excerpt](https://learn.microsoft.com/en-us/azure/azure-functions/functions-container-apps-hosting): "Container Apps uses the power of the underlying Azure Kubernetes Service (AKS) while removing the complexity of having to work with Kubernetes APIs."
- Azure Container Apps is all about, quickly spin up to handle load (scaler includes HTTP, pull-based events, cron), then slowly reduce replica to zero
- Events must be pull-based ([KEDA](https://keda.sh/))
   - Number of blobs, but not changes to blobs
   - Queue is okay, but not Cosmos DB changes
   - Kubernetes style of handling events
   - Event Grid does not support pull for events from Azure services
      - Event Grid can route events to Azure Queue storage or Azure Event Hubs
- Jobs does not support Dapr (microservices orchestration) and no ingress
   - No HTTP, but KEDA
- Can run infinite/continuous process (minimum replica = 1)
- Can deploy from private registry
- Can run Azure Functions by hosting the function on a Docker image, with limited triggers: HTTP, Queue storage, Service Bus, Event Hubs, Kafka. No Cosmos DB and not feature on par

## A bit hands-on Azure Container Apps Job

- Azure Container Apps Job did not emit log properly to log workspace
- Once a job trigger is configured, it is not possible to reconfigure it
- Takes about 15 seconds to boot and run
- One job resource = one job + one trigger

## 2024-01-22

## Iterator/iterable/generator

- `Iterator` is `next`, optional `return` and `throw`
- `Iterable` is `[Symbol.iterator](): { return { next, return, throw } satisfies Iterator<T>; }`
   - `Iterator` and `Iterable` is interchangeable
- `IterableIterator` = `Iterable` + `Iterator` = `{ [Symbol.iterator]() } & { next(), return(), throw() }`
- `Generator` is `IterableIterator` with required `return` and `throw`, i.e. all featured and iterable
- I/O
   - Input: `Array<T>.from(Iterable<T>)`
   - Output: `new Map<T>().values instanceof IterableIterator<T>`
- Siblings
   - `Observable` (`complete`/`error`/`next`) vs. `Generator` (`next`/`return`/`throw`)
      - `Generator` is suspended/on-demand/pull-based, it will not run in background and do not need a worker to drive its data
      - `Observable` is event-based, it requires a worker to drive its data
   - `Observable` vs. `EventTarget`
      - `EventTarget` is real time. If no one listen to event, dispatched events will be lost. `Observable` buffer it until subscriber ready for it
      - When subscribing to an `EventTarget`, it does not know about it. `Observable` knows when someone subscribes to it and normally start a new instance/operation
      - `EventTarget` is singleton (one in its world), and `Observable` is single instance (many in its world)
   - `Observable` (`complete`/`error`/`next`) vs. [`ReadableStream`](https://developer.mozilla.org/en-US/docs/Web/API/ReadableStream) (`close`/`enqueue`/`error`)
      - `Observable` is push-based (must have a worker), `ReadableStream` can be either or both push-based and pull-based (not having a worker)
      - When implementing pull-based `ReadableStream`, it has watermark and can be automatically corked by not pulling if watermark is high
      - `ReadableStream` can easily tee and perform transformation (N:M transformation)
   - `ReadableStream` vs. `Generator`
      - `Generator` is easier to write thanks to `function* ()` syntactic sugars
         - Say, "after generator is completely iterated, run some logics" is not trivial to build using `ReadableStream` but `Generator`

### For-loop with iterable/generator

- Using for-loop with generator will lose some ability: no return value and exception thrown cannot be caught in generator
   - `try`-`finally` in generator will still work
   - `yield` in `finally` may not work because exception thrown cannot be caught in generator, and `yield` in `finally` will simply stop `finally`
   - Maybe refrain from `yield` in `finally`
- Iterable should generally use with for-loop, which don't call `next` with a value or expose `return`, thus it is `Iterator<T>` instead of `Iterable<T, TReturn, TNext>`
   - However, generator natively support return/throw and can become iterable, for-loop-ing a generator may miss some values

Read about [Generator return on MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Generator/return).

### Converting `Iterator<T>` into `IterableIterator<T>`

```ts
export default class IterableIteratorFromIterator<T, TReturn, TNext> implements IterableIterator<T> {
  constructor(iterator: Iterator<T, TReturn, TNext>) {
    this.next = iterator.next.bind(iterator);
    this.return = iterator.return && iterator.return.bind(iterator);
    this.throw = iterator.throw && iterator.throw.bind(iterator);
  }

  [Symbol.iterator](): IterableIterator<T> {
    return this;
  }

  next: () => IteratorResult<T>;
  return?(value?: TReturn): IteratorResult<T, TReturn>;
  throw?(e?: any): IteratorResult<T, TReturn>;
}
```

### Converting `ReadableStream<T>` into `AsyncIterableIterator<T>`

```ts
export default async function* <T>(readableStream: ReadableStream<T>): AsyncIterableIterator<T> & AsyncIterator<T, T> {
  const reader = readableStream.getReader();

  for (;;) {
    const response = await reader.read();

    if (response.done) {
      return response.value;
    }

    yield response.value;
  }
}
```

## 2024-01-10

### Valibot

- `union()` means `T | U` and `intersect()` means `T & U`
- `isoTimestamp()` is not quite ISO yet, some improvements could be done
- Validating all type of inputs is nice, because people may not use TypeScript to write their integration code
- `parse()` is great at pumping what's wrong, not great at visualizing the wrongs for human

## 2024-01-09

[Interesting read on focus indicator](https://www.sarasoueidan.com/blog/focus-indicators/) around buttons.

## 2024-01-08

### Azure Functions: Sub-orchestration vs. activity

- Sub-orchestration primarily reduce replay cost
   - After completed, the "heap" in sub-orchestration will be discarded
   - Can be use to reduce replay time in parent orchestration and minimize point of failures
- Orchestration replay time is largely based on 2 things
   - High impact: Number of activities executed (total size of activity output)
   - Medium impact: Working set size (size of each activity output)
   - Each activity output is saved into a TGZ file, many activities executed means downloading many TGZ files, means higher chance of failures
      - When an action start, if it failed to download history, it will be timed out after 5 minutes
   - Consider [extending orchestration session](https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-azure-storage-provider#extended-sessions) (lingering orchestration) to reduce replay boot time
- No complex logics in sub-orchestration
   - Orchestration replay means it promotes deterministic, which also means idempotency (use cache, minimize refetch)
   - Don't mess with `isReplaying`, doesn't worth the complexity
- Activity should only run for a short period of time (< 5 minutes)
   - Sub-orchestration is the pattern for running longer jobs
- [Some tips here](https://spzsource.github.io/azure/2020/03/07/durable-function-performance-dependence-on-different-payload-size.html)
- Large working set (> 64 KB)
   - Large working set is saved to storage blob, instead of storage queue
   - Waking up orchestrator is literally queueing in storage queue
   - Rehydrating large working set in orchestration is prone to failure (task being cancelled)
   - If possible, keep large working set in activity and don't output it back to orchestration

### Dealing with 429 Too Many Requests

Consider using Service Bus to queue HTTP calls that might return as 429. If 429 is received, requeue the message with a schedule based on 429 cooldown period.

## 2024-01-07

### NoSQL: Data modelling

- When partition key is same, we could potentially keep documents in the same container
- Documents should not keep in the same container when:
   - Change feed is required for certain type of documents

## 2023-12-26

### Authentication in Azure Functions (a.k.a. EasyAuth)

- Not very useful in Azure Functions alone, may work better in Web Apps or Static Web Apps, or Azure Front Door
   - The cookie will save on the Azure Functions domain and it requires 3P cookie which is deprecating
- Don't work on local Azure Functions Emulator
- Probably originated from Azure Mobile App Service (Project Zumo)
- Read this, https://learn.microsoft.com/en-us/azure/app-service/configure-authentication-customize-sign-in-out
- To authenticate (via MSAL so I can auth on another domain):
   1. Use MSAL with scopes of `openid`
   2. Grab the `idToken` from MSAL call
   3. Send it to /.auth/login/aad with `{ "access_token": idToken }`
   4. Should return with `{ "authenticationToken" }`, this is a local token
   5. On every API calls, add `X-ZUMO-AUTH` with the content of `authenticationToken`
- It works on many scenarios except Server-Sent Events and Web Socket, which headers cannot be altered
   - I remember new `fetch()` could now build a Web Socket and passing headers, but I could not find it now
   - [`@microsoft/fetch-event-source`](https://npmjs.com/package/@microsoft/fetch-event-source) is outdated and don't like its API signature

### Azure Cosmos DB change feed triggering Azure Functions

- Better queue it up to Azure Service Bus
- Otherwise, when Azure Functions fail, you have no way (or too painful) to retry

### Azure Cosmos DB patch operation

- Consider you are working on `{ tags: ['area-ui', 'bug'] }`
- While it is easy to add stuff to `tags` without concerning about concurrency, like: `{ op: 'add', path: '/tags/-', value: 'area-accessibility' }`, `/-` means append
- It is difficult to remove stuff because you need index, like: `{ op: 'remove', path: '/tags/2' }`
- For concurrency requirements, maybe use another document

### Azure Cosmos DB data modelling

These are fun reads:

- https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/model-partition-example
- https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/modeling-data

Fun part: a container can store 2+ types of documents, and they just need to agree on partition key.

For the "add/remove tags" concurrency problem in the "patch operation" section above, could try out these 2 documents in the same container:

```js
{
  "id": "u-00001",
  "userId": "u-00001", // this is partition key
  "type": "user",
  "name": "John Doe"
}
```

```js
{
  "id": "t-00001",
  "userId": "u-00001", // this is partition key
  "type": "tag",
  "tag": "bug"
}
```

Then, when querying the container by `userId`, we grab all documents (of type `user` and `tag`). Then we can turn them back into object model.

```ts
database.container('user').items.readAll({ partitionKey: userId }).fetchAll();
```

#### In other words

If we want to model this object in database:

```js
{
  id: 'b-00001',
  description: 'Button not working',
  tags: ['bugs', 'area-ui']
}
```

Traditionally, you will write this in relational database:

| ID | Description | Tags |
| - | - | - |
| `b-00001` | Button not working | `bugs,area-ui` |

Then, you would normalize it into 2 tables:

| ID | Description |
| - | - |
| `b-00001` | Button not working |

| ID | Bug ID (FK) | Tag |
| - | - | - |
| `t-00001` | `b-00001` | `bugs` |
| `t-00001` | `b-00001` | `area-ui` |

In relational database, you should do 2 queries to get the result for object model.

But in document DB, you would do:

| ID | Bug ID (PK) | Type | Description | Tag |
| - | - | - | - | - |
| `b-00001` | `b-00001` | `bug` | Button not working | |
| `t-00001` | `b-00001` | `tag` | | `bugs` |
| `t-00002` | `b-00001` | `tag` | | `area-ui` |

And you get everything in a single query, while data is normalized.

### JavaScript async iterator

```ts
const iterate = () => ({
  [Symbol.asyncIterator]: () => ({
    next(): IteratorResult<number> {
      // ...
    }
  }
};
```

When returning `{ done: true, value: 123 }`, the value `123` will probably lost if iteration is done through for-loop.

### Azure Functions: Server-Sent Events

tl;dr not supported, it just buffer up before sending the response body.

Read about [MDN Server-Sent Events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events).

Code snippet tested and the result is buffered.

```ts
app.http('...', {
  async handler() {
    const body: AsyncIterator<Uint8Array> = build(); // Will build a SSE output stream

    return { body, contentType: 'text/event-stream' };
  }
}
```

### MSAL

- MSAL is great if you use it the way it's intended
- You can't read "access token" when using `@azure/msal-browser` because you shouldn't access sensitive stuff in browser
- Acquire token by redirect is nice, because it auto-remove `#code=`

## 2023-12-25

### Azure Durable Functions

- Orchestration through `yield` by replaying
   - Clever, `yield` is provingly good for orchestration and pause/resume, see `redux-saga`
   - Replay is mostly good, except some limitations because replay is not 100% exact
- Don't be lazy: type out activity input/output via [`valibot`](https://npmjs.com/package/valibot)

### Type template for [`valibot`](https://npmjs.com/package/valibot)

```ts
const activityInput = () => object({
  id: string(),
  name: string()
});

export default activityInput;

export const parseActivityInput = (data: unknown) => Object.freeze(parse(activityInput(), data)); // Or deep freeze

export type ActivityInput = ReadonlyDeep<Output<ReturnType<typeof activityInput>>>;
```

### [`react-window`](https://npmjs.com/package/react-window)

A package to provide virtualized scrolling to anything. Another Fluent UI Contrib to integrate `<DataGrid>` with it.

- Requires JavaScript to set `width`/`height` of the container which hold the virtualized viewport
- Can't <kbd>CTRL</kbd> + <kbd>F</kbd> to find stuff (via Fluent UI Contrib)
- Maybe just use [CSS `content-visibility: auto`](https://developer.mozilla.org/en-US/docs/Web/CSS/content-visibility) is good enough (no Safari support)

### Why I still don't like Fluent UI v9

- `<DataGrid>` has serious performance issues:
   - Why hovering on 2,000 rows is slow?
   - Why sorting 2,000 rows takes seconds?
   - Why I need to copy `<DataGrid>` template and impossible to recite?
   - Why the sample/scaffold/template is not using/encouraging `useCallback` at all?
      - Why web component devs are not familiar with `useCallback`/`useMemo`?
   - Maybe it's opinionated, but my opinions aren't about their opinions, it's about their facts
- UI is good for desktop, not great for mobile (too small, etc.)
   - If you want "write once", it is still okay

## 2023-12-24

### Azure Cosmos DB: throttling request unit

Throttling on client-side (Azure Functions-side) using [`limiter`](https://npmjs.com/package/limiter) package.

```ts
const limiter = new RateLimiter({ interval: 'second', tokensPerInterval: 50 });

for (const id of idsToRead) {
  await limiter.removeTokens(1); // Assume minimum request charge is 1

  const result = await database.container('user').item(id).read();

  limiter.removeTokens(result.requestCharge - 1); // No need to await, we already spent that charge. Next caller will pause if throttled

  yield result.resource;
}
```

Throttling through Azure Service Bus scheduled enqueue is not great:

- When some operations fell behind, the scheduled time could be passed for more transactions
- A boosting effect will occur (many transactions will be executed at the same time)
- The more transactions to execute, the more likely to get 429, the more likely to fail on the Service Bus processing, the more likely to retry, and more transactions will run again

### Happy Hacking Keyboard

- It is easy to learn <kbd>CTRL</kbd> and <kbd>CAPSLOCK</kbd> swap
- It is easy to learn no <kbd>F1</kbd>-<kbd>F12</kbd> keys
- You don't move your palm at all and you can be very focused on typing and thinking
- Reduce 80% mouse usage
- To do <kbd>CTRL</kbd> + <kbd>DELETE</kbd> on a normal keyboard:
   - <kbd>Fn</kbd> + <kbd>CTRL</kbd> + <kbd>`</kbd> won't work
   - <kbd>CTRL</kbd> + <kbd>Fn</kbd> + <kbd>`</kbd> will work

## 2023-12

### Syncing Pi-Hole configuration online

- Not easy at the current moment
- `/etc` files of Pi-Hole can be huge files (about 1 GB). And some configuration stored inside their DB files (binary)
- Some OSS projects attempts to sync. But I think it's non-trivial
- `rcp` is still good
- Editing Pi-Hole configuration online is not very helpful because settings store inside DB files
