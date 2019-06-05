**Note: that this content is work-in-progress at the early stage of
development.**


# Code Examples

## Instantiation of AudioDeviceClient

```js
async () => {
  const devices = await navigator.mediaDevices.enumerateDevices();

  // Scenario: device #0 and #2 are audio input and output devices respectively.
  const constraints = {
    inputDeviceId: devices[0].deviceId,
    outputDeviceId: devices[2].deviceId,
    sampleRate: 8000,
    callbackBufferSize: 512,
    inputChannelCount: 2,
    outputChannelCount: 6,
  };

  const client =
      await navigator.mediaDevices.getAudioDeviceClient(constraints);
  await client.addModule('my-client.js');
  await client.start();
};
```

## AudioContext support

```js
// A client can instantiate an AudioContext.
const audioContext = client.getContext();

const oscillator = new OscillatorNode(audioContext);
oscillator.connect(audioContext.destination);
oscillator.start();
```

## Definition of AudioDeviceCallback

```js
/* AudioDeviceClientGlobalScope: my-client.js */

import Engine from './my-audio-engine.js';

// An imaginary function that creates a storage for multi-channel audio data.
const contextOutput = generateFloat32Arrays(2, callbackBufferSize);

// Main process function of device client's global scope.
setDeviceCallback((input, output, contextCallback) => {
  // |input| will be routed to Web Audio API's |context.source| node, and
  // the result from context renderer will fill up |contextOutput|. With the
  // setting of 512 sample frames, this will pull the graph 4 times.
  contextCallback(input, contextOutput);

  // Takes the data rendered by AudioContext and perform custom processing to
  // fill out |output| data.
  Engine.render(contextOutput, output);
});
```

## Raw I/O mode

```js
const constraints = {
  mode: 'raw',
  inputDeviceId: devices[0].deviceId,
  outputDeviceId: devices[2].deviceId,
  callbackBufferSize: 512,
  inputChannelCount: 2,
  outputChannelCount: 2,
};

async () => {
  const client = await navigator.mediaDevices.getAudioDeviceClient(constraints);
  await client.addModule('my-client.js');
};
```

```js
/* AudioDeviceClientGlobalScope: my-client.js */

import Engine from './my-audio-engine.js';

setDeviceCallback(Engine.provideInput, Engine.render);
```