

WebKit Inspector. go to settings > console. turn Media Logging and MSE Logging to verbose.
this log is pretty reliable .. the wasPlaybackPrevented part.
       [Log] HTMLMediaElement::handleAutoplayEvent(AC2466075C23F684) hasAudio = true wasPlaybackPrevented = true hasMainContent = false



gesture is required for autoplay w/ audio. if muted, autoplay attribute if enough.

this line is what mostly makes this call
https://github.com/WebKit/WebKit/blob/c7229df52bd1e42c1508970472b0e4792d24140c/Source/WebCore/html/MediaElementSession.cpp#L433


its called from here:
https://github.com/WebKit/WebKit/blob/c7229df52bd1e42c1508970472b0e4792d24140c/Source/WebCore/html/HTMLMediaElement.cpp#LL2699


when a gesture is created...
   `UserGestureIndicator gestureIndicator(ProcessingUserGesture` in https://github.com/WebKit/WebKit/blob/14d2a0be85d000c43f1da208426729b364494438/Source/WebCore/page/EventHandler.cpp
   found in:
   - mousepress
   - mousedoubleclick
   - mouserelease
   - touch

it propogates up and down the frame tree:
   https://github.com/WebKit/WebKit/blob/14d2a0be85d000c43f1da208426729b364494438/Source/WebCore/dom/UserGestureIndicator.cpp#L62-L78
unfortunately that frametree walk is specific to the documents loadded in each frame...
i wish it was more frame centric..

but.. i wonder if i can "load" the player iframe in like.. mousedown and then use a 'click' handler to set gesture.. if that'll satisfy.
...not sure.


also.. they added usergessture propogation through a fetch() promise: https://github.com/WebKit/WebKit/commit/b85e5fd57aaef95e15eb4efd07e3fd31d5891caa


```cpp
static Seconds maxIntervalForUserGestureForwardingForFetch { 10 };
```
https://github.com/WebKit/WebKit/blob/14d2a0be85d000c43f1da208426729b364494438/Source/WebCore/dom/UserGestureIndicator.cpp#LL87

i got up to 10 seconds?
https://github.com/WebKit/WebKit/blob/main/LayoutTests/http/tests/media/user-gesture-preserved-across-xmlhttprequest-expected.txt and can combo some timeouts with the fetch??
the tests showing that `fetch().then(r => r.blob()).then(.....)` still work suggest that reading it as a stream should also preserve the gesture.. adn stream might be easier to manipulate..  https://developer.mozilla.org/en-US/docs/Web/API/Streams_API/Using_readable_streams#consuming_a_fetch_as_a_stream


```js
[
{ title : 'gesture -> "long" Fetch -> video playback: should ❌', action : 'play',  withkey : true,  success : false,  timeout : 0.2, sequence : ['300 Fetch'] },
{ title : 'gesture -> "long" XHR -> video playback: should ❌', action : 'play',  withkey : true,  success : false,  timeout : 0.2, sequence : ['300 XHR'] },
{ title : 'gesture -> fetch -> "long" blob -> video playback: should ❌', action : 'play',  withkey : true,  success : false,  timeout : 0.2, sequence : ['300 FetchBlob'] },
{ title : 'gesture -> fetch -> crypto -> video playback: should ❌ since the user gesture will not be propagated through other API even though it returns promises', action : 'play',  withkey : true,  success : false,  sequence : ['0 FetchThenCrypto'] },
{ title : 'gesture -> promise -> video playback: should ❌ because promise only propagates user gesture propagated through Fetch', action : 'play',  withkey : true,  success : false,  sequence : ['0 promise'] },
{ title : 'gesture -> fetch -> blob -> video playback: should 😁', action : 'play',  withkey : true,  success : true,  sequence : ['100 FetchBlob'] },
{ title : 'gesture -> fetch -> video playback: should 😁', action : 'play',  withkey : true,  success : true,  sequence : ['100 Fetch'] },
{ title : 'gesture -> fetch -> promise -> video playback: should 😁', action : 'play',  withkey : true,  success : true,  sequence : ['0 FetchThenPromise'] },
{ title : 'gesture -> timeout -> XHR -> timeout -> video playback: should 😁', action : 'play',  withkey : true,  success : true,  sequence : ['100 timeout', '100 XHR', '100 timeout'] },
{ title : 'gesture -> XHR -> timeout -> XHR -> video playback: should 😁', action : 'play',  withkey : true,  success : true,  sequence : ['100 XHR', '100 timeout', '100 xhr'] },
]
```


