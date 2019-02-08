**Note: that this content is work-in-progress at the early stage of
development.**


# WebIDL (WIP)

### AudioDeviceClientState
An `enum` represents the current state of the client.

```webidl
enum AudioDeviceClientState {
  "pending",
  "running",
  "stopped"
}
```

### AudioDeviceClient

The low-level representation of an audio device. Provides various
hardware-related properties and a path to the associated `AudioContext` for Web
Audio API tasks.

```webidl
[Exposed=Window, SecureContext, NoConstructor]
interface AudioDeviceClient : EventTarget {
  readonly attribute USVString deviceId;
  readonly attribute float sampleRate;
  readonly attribute unsigned callbackBufferSize;
  readonly attribute unsigned inputChannelCount;
  readonly attribute unsigned outputChannelCount;
  readonly attribute AudioDeviceClientState state;
  readonly attribute MessagePort port;
  attribute EventHandler onstatechange;
  Promise<void> start(USVString moduleUrl);
  Promise<void> stop();
  [SameObject] AudioContext getContext();
}
```

### AudioDeviceClientConstraints

A dictionary describes user-specified device constraints. It can be passed when
`AudioDeviceClient` object is instantiated.

```webidl
dictionary AudioDeviceClientConstraints {
  USVString deviceId;
  optional float sampleRate;
  unsigned callbackBufferSize = 128;
  unsigned inputChannelCount = 0;
  unsigned outputChannelCount = 2;
}
```

### MediaDevices.getAudioDeviceClient()

An `AudioDeviceClient` instance can be instantiated from
`navigator.mediaDevices`.

```webidl
[Exposed=Window]
partial interface MediaDevices : EventTarget {
  Promise<AudioDeviceClient> getAudioDeviceClient(
      AudioDeviceClientConstraints constraints);
}
```

### AudioContextCallback

A callback function that processes a single AudioContext render quantum. User
provides the content of `input` array, and then the context will render the
content of the `output` array.

```webidl
callback AudioContextCallback = void (sequence<Float32Array> input,
                                      sequence<Float32Array> output);
```

### AudioDeviceCallback

A callback function that processes a single `AudioDeviceClient` render quantum.
The input array will be provided by the system and user files the content of the
`output` array. User also can use the passed `callback` function to explicitly
pull samples from the associated context’s audio graph.

```webidl
callback AudioDeviceCallback = void (sequence<Float32Array> input,
                                     sequence<Float32Array> output,
                                     AudioContextCallback callback);
```

### AudioDeviceClientGlobalScope

This global scope is similar to `AudioWorkletGlobalScope` and runs on a
dedicated audio rendering thread that can potentially run with RT priority. The
functionality is fairly limited for the optimum rendering performance, so any
thread-blocking operation (i.e. networking, dynamic module loading and etc.)
should not be permitted.

```webidl
[Exposed=AudioDeviceClient]
interface AudioDeviceClientGlobalScope {
  void close();
  void postMessage(any message, sequence<object> transfer);
  void setDeviceCallback(AudioDeviceCallback callback);
  attribute readonly DOMHighResTimeStamp currentTime;
  readonly attribute float renderCapability;
  attribute EventHandler onerror;
  attribute EventHandler onmessage;
  attribute EventHandler onmessageerror;
}
```
