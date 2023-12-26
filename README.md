# Today I learnt

Point form for speedy writing. 80% correct at the time of writing. Just remind myself what I did.

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
  "userId": "u-0001", // this is partition key
  "type": "user",
  "name": "John Doe"
}
```

```js
{
  "id": "t-00001",
  "userId": "u-0001", // this is partition key
  "type": "tag",
  "tag": "bug"
}
```

Then, when querying the container by `userId`, we grab all documents (of type `user` and `tag`). Then we can turn them back into object model.

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