### experimenting

- this works in desktop safari:
```js
  document.querySelector('.playifrvid').addEventListener('click', e => {
    setTimeout(() => {
      document.querySelector('iframe').contentWindow.document.querySelector('video').play();
    }, 990);
  });
```
- and without a timeout it works obv
- once the timeout is >=1000 it fails with:
> Error HTMLMediaElement::play(4A65A19E76E38A17) rejecting promise: UserGestureRequired
> Error Unhandled Promise Rejection: NotAllowedError: The request is not allowed by the user agent or the platform in the current context, possibly because the user denied permission.

.. but weirdly when i roughly do the same experiment with the YT api shit. i can go over 1000 and it plays fine...... (in desktop safari.. very weird.)


base.js:9437  g.k.play .. first line of this fn is where .play() is invoked.
thanks `debug(HTMLMediaElement.prototype.play)`  ! :)

i can see a setTimeout(fn, undefined) and a promise in this callstack. it origins from `  writeEmbed();` thats in an inline `<script>` in the API page, so.. that's cool.
and the promise is a Promise.resolve() so its synchronous.  (at least in chrome)..


using l-yt-e on my m1 laptop on home wifi, i see click -> ytapi.playVideo() as 500ms.






... right as i was putting this down. i was looking at these log lines:

```
[Warning] ytplayer invoking playVideo – 50379 (lite-yt-embed.js, line 169)
[Info] HTMLMediaElement::load(667F51CF7E5E9602)
[Log] HTMLMediaElement::prepareForLoad(667F51CF7E5E9602) gesture = false
```

if gesture=true we're good.
and i notice its never true in browserstaack... and... it probably has to do with how browserstack resimulates clicks.
so i think i need real hardware to really test this current approach.

(however the async callstack that safari constructs for that promise stillllllll weirds me out)




--------

## useragents and navigators..

Safari Mac OS `navigator` in March 2024:

```js
appCodeName: "Mozilla"
appName: "Netscape"
appVersion: "5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/17.3 Safari/605.1.15"
audioSession: AudioSession {type: "auto", addEventListener: function, removeEventListener: function, dispatchEvent: function}
clipboard: Clipboard {read: function, readText: function, write: function, writeText: function, addEventListener: function, …}
cookieEnabled: true
credentials: CredentialsContainer {get: function, store: function, create: function, preventSilentAccess: function}
geolocation: Geolocation {getCurrentPosition: function, watchPosition: function, clearWatch: function}
hardwareConcurrency: 8
language: "en-US"
languages: ["en-US"] (1)
locks: LockManager {request: function, query: function}
maxTouchPoints: 0
mediaCapabilities: MediaCapabilities {decodingInfo: function, encodingInfo: function}
mediaDevices: MediaDevices {ondevicechange: null, enumerateDevices: function, getSupportedConstraints: function, getUserMedia: function, getDisplayMedia: function, …}
mediaSession: MediaSession {metadata: null, playbackState: "none", coordinator: MediaSessionCoordinator, readyState: "havenothing", playlist: [], …}
mimeTypes: MimeTypeArray {0: MimeType, 1: MimeType, application/pdf: MimeType, text/pdf: MimeType, length: 2, item: function, namedItem: function}
onLine: true
pdfViewerEnabled: true
permissions: Permissions {query: function}
platform: "MacIntel"
plugins: PluginArray {0: Plugin, 1: Plugin, 2: Plugin, 3: Plugin, 4: Plugin, PDF Viewer: Plugin, Chrome PDF Viewer: Plugin, Chromium PDF Viewer: Plugin, Microsoft Edge PDF Viewer: Plugin, WebKit built-in PDF: Plugin, …}
product: "Gecko"
productSub: "20030107"
serviceWorker: ServiceWorkerContainer {controller: null, ready: Promise, oncontrollerchange: null, onmessage: null, onmessageerror: null, …}
standalone: false
storage: StorageManager {persisted: function, persist: function, estimate: function, getDirectory: function}
userActivation: UserActivation {hasBeenActive: false, isActive: false}
userAgent: "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/17.3 Safari/605.1.15"
vendor: "Apple Computer, Inc."
vendorSub: ""
wakeLock: WakeLock {request: function}
webdriver: false
```


Safari iOS (iphone) `navigator` in march 2024:

