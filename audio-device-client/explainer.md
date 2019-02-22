# Audio Device Client: Better and Faster Audio I/O on Web

This is a proposal for a new web API, `Audio Device Client`, which funcions as
an intermediate layer between Web Audio API and actual audio devices used by the
browser. It exposes various low-level properties that have been completely
hidden or unavailable to developers.


## Rationale

Web Audio API has been criticized by the lack of ability to run low-level audio
processing properly. W3C Audio Working Group addressed the problem with the
newly added Audio Worklet, but it is still confined by the boundary of Web Audio
API's graph rendering mechanism.

To overcome this limitation, some "pro-audio" web apps were designed with a new
processing model and it bypasses the most of Web Audio API's graph system by
utilizing Audio Worklet, SharedArrayBuffer and Worker.

![Design pattern: WebAudio Powerhouse](https://github.com/WebAudio/web-audio-cg/blob/master/audio-device-client/images/img-1-design-pattern.png "Design pattern: WebAudio Powerhouse")

This setup is convenient to take advantage of the code compiled into
WebAssembly; therefore this new model cuts the engineering cost significantly
because audio developers can bring existing source codes to the web platform
with minimal effort. Additionally, deploying to multiple targets by using the
same source code ensures identical sonic results across various platforms
(e.g. encoding/decoding, signal processing or an entire audio application).

However, this convoluted workaround only solves some of problems that this breed
of audio-centric applications faces. It is still locked by the graph render
quantum size (128 sample-frames), and there is no unified API that configures
essential properties of audio rendering mechanism such as I/O device selection,
multi-channel I/O support, configurable sample rates and more.

The slow adoption of Web Audio API from pro-audio or game industry is the clear
evidence of why this problem is important to solve.


## Key Features

- Provides a dedicated global scope on a separate thread
  (real-time/high-priority when permitted) for audio processing purpose
- Selecting audio I/O devices can be done via `MediaTrackConstraints` pattern
- Supports variable callback buffer size (as opposed to 128 sample-frames
  limitation of Web Audio API)
- Provides implicit sample rate conversion when needed
- Serves I/O audio data from hardware in a single callback function
- Provides an AudioContext instance when requested
- HTTPS only and follows Autoplay policy


## Non-goals

- Does NOT replace Web Audio API
- Media encoding/decoding service


## Potential Use Cases

- A __teleconference app__ with custom audio processing and direct hardware
  access (e.g. complex echo cancellation, source separation, audio
  spatialization, and auditory scene analysis/music information retrieval)
- Porting __existing audio engines__ to the web platform with minimal
  engineering cost.
- A __hybrid audio application__ that uses Web Audio API's graph system
  (processing and synthesis) and customized audio hardware configuration.


## How Audio Device Client Works

![Audio Device Client](https://github.com/WebAudio/web-audio-cg/blob/master/audio-device-client/images/img-2-audio-device-client.png "Audio Device Client")

As discussed, the Audio Device Client will be an intermediate layer between Web
Audio API and actual audio devices used by browser's audio service. When it is
instantiated, UA will configures hardware accordingly and sets up a global scope
with a audio rendering thread.

For querying constraints, a pattern of `mediaDevices.enumerateDevices()` can
be used. A device client constructor takes a constraint dictionary and generates
an instance when the query is acceptable by UA. Otherwise, the promise will be
rejected.

```js
/* async scope */

const devices = await navigator.mediaDevices.enumerateDevices();

// An imaginary helper function for selecting device IDs for I/O.
const audioDeviceIds = getMyAudioDeviceIds(devices);

const clientConstraints = {
  inputDeviceId: audioDeviceIds.input,
  outputDeviceId: audioDeviceIds.output,
  sampleRate: 8000,
  callbackBufferSize: 512,
  inputChannelCount: 2,
  outputChannelCount: 6,
};

const client =
    await navigator.mediaDevices.getAudioDeviceClient(clientConstraints);
await client.addModule('my-client.js');
await client.start();
```

Operations in `AudioDeviceClientGlobalScope` is somewhat similar to Audio
Worklet. A callback function should be defined with JS and WASM so it can be
invoked by UA periodically.

Note that the user code can trigger the `AudioContext` to render its graph by
calling `contextCallback` function. After that, the rendered data from the
context can be processed in the client's global scope.

```js

import Processor from './my-audio-processor.js';

// An imaginary function that creates a storage for multi-channel audio data.
const contextOutput = generateFloat32Arrays(2, callbackBufferSize);

// Main process function of device client's global scope.
const process = (input, output, contextCallback) {
  // |input| will be routed to Web Audio API's |context.source| node, and
  // the result from context renderer will fill up |contextOutput|.
  contextCallback(input, contextOutput);

  // Takes the data rendered by AudioContext and perform custom processing to
  // fill out |output| data.
  Processor.process(contextOutput, output);
};

setDeviceCallback(process);
```

Lastly, an instance of `AudioContext` can be obtained from the device client.
This is optional and user can choose not to use `AudioContext` at all.

```js
// A client can instantiate an AudioContext.
const audioContext = client.getContext();

const oscillator = new OscillatorNode(audioContext);
oscillator.connect(audioContext.destination);
oscillator.start();
...
```


## Alternative Designs


### Relaxing the "128" rule for `AudioContext`

This idea was quickly turned down by Audio Working Group because it changes the
fundamental of Web Audio API implementation. Also this does not address
problems outside of Web Audio API such as I/O hardware configuration.


## Conclusion

The Audio Device Client brings the low-level audio functionality to the web
platform without unnecessary complexity. It provides closer access to audio
hardware with configurable parameters such as sample rate, callback buffer size
and channel count. The user code can process data in an isolated global scope
that runs one a dedicated thread with high priority when permitted. This new
design solves many issues that Web Audio API currently faces by exposing a
fundamental layer of audio system via an easy and safe path for developers.
