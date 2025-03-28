# Today I learnt

Point form for speedy writing. 80% correct at the time of writing. Just remind myself what I did. Timestamp is not accurate as I may recall things and writing something I learn some time ago.

## Table of contents

| Area         | Product                     | Date                                                                          |
| ------------ | --------------------------- | ----------------------------------------------------------------------------- |
| Azure        | Azure Container Apps        | [2024-01-27](#2024-01-27)                                                     |
| Azure        | Azure Cosmos DB Core        | [2023-12-24](#2023-12-24) [2023-12-26](#2023-12-26) [2024-01-07](#2024-01-07) |
| Azure        | Azure Functions             | [2023-12-25](#2023-12-25) [2023-12-26](#2023-12-26) [2024-01-08](#2024-01-08) |
| Azure        | Azure Managed Identity      | [2024-07-13](#2024-07-13)                                                     |
| Clover       | Rest API                    | [2024-12-21](#2024-12-21)                                                     |
| JavaScript   | Iterable/iterator/generator | [2024-01-22](#2024-01-22)                                                     |
| JavaScript   | Valibot                     | [2024-01-10](#2024-01-10)                                                     |
| Raspberry Pi | CUPS                        | [2025-02-15](#2025-02-15)                                                     |
| Raspberry Pi | Pi-Hole                     | [2024-02-27](#2024-02-27) [2023-12](#2023-12)                                 |
| Raspberry Pi | Wireguard                   | [2025-02-15](#2025-02-15)                                                     |
| React        | Fluent UI                   | [2023-12-25](#2023-12-25)                                                     |
| Hardware     | Happy Hacking Keyboard      | [2023-12-24](#2023-12-24)                                                     |

## 2025-02-15

### CUPS (Common Unix Printing System)

```sh
sudo apt install cups printer-driver-gutenprint

# Auto-start CUPS
sudo systemctl enable --now cups.service

# Enable admin
sudo usermod -a -G lpadmin pi

# Enable remote access
sudo cupsctl --remote-any

# Restart CUPS to save changes
sudo systemctl restart cups
```

Then, navigate to `https://<hostname>:631/` to add a printer. Check "Share this printer".

- Android
   - Install [Mopria app](https://mopria.org/print-from-android)
   - Add printer, hostname is `<hostname>:631/printers/Your_Printer_Name` (no `http://`)
- Windows
   - Add IPP printer with URL of `http://<hostname>:631/printers/Your_Printer_Name`

### Wireguard client in Raspberry Pi (Bookworm)

```sh
sudo apt update

# "openresolv" required as Raspberry Pi does not have "resolvconf".
sudo apt install wireguard openresolv

# Import wg0.conf.
# Add "PersistentKeepalive = 25" if the NAT router kills UDP too soon
sudo pico /etc/wireguard/wg0.conf

sudo wg-quick up wg0
sudo systemctl enable --now wg-quick@wg0
```

### Wi-Fi configuration in Raspberry Pi (Bookworm)

Run `nmtui` in terminal.

## 2025-02-13

- Testing in browser
   - Using `jest.fn` and `jest.spyOn` in browser means `import { fn, spyOn } from 'jest-mock'` under `<script type="module">`
   - [Import map](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/script/type/importmap) is like a much simplified version of `package.json`
- To efficiently spy a function and setup expectation
   - Spy: `spyOn(console, 'log')`, no need to assign to a constant
   - Expectation: `expect(console.log).toHaveBeenCalledTimes(1)`

## 2024-12-30

- Bundling in monorepo
   - Monorepo with hoisted dependencies means most `/node_modules/` are located at the root of the project
   - Deploying a package to run in container means the root `/node_modules/` need to be packed
      - The root `/node_modules/` may contains dependencies used by other packages
   - Bundling helps picking the minimal dependencies need to be packaged to run

## 2024-12-24

### Azure CosmosDB

- Official emulator does not run on ARM64 yet
   - Linux and ARM64 is in preview, https://learn.microsoft.com/en-us/azure/cosmos-db/emulator-linux
   - The image is huge (> 2 GB)
   - `EXISTS` is not implemented
- [cosmium](https://github.com/pikami/cosmium/) is an unofficial emulator written in Go
   - Patch operation support is limited to [RFC-6902](https://www.rfc-editor.org/rfc/rfc6902)
      - No `set` operation support, [etc.](https://learn.microsoft.com/en-us/azure/cosmos-db/partial-document-update#supported-operations)

#### Quirks

| Scenario                         | Real | Official emulator  | cosmium         |
| -------------------------------- | ---- | ------------------ | --------------- |
| Bracket notation                 | ✅   | ✅                 | ✅              |
| Bracket notation with parameter  | ✅   | ❌ Syntax error    | ❌ Return empty |
| `ARRAY_CONTAINS`                 | ✅   | ✅                 | ❌ Return empty |
| `ARRAY_CONTAINS` with parameter  | ✅   | ❌ Syntax error    | ❌ Return empty |
| `EXISTS` subquery                | ✅   | ❌ Not implemented | ✅              |
| `EXISTS` subquery with parameter | ✅   | ❌ Not implemented | ✅              |
| `batch()`                        | ✅   | ?                  | ❌ Emulator-side process error |
| `bulk()`                         | ✅   | ?                  | ❌ Emulator-side process error |

Code snippets:

- Bracket notation: `WHERE c.map[@name] = @value`
- `ARRAY_CONTAINS`: `WHERE ARRAY_CONTAINS(c.array, @value)`
- `EXISTS`: `WHERE EXISTS (SELECT p FROM p IN c.array WHERE p = @value)`

## 2024-12-21

### Clover

- `orderBy=modifiedTime+ASC` is not working properly
   - It returns data in ascending order, but not from day 0, but some random days
   - However, descending is working properly
- Pagination (`limit`/`offset`) only works for 90 days of data
   - It will return end of data after 90 days
   - Use `modifiedTime` for pagination instead, i.e. `filter=modifiedTime%3C%3D1734773188000`
- Pagination (`limit`/`offset`) is too naive to use with real-time data
   - Page overlap logic is required when data is expected to update in real-time
- Customer table does not have `modifiedTime` field despite there is a `metadata.modifiedTime` field
   - Default seems to be order by `customerSince DESC`
      - Order by `customerSince ASC` will sort it in ascending order
      - Order by `modifiedTime DESC` will do nothing
      - Order by `metadata.modifiedTime DESC` will throw HTTP 400
   - Modifying a customer will not move its position
- Product item table will update frequently because every order will change item stock
- Webhooks requires OAuth and probably publishing app publicly
   - Per documentation, webhooks may not be very reliable
- If server is too busy, it may simply kill the socket connection instead of returning 429
- For HTTP 429, despite throttling is 16 requests per second for token access, the quota does not reset frequently

## 2024-11-07

### Web Components

Template of `MyElement.ts`:

```ts
export const observedAttributes = Object.freeze(['data-value']); // For HTML sanitizer
export const tagName = 'my-tag-name'; // For HTML sanitizer

type ObservedAttribute = IterableElement<typeof observedAttributes>;

class MyElement extends HTMLElement {
  static observedAttributes: readonly string[] = observedAttributes;
}

let defined = false;

export function defineMyElement() {
  if (!defined) {
    customElements.define(tagName, MyElement);
    defined = true;
  }
}

// Type-friendly way to create the element.
export function createMyElement(
  ownerDocument: Document,
  attributesInitDict: Readonly<{ [K in ObservedAttribute]?: string | undefined; }>
): MyElement {
  defineMyElement();

  const myElement = ownerDocument.createElement(tagName) as MyElement;

  myElement.dataset['value'] = attributesInitDict['data-value'];

  return myElement;
}
```

## 2024-10-24

- `markdown-it` vs. `micromark`, https://github.com/microsoft/BotFramework-WebChat/pull/5330
- `micromark` is more like a SAX pipeline
   - We can't parse Markdown via `mdast` into AST and render it via `micromark` because `micromark` is SAX and not AST

## 2024-08-25

- `<input type="hidden">` will not participate in [HTML Constraint Validation](https://developer.mozilla.org/en-US/docs/Web/HTML/Constraint_validation)
   - Calling `HTMLInputElement.setCustomValidity('Some error')` will not fail `HTMLFormElement.checkValidity()`

## 2024-08-22

- Imperative function is "what you would do"
   - `while (speed < 60) { accelerate(); }`
- Declarative function is "what you want"
   - `cruiseControl(60);`

## 2024-07-19

### Using P-in-P as widgets

- `<video>` is currently the only way to do P-in-P
- Steps:
   1. Create `<canvas>`, no need to attach to DOM
   1. `videoElement.muted = true` to allow programmatically play
   1. `videoElement.srcObject = canvasElement.captureStream()` to play `<canvas>` in the `<video>` at zero FPS (on-demand)
   1. Draw on canvas
   1. [`MediaStream.getVideoTracks()[0].requestFrame()`](https://developer.mozilla.org/en-US/docs/Web/API/CanvasCaptureMediaStreamTrack/requestFrame) to capture `<canvas>` into `<video>`
   1. `await videoElement.play()` to start playing the video again
   1. On `videoElement.timeUpdate` event, call `videoElement.pause()` to pause immediately
      - This allows browser/device to go to sleep
   1. On `click` event, call `videoElement.requestPictureInPicture()`, P-in-P requires gesture

## 2024-07-13

### Azure Managed Identity

- Managed Identity is a resource running under resource group, similar to App Registrations but running under directory
   - Easier to clean up
- 2 ways to authenticate the running code as managed identity: federated identity or running under Azure (with identity assigned)
- Producing token: One resource (say, Web Apps) can be operated under 1+ identities. Which identity to use to talk can be selected.
   - Usually, an HTTP token server on localhost:12345 will be able to generate token for code that run under Azure
   - Different service use different implementation token server
   - Use `new ManagedIdentityCredential({ clientId: 'or process.env.AZURE_CLIENT_ID' }).getToken('https://vault.azure.net')`
      - https://vault.azure.net is the scope
      - A single scope must be set, otherwise, it will conside it's multiple scopes and getting the token will fail
- Consuming token: varies from service to service
    - Computer Vision use `Authorization: Bearer eyJ`
    - Speech SDK use `Authorization: Bearer aad#/subscription/...#eyJ`

#### Token server

Each Azure service have their own similar implementation of token server and is only accessible locally on the same box.

```
export IDENTITY_ENDPOINT=http://localhost:4141/MSI/token
export IDENTITY_HEADER=12345678-1234-5678-abcd-12345678abcd

wget --header "x-identity-header: $IDENTITY_HEADER" $IDENTITY_ENDPOINT?resource=https://vault.azure.net&api-version=2019-08-01

GET /MSI/token?resource=https://vault.azure.net&api-version=2019-08-01 HTTP/1.1
Host: localhost:4141
X-IDENTITY-HEADER: 12345678-1234-5678-abcd-12345678abcd
```

`resource` means scope. For example, Speech Services scope is https://cognitiveservices.azure.com. This is not OIDC, nothing at https://vault.azure.net/.well-known/openid-configuration.

## 2024-06-29

### Jest to mocha

```sh
npm install expect mocha sinon --save-dev
```

```diff
+ import { expect } from 'expect';
+ import { fake } from 'sinon';

- test('should work', () => {
+ it('should work', () => {

-   const fn = jest.fn();
+   const fn = fake(() => {});

    fn(1);

-   expect(fn).toHaveBeenCalledTimes(1);
+   expect(fn).toHaveProperty('callCount', 1);

-   expect(fn).toHaveBeenCalledNthWith(1, 1);
+   expect(fn.getCall(0)).toHaveProperty('args', [1]);
  });
```

## 2024-06-26

- If a HD/4K UVC is connected via USB 2.0, it will not announce availability of 1920x1080 YUV2 and formats that requires bandwidth of USB 3.1
- VLC is better at controlling audio buffering than ffmpeg/ffplay

### Play Elgato HD60 X on a Raspberry Pi

Elgato HD60 X is using standard 1920x1080 YUYV (4:2:2) and NV12 (4:2:0), so it is supported by `v4l2` without any extra drivers. Tested to work under Raspberry Pi Lite OS (Bookworm) with `sudo apt-get install xinit vlc`.

This is more-or-less UVC-to-HDMI converter. Using 1920x1080 (HD) to output to ATEM Mini Pro, should be good for 3840x2160 (4K) as well. So I can play Xbox in 4K while streaming RTMP via ATEM Mini Pro at HD. Essentially bundling Elgato HD60 X and Raspberry Pi 4 together as a HDMI downscalable splitter.

Total latency from Xbox Series X -> Elgato HD60 X -> Raspberry Pi 4 -> ATEM Mini Pro -> RTMP server -> OBS is about 0.5-1 seconds. RTMP is the biggest factor.

Put this under `crontab` with `@reboot /home/pi/playback.sh`.

```sh
cvlc \
  v4l2:///dev/video0:width=1920:height=1080:chroma=YUYV &

cvlc \
  --audio-desync=12 \
  -A alsa \
  --alsa-audio-device sysdefault:CARD=vc4hdmi0 \
  alsa://hw:CARD=X,DEV=0
```

## 2024-06-25

- View webcam (UVC) on Raspberry Pi, so it "converts" USB-C webcam into HDMI signal
   - `sudo apt-get install ffmpeg vlc xinit`
   - `v4l2-ctl --list-formats-ext` to see what resolution/chroma it support
   - VLC: `cvlc v4l2:///dev/video0:chroma=H264:width=1920:height=1080` as 1920x1080 with h.264 "chroma"
   - ffplay: `ffplay /dev/video0 -f v4l2 -input_format h264 -video_size 1920x1080 -vcodec h264_v4l2m2m`
      - This is decoded via Raspberry Pi hardware decoder (`h264_v4l2m2m`)
- Each webcam has different resolution/chroma, for example
   - Razer Kiyo Pro output 1920x1080 of h.264 or MJPEG, or 640x360 as YUYV (4:2:2) or NV12 (4:2:0)
   - ATEM Mini output 1920x1080 of MJPEG
   - Elgato HD60 X output 1280x720 YUYV (4:2:2) or NV12 (4:2:0)
      - ~HD and 4K profiles on Elgato HD60 X is not detected by `v4l2`~ (This is because using a USB 2.0 cable)
- Both Windows and Android (Xperia 1 V) can use Elgato HD60 X with HD/4K signal of unknown chroma, seems limitation on `v4l2` instead of proprietary chroma/codec

## 2024-06-01

- tsup IIFE
  - Will emit `var globalName = (() => { ... });`, not exactly UMD but close
  - Ignore `external` and will bundle everything because IIFE cannot load other deps via require/import
  - Need `platform: 'browser' to load "browser" conditions

## 2024-05-16

### React performance

The following is less performant than the latter.

```diff
  const MyComponent = memo(...);
  const App = ({ children }) => (
    <MyComponent>
-     <div>{children}<div>
+     {useMemo(() => <div>{children}<div>, [children])}
    </MyComponent>
  );
```

This is because the `children` props change on every re-render.

```diff
- <MyComponent children={<div>{children}</div>} />
+ <MyComponent children={useMemo(() => <div>{children}</div>, [children])} />
```

## 2024-05-01

- Hoisted vs. non-hoisted
   - In non-hoisted, some packages may bring another version of production package directly under root `/node_modules` as transient
   - The "wrong" version will become more visible and could be mistook by esbuild or Webpack

## 2024-04-25

### `tsup`

- `splitting` means if it should code-split common parts across 2 entrypoints (`true`/`undefined`), or just duplicate them (`false`)
   - For React Context, it is important to have a single code, rather than duplicated
- Instead of moving stuff from `dependencies` to `devDependencies`, we can also mark a package via `noExternal: ['bundle-this-package']`
- Type portability means, all types used in all exported code are exported as well
   - If there are types that we don't want to export (internal/private), we should rewrite the type in the exported code so we cut the connection there
- `dts: true` seems not checking type portability, but `experimentalDts: true` do check
- `experimentalDts` or `@microsoft/api-extractor` requires the `tsconfig.json` to put on the project root, than next to the code inside `/src/`
- Node.js don't know about `package.json/module` at all, it is a de facto standard used by Webpack et al. only

### Being `create-react-app`-friendly

- Don't use `.cjs`/`.mjs` file extension, use `.js` only
   - Otherwise, Webpack in `create-react-app` will consider it an asset file similar to `.gif` or `.txt`, i.e. returning a string and copied to the asset folder
   - `package.json/type` should be the module format of the `.js` file referenced by the `package.json/main` field

## 2024-04-14

- CSS: `prefers-reduced-motion: reduce` does not stop GIF animation from playing

## 2024-04-09

Easter Eggs = breathe life into product.

EEE, maybe: expected, exceeded, extraordinary.

## 2024-03-31

### Semantic version: `0.0.1` vs. `0.1.0` vs. `1.0.0`

- Why `0.0.1` is consider less stable?
   - `0.0.2` is not picked up by `npm install my-package@^0.0.1`
   - `0.0.1` -> `0.0.2` is considered a major bump, could introduce breaking changes
   - `0.0.1` can still be a very high quality build, but it has a tendency to introduce breaking changes in short future, i.e. unstable
   - Unstable and production ready are two different metrics, they are orthogonal of each other. A version can be both unstable and production ready
- In some perspective:
   - `0.0.1`: "I will break your stuff on next release."
      - The product is in experimental phase
   - `0.1.0`: "I will add new feature on next release. Bugs could be fixed along with new features."
      - The product is in exponential growth phase
   - `1.0.0`: "I am okay to pause new work and fix bugs."
      - The product has full support capacity
   - `0.0.1` and `0.1.0` doesn't mean quality issues. It is more about prioritizing release schedule over full support

## 2024-03-21

### GitHub Actions

- Why prevent outsider from running workflow despite the workflow is read-only and use a read-only token?
   - Outsider can modify workflow in a pull request and run their own payload

## 2024-02-28

### [JSON-LD](https://www.w3.org/TR/json-ld11/) and [Schema.org](https://schema.org/)

- [Base IRI](https://www.w3.org/TR/json-ld11/#base-iri): `{ "@id": "" }` (empty string) to represent the document base (e.g. the thing that describes the current webpage)
- [Blank node identifiers](https://www.w3.org/TR/json-ld11/#identifying-blank-nodes): `{ "@id": "_:b1" }` or `{ "@id": "_:any-valid-string" }` to represent nodes that appears locally
   - Blank node identifiers is used for serializing a graph with a cyclic dependencies and flattening
   - Nodes that reference other nodes is called [blank node](https://www.w3.org/TR/rdf11-concepts/#dfn-blank-node) and it should only have the `@id` property specified
- `{ "@type": "@json" }` to mark the data as JSON and keep it as-is during JSON-LD transformation
   - Otherwise, JSON-LD processor will ignore unknown properties and removed during transformation
- For multiple inheritance, use `{ "@type": ["DigitalDocument", "MediaObject"] }` (with [most-recently-defined-wins](https://www.w3.org/TR/json-ld11/#advanced-context-usage))
- [JSON array in JSON-LD](https://www.w3.org/TR/json-ld11/#advanced-concepts) is implicitly unordered (a.k.a. set), ordered needs to be explicitly specified (a.k.a. list)
   - JSON-LD considers set/list is a special type of map with an indexer
- Flattened vs. embedded graph
   - Flattened: all nodes are at top-level and potentially connected using IRIs or blank nodes
   - Embedded: nodes can be nested into another node, for referencing other nodes that already exists in the graph, IRIs or blank nodes maybe used
- Schema.org specifics
   - Singular vs. plural: both is allowed for all properties. Look at property description if they should be explicitly plural
      - [`keywords`](https://schema.org/keywords) is very likely to be plural
      - [`firstAppearance`](https://schema.org/firstAppearance) is very likely to be singular
   - [Key concepts of `Claim`](https://github.com/schemaorg/schemaorg/issues/1828#issuecomment-473390948)
   - "Some data is better than no data."
- Very Google-driven and targeting SEO scenarios

## 2024-02-27

### CSS

- `align-items: flex-start` or `align-self: flex-start` will interrupt `text-overflow: ellipsis`

### Wireguard client on Raspberry Pi

1. Install PiVPN by `curl -L https://install.pivpn.io/ | bash`
1. Overwrite `/etc/wireguard/wg0.conf` with the client `.conf` file (i.e. the content of QR code)
1. Reboot, done

## 2024-02-25

### Remote print to Brother VC-500W

- Generally, rebroadcast mDNS and proxy TCP 9100 as Brother is applying industry standard for the label printer
- Remote printing to printers on network over Wireguard is possible by simply proxying mDNS and TCP 9100
- Capture mDNS entry (only `ipp` is needed for Android app to work, `http`, `ipps` and `workstation` is not required)
- Proxy (a.k.a. rebroadcast) the mDNS entry
- Proxy the traffic at TCP 9100
- One IP address for one printer only, multiple IP addresses required for multiple printers

On the otherhand, the micro USB port on Brother VC-500W could be simply a USB printer and might be able to expose via CUPS. I didn't explore this area.

If mDNS can be captured over Wireguard (non-multicast network), it could be possible to automatic proxy a printer by simply providing an IP address. I am unsure if mDNS is reachable by simple unicasting.

## 2024-02-17

### Brother VC-500W color label printer

- Brother VC-500W broadcast itself as an AirPrint printer (`pdl = application/octet-stream,image/jpeg,image/png,image/urf`, `kind = roll`, `printer-type = 0x904E`)
- [Android app](https://play.google.com/store/apps/details?id=com.brother.ptouch.colorlabeleditor2) talks to the printer via AirPrint protocol (`ipp` over port 631, no need `ipps`, `http` and `workstation`)
- It broadcasts itself through mDNS over 224.0.0.251:5353
- Despite the mDNS says the printer is listening to TCP port 631, the Android app connects to TCP port 9100

## 2024-01-27

## [Reading docs about Azure Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/)

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
- Once a job trigger is configured, it is not possible to reconfigure it with another trigger
- Takes about 15 seconds to boot and run
- One job resource = one job + one trigger + multiple container (init + job)

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