```js
appCodeName: "Mozilla"
appName: "Netscape"
appVersion: "5.0 (iPhone; CPU iPhone OS 12_5_7 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/12.1.2 Mobile/15E148 Safari/604.1"
cookieEnabled: true
geolocation: Geolocation {getCurrentPosition: function, watchPosition: function, clearWatch: function}
language: "en-US"
languages: ["en-US"] (1)
mediaDevices: MediaDevices {ondevicechange: null, enumerateDevices: function, getSupportedConstraints: function, getUserMedia: function, addEventListener: function, …}
mimeTypes: MimeTypeArray {length: 0, item: function, namedItem: function}
onLine: true
platform: "iPhone"
plugins: PluginArray {length: 0, item: function, namedItem: function, refresh: function}
product: "Gecko"
productSub: "20030107"
serviceWorker: ServiceWorkerContainer {controller: null, ready: Promise, oncontrollerchange: null, onmessage: null, onmessageerror: null, …}
standalone: false
userAgent: "Mozilla/5.0 (iPhone; CPU iPhone OS 12_5_7 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/12.1.2 Mobile/15E148 Safari/604.1"
vendor: "Apple Computer, Inc."
vendorSub: ""
webdriver: false
```


Safari iPad OS `navigator` in march 2024:

```js
appCodeName: "Mozilla"
appName: "Netscape"
appVersion: "5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/17.3.1 Safari/605.1.15"
audioSession: AudioSession {type: "auto", addEventListener: function, removeEventListener: function, dispatchEvent: function}
clipboard: Clipboard {read: function, readText: function, write: function, writeText: function, addEventListener: function, …}
cookieEnabled: true
credentials: CredentialsContainer {get: function, store: function, create: function, preventSilentAccess: function}
geolocation: Geolocation {getCurrentPosition: function, watchPosition: function, clearWatch: function}
hardwareConcurrency: 4
language: "en-US"
languages: ["en-US"] (1)
locks: LockManager {request: function, query: function}
maxTouchPoints: 5
mediaCapabilities: MediaCapabilities {decodingInfo: function, encodingInfo: function}
mediaDevices: MediaDevices {ondevicechange: null, enumerateDevices: function, getSupportedConstraints: function, getUserMedia: function, addEventListener: function, …}
mediaSession: MediaSession {metadata: null, playbackState: "none", setActionHandler: function, callActionHandler: function, setPositionState: function}
mimeTypes: MimeTypeArray {0: MimeType, 1: MimeType, application/pdf: MimeType, text/pdf: MimeType, length: 2, item: function, namedItem: function}
onLine: true
pdfViewerEnabled: true
permissions: Permissions {query: function}
platform: "MacIntel"
plugins: PluginArray {0: Plugin, 1: Plgin, 2: Plugin, 3: Plugin, 4: Plugin, PDF Viewer: Plugin, Chrome PDF Viewer: Plugin, Chromium PDF Viewer: Plugin, Microsoft Edge PDF Viewer: Plugin, WebKit built-in PDF: Plugin, …}
product: "Gecko"
productSub: "20030107"
serviceWorker: ServiceWorkerContainer {controller: null, ready: Promise, oncontrollerchange: null, onmessage: null, onmessageerror: null, …}
standalone: false
storage: StorageManager {persisted: function, persist: function, estimate: function, getDirectory: function}
userActivation: UserActivation {hasBeenActive: false, isActive: false}
userAgent: "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/17.3.1 Safari/605.1.15"
vendor: "Apple Computer, Inc."
vendorSub: ""
wakeLock: WakeLock {request: function}
webdriver: false
```






Safari iOS (iphone) `navigator` in from 2022ish

```js
appCodeName: "Mozilla"
appName: "Netscape"
appVersion: "5.0 (iPhone; CPU iPhone OS 12_5_6 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/12.1.2 Mobile/15E148 Safari/604.1"
cookieEnabled: true
geolocation: Geolocation {getCurrentPosition: function, watchPosition: function, clearWatch: function}
language: "en-US"
languages: ["en-US"] (1)
mediaDevices: MediaDevices {ondevicechange: null, enumerateDevices: function, getSupportedConstraints: function, getUserMedia: function, addEventListener: function, …}
mimeTypes: MimeTypeArray {length: 0, item: function, namedItem: function}
onLine: true
platform: "iPhone"
plugins: PluginArray {length: 0, item: function, namedItem: function, refresh: function}
product: "Gecko"
productSub: "20030107"
standalone: false
userAgent: "Mozilla/5.0 (iPhone; CPU iPhone OS 12_5_6 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/12.1.2 Mobile/15E148 Safari/604.1"
vendor: "Apple Computer, Inc."
vendorSub: ""
webdriver: false
```
