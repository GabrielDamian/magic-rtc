# WebRTC Audio Call Application - Code Flow Analysis

## Core Components Initialization

```javascript
let localConnection = null;      // RTCPeerConnection instance
let localStream = null;         // Local audio MediaStream
let dataChannel = null;         // RTCDataChannel for signaling
let remoteStream = new MediaStream(); // Remote audio stream
let iceCandidates = [];        // Collected ICE candidates
```

## Function Flow Analysis

### 1. Connection Creation Flow (`createConnection()`)
Triggered by: "Create Connection" button click
```javascript
createConnection()
├── Creates new RTCPeerConnection with STUN servers
├── Sets up event listeners:
│   ├── onicegatheringstatechange
│   ├── oniceconnectionstatechange
│   ├── onconnectionstatechange
│   ├── onicecandidate
│   └── ontrack
├── Creates data channel for signaling
└── Calls setupDataChannel()
```

### 2. Call Initiation Flow (`startCall()`)
Triggered by: "Start Call" button click
```javascript
startCall()
├── Requests microphone access (getUserMedia)
├── Adds local audio track to connection
├── Creates offer (createOffer)
├── Sets local description (setLocalDescription)
├── Collects ICE candidates
└── Copies offer+candidates to clipboard
```

### 3. Data Channel Setup Flow (`setupDataChannel()`)
Called by: createConnection()
```javascript
setupDataChannel()
├── Sets up channel event handlers:
│   ├── onopen
│   ├── onclose
│   ├── onerror
│   └── onmessage -> Handles offers/answers
└── Processes incoming signaling messages
```

### 4. Offer Handling Flow (`handleOffer()`)
Triggered by: Receiving offer via connectWithPeer()
```javascript
handleOffer()
├── Creates connection if doesn't exist
├── Sets remote description
├── Adds received ICE candidates
├── Gets microphone access
├── Adds local audio track
├── Creates answer
├── Sets local description
└── Copies answer+candidates to clipboard
```

### 5. Answer Handling Flow (`handleAnswer()`)
Triggered by: Receiving answer via connectWithPeer()
```javascript
handleAnswer()
├── Sets remote description
├── Adds received ICE candidates
└── Establishes connection
```

### 6. Peer Connection Flow (`connectWithPeer()`)
Triggered by: "Connect with Peer" button click
```javascript
connectWithPeer()
├── Parses connection data
└── Routes to:
    ├── handleOffer() if offer received
    └── handleAnswer() if answer received
```

### 7. Call Termination Flow (`hangUp()`)
Triggered by: "End Call" button click
```javascript
hangUp()
├── Stops local audio tracks
├── Closes peer connection
├── Closes data channel
├── Cleans up media streams
└── Resets UI state
```

## WebRTC Implementation Details

### Event Handler Chain

1. **ICE Gathering**:
```javascript
onicecandidate
├── Collects ICE candidates
└── Bundles with SDP when complete
```

2. **Connection State Changes**:
```javascript
onconnectionstatechange
├── 'connecting': Starts 30s timeout
├── 'connected': Updates UI, clears timeout
└── 'disconnected/failed': Updates UI
```

3. **Track Handling**:
```javascript
ontrack
├── Adds remote track to stream
└── Sets audio element source
```

### Data Channel Communication

The data channel handles signaling through structured messages:
```javascript
{
    type: 'offer/answer',
    sdp: RTCSessionDescription,
    candidates: [RTCIceCandidate]
}
```

### Connection Establishment Process

1. **Caller Side**:
```
createConnection()
    → startCall()
        → createOffer()
            → setLocalDescription()
                → Share offer+candidates
```

2. **Receiver Side**:
```
connectWithPeer()
    → handleOffer()
        → setRemoteDescription()
            → createAnswer()
                → setLocalDescription()
                    → Share answer+candidates
```

3. **Final Connection**:
```
handleAnswer()
    → setRemoteDescription()
        → Add ICE candidates
            → Connection established
```

## State Management

### Connection States:
- Disconnected: Initial state or after hangup
- Connecting: During offer/answer exchange
- Connected: Active call session
- Failed: Connection error occurred

### Audio States:
- Waiting: Initial state
- Microphone active: Local audio captured
- Active: Call in progress
- Not connected: Call ended or error

The application maintains these states through event handlers and updates the UI accordingly, ensuring users always know the current status of their connection and call.