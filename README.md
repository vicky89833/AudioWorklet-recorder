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
