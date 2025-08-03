# AudioWorklet-recorder

Let me explain both concepts in detail:

## 1. Where is the `port` connected?

The `port` in the AudioWorkletProcessor is part of the communication channel between:

```
Main Thread (JavaScript) ↔ AudioWorkletNode ↔ AudioWorkletProcessor (AudioWorkletGlobalScope)
```

Here's how the connection works:

1. **In the main thread**:
   - When you create an `AudioWorkletNode`, it automatically establishes a `MessagePort` connection
   - You access it via `audioWorkletNode.port`

2. **In the processor**:
   - The `port` property is automatically available in your `AudioWorkletProcessor` instance
   - It's connected to the corresponding `AudioWorkletNode` in the main thread

Example flow:
```javascript
// Main thread
const processorNode = new AudioWorkletNode(context, 'recorder-processor');
processorNode.port.postMessage('hello processor!');

// Processor thread (in your RecorderProcessor)
this.port.onmessage = (e) => {
  console.log(e.data); // Will receive 'hello processor!'
};
```

## 2. Why do we register the processor with `registerProcessor`?

The `registerProcessor()` call serves two critical purposes:

1. **Exposes the processor to the AudioWorklet system**:
   - Makes your processor class available to be instantiated when creating AudioWorkletNodes
   - The string name ('recorder-processor') is what you use when creating nodes

2. **Establishes the processor in the AudioWorkletGlobalScope**:
   - Each AudioWorklet has its own JavaScript scope (different from main thread)
   - `registerProcessor` adds your class to this special scope

### Complete Connection Flow:

1. **Main Thread**:
```javascript
// 1. Load the processor code
await audioContext.audioWorklet.addModule('processor.js');

// 2. Create node with registered name
const workletNode = new AudioWorkletNode(audioContext, 'recorder-processor');

// 3. Set up communication
workletNode.port.onmessage = (e) => {
  console.log('Message from processor:', e.data);
};
```

2. **Processor Thread** (in processor.js):
```javascript
class RecorderProcessor extends AudioWorkletProcessor {
  constructor() {
    super();
    this.port.onmessage = (e) => {
      // Handle messages from main thread
    };
  }
  
  process(inputs) {
    // Send messages to main thread
    this.port.postMessage('audio data');
  }
}

// Makes the processor available to the AudioWorklet system
registerProcessor('recorder-processor', RecorderProcessor);
```

### Key Points:

- The `port` is automatically connected when the AudioWorkletNode is created
- `registerProcessor` is required to make your processor class available to the audio worklet system
- Communication is asynchronous and uses the MessageChannel API under the hood
- The processor runs in a separate audio rendering thread, not the main JavaScript thread

This architecture allows for:
- Real-time audio processing without blocking the main thread
- Thread-safe communication between main thread and audio thread
- Efficient audio data transfer through the ports


# AudioWorklet Usage and Control in the Recorder

Let me explain how the AudioWorklet is used and managed in this audio recorder implementation:

## How AudioWorklet is Used

### 1. **Initialization**
- **Loading the Processor**: 
  ```typescript
  await this.audioContext.audioWorklet.addModule('src/audio-processor.js');
  ```
  This loads the AudioWorklet processor script into the audio context.

- **Creating the Node**:
  ```typescript
  this.processorNode = new AudioWorkletNode(this.audioContext, 'recorder-processor');
  ```
  Creates an instance of the registered processor ("recorder-processor").

### 2. **Audio Processing Flow**
1. Microphone input → MediaStream
2. MediaStream → MediaStreamSourceNode
3. SourceNode → AudioWorkletNode (our processor)
4. Processor sends data back via MessagePort

### 3. **Real-time Audio Processing**
The processor's `process()` method runs continuously (typically every 128-4096 samples) in a separate audio thread:

```javascript
process(inputs) {
  if (this.stopProcessing) return false;
  
  const input = inputs[0];
  if (input && input.length > 0) {
    this.port.postMessage(input[0]);
  }
  return true;
}
```

## How AudioWorklet is Stopped

There are several ways the AudioWorklet processing is stopped:

### 1. **Explicit Stop Command**
```typescript
this.processorNode.port.postMessage('stop');
```
This sends a message to the processor to set `stopProcessing = true`, causing the next `process()` call to return `false` and terminate the processor.

### 2. **Disconnecting Nodes**
```typescript
this.processorNode.disconnect();
```
Removes the processor from the audio graph, stopping audio flow.

### 3. **Closing AudioContext**
```typescript
this.audioContext.close();
```
Stops all audio processing in the context, including all worklet processors.

### 4. **Stopping MediaStream Tracks**
```typescript
this.mediaStream.getTracks().forEach(track => track.stop());
```
Stops the source of audio data, which eventually stops the processor since no new data arrives.

## Key Lifecycle Points

1. **Start**:
   - Create AudioContext
   - Get microphone permission
   - Load and initialize AudioWorklet
   - Connect audio nodes

2. **Processing**:
   - Audio data flows continuously
   - Processor sends chunks to main thread
   - Main thread collects chunks

3. **Stop**:
   - Send stop message
   - Disconnect nodes
   - Close context
   - Stop media tracks

4. **Cleanup**:
   - Release resources
   - Generate WAV file
   - Update UI

## Threading Considerations

- **AudioWorklet runs in a separate audio rendering thread** (not main thread)
- **MessagePort communication is asynchronous** between threads
- **Audio processing must be real-time** (no blocking operations)
- **The processor can be stopped from either thread** (main thread via disconnect, or audio thread by returning false)

This implementation provides a robust way to record audio with proper start/stop control while maintaining low latency through the AudioWorklet API.

