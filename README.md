# Today I learnt

## 2023-12-26

### Authentication in Azure Functions (a.k.a. EasyAuth)

- Not very useful in Azure Functions alone, may work better in Web Apps or Static Web Apps, or Azure Front Door
   - The cookie will save on the Azure Functions domain and it requires 3P cookie which is deprecating
- Don't work on local Azure Functions Emulator
- Probably originated from Azure Mobile App Service (Project Zumo)
- Read this, https://learn.microsoft.com/en-us/azure/app-service/configure-authentication-customize-sign-in-out
- To authenticate (via MSAL so I can auth on another domain):
   1. Use MSAL with scopes of "openid"
   2. Grab the `idToken`
   3. Send it to /.auth/login/aad with `{ "access_token": idToken }`
   4. Should return with `{ "authenticationToken" }`, this is a local token
   5. On every API calls, add `X-ZUMO-AUTH` with the content of `authenticationToken`
- It works on many scenarios except Server-Sent Events and Web Socket, which headers cannot be altered
   - I remember new `fetch()` could now build a Web Socket and passing headers, but I could not find it now
   - [`@microsoft/fetch-event-source`](https://npmjs.com/package/@microsoft/fetch-event-source) is outdated and don't like its API signature

### Azure Cosmos DB change feed triggering Azure Functions

- Better queue it up to Service Bus
- Otherwise, when Azure Functions fail, you have no way to retry

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

When returning `{ done: true, value: 123 }`, the value `123` will probably gone if it's done through for-loop.

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
