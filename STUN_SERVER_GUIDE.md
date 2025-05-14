# Understanding STUN Servers and WebRTC Backend Implementation

## STUN Server Explanation

### What is a STUN Server?
STUN (Session Traversal Utilities for NAT) is a protocol that helps devices behind NAT (Network Address Translation) discover their public IP address and determine any restrictions in their ability to communicate with other devices.

### How Google's STUN Server Works
```
stun:stun.l.google.com:19302
```

1. **Connection Process**:
   - Your device sends a request to stun.l.google.com:19302
   - The STUN server responds with your public IP address and port
   - This information is used in the WebRTC connection process

2. **NAT Traversal**:
   ```
   Local Device (Private IP)
        ↓
   NAT Router (Public IP)
        ↓
   STUN Server
        ↓
   Returns Public IP:Port
   ```

## Backend Implementation (AWS Lambda)

Here's how to implement a simple WebRTC signaling server using AWS Lambda:

### 1. Lambda Function (signaling.js)

```javascript
const AWS = require('aws-sdk');
const dynamoDB = new AWS.DynamoDB.DocumentClient();

// Handle WebSocket connections
exports.handleConnection = async (event) => {
    const connectionId = event.requestContext.connectionId;
    
    try {
        await dynamoDB.put({
            TableName: 'WebRTCConnections',
            Item: {
                connectionId: connectionId,
                timestamp: Date.now()
            }
        }).promise();
        
        return {
            statusCode: 200,
            body: 'Connected'
        };
    } catch (err) {
        return {
            statusCode: 500,
            body: 'Failed to connect: ' + JSON.stringify(err)
        };
    }
};

// Handle WebSocket messages
exports.handleMessage = async (event) => {
    const connectionId = event.requestContext.connectionId;
    const message = JSON.parse(event.body);
    
    try {
        switch (message.type) {
            case 'offer':
            case 'answer':
            case 'ice-candidate':
                await relayMessage(message, connectionId);
                break;
            case 'register':
                await registerPeer(message.roomId, connectionId);
                break;
        }
        
        return {
            statusCode: 200,
            body: 'Message processed'
        };
    } catch (err) {
        return {
            statusCode: 500,
            body: 'Failed to process message: ' + JSON.stringify(err)
        };
    }
};

// Relay WebRTC messages between peers
async function relayMessage(message, senderConnectionId) {
    const apiGateway = new AWS.ApiGatewayManagementApi({
        endpoint: process.env.WEBSOCKET_ENDPOINT
    });
    
    // Get all connections in the same room
    const connections = await dynamoDB.query({
        TableName: 'WebRTCRooms',
        KeyConditionExpression: 'roomId = :roomId',
        ExpressionAttributeValues: {
            ':roomId': message.roomId
        }
    }).promise();
    
    // Send message to all peers except sender
    const sendPromises = connections.Items
        .filter(conn => conn.connectionId !== senderConnectionId)
        .map(async (conn) => {
            try {
                await apiGateway.postToConnection({
                    ConnectionId: conn.connectionId,
                    Data: JSON.stringify(message)
                }).promise();
            } catch (err) {
                if (err.statusCode === 410) {
                    // Remove stale connections
                    await dynamoDB.delete({
                        TableName: 'WebRTCRooms',
                        Key: {
                            roomId: message.roomId,
                            connectionId: conn.connectionId
                        }
                    }).promise();
                }
            }
        });
    
    await Promise.all(sendPromises);
}

// Register peer in a room
async function registerPeer(roomId, connectionId) {
    await dynamoDB.put({
        TableName: 'WebRTCRooms',
        Item: {
            roomId: roomId,
            connectionId: connectionId,
            timestamp: Date.now()
        }
    }).promise();
}

// Handle disconnections
exports.handleDisconnection = async (event) => {
    const connectionId = event.requestContext.connectionId;
    
    try {
        await dynamoDB.delete({
            TableName: 'WebRTCConnections',
            Key: {
                connectionId: connectionId
            }
        }).promise();
        
        // Also remove from rooms
        const rooms = await dynamoDB.scan({
            TableName: 'WebRTCRooms',
            FilterExpression: 'connectionId = :connectionId',
            ExpressionAttributeValues: {
                ':connectionId': connectionId
            }
        }).promise();
        
        const deletePromises = rooms.Items.map(room =>
            dynamoDB.delete({
                TableName: 'WebRTCRooms',
                Key: {
                    roomId: room.roomId,
                    connectionId: connectionId
                }
            }).promise()
        );
        
        await Promise.all(deletePromises);
        
        return {
            statusCode: 200,
            body: 'Disconnected'
        };
    } catch (err) {
        return {
            statusCode: 500,
            body: 'Failed to disconnect: ' + JSON.stringify(err)
        };
    }
};
```

### 2. DynamoDB Tables

1. **WebRTCConnections Table**:
```javascript
{
    connectionId: String, // Partition Key
    timestamp: Number
}
```

2. **WebRTCRooms Table**:
```javascript
{
    roomId: String,      // Partition Key
    connectionId: String, // Sort Key
    timestamp: Number
}
```

### 3. API Gateway WebSocket Configuration

```yaml
WebSocketAPI:
  Type: AWS::ApiGatewayV2::Api
  Properties:
    Name: WebRTC Signaling
    ProtocolType: WEBSOCKET
    RouteSelectionExpression: $request.body.action
```

Routes:
- `$connect` → handleConnection
- `$disconnect` → handleDisconnection
- `$default` → handleMessage

### Client-Side Implementation

```javascript
class SignalingClient {
    constructor(websocketUrl) {
        this.ws = new WebSocket(websocketUrl);
        this.setupWebSocket();
    }

    setupWebSocket() {
        this.ws.onopen = () => {
            console.log('Connected to signaling server');
        };

        this.ws.onmessage = (event) => {
            const message = JSON.parse(event.data);
            switch(message.type) {
                case 'offer':
                    handleOffer(message);
                    break;
                case 'answer':
                    handleAnswer(message);
                    break;
                case 'ice-candidate':
                    handleIceCandidate(message);
                    break;
            }
        };
    }

    sendMessage(message) {
        if (this.ws.readyState === WebSocket.OPEN) {
            this.ws.send(JSON.stringify(message));
        }
    }

    joinRoom(roomId) {
        this.sendMessage({
            type: 'register',
            roomId: roomId
        });
    }

    sendOffer(offer, roomId) {
        this.sendMessage({
            type: 'offer',
            offer: offer,
            roomId: roomId
        });
    }

    sendAnswer(answer, roomId) {
        this.sendMessage({
            type: 'answer',
            answer: answer,
            roomId: roomId
        });
    }

    sendIceCandidate(candidate, roomId) {
        this.sendMessage({
            type: 'ice-candidate',
            candidate: candidate,
            roomId: roomId
        });
    }
}
```

## How It All Works Together

1. **STUN Server Role**:
   - Helps peers discover their public IP addresses
   - Enables NAT traversal for direct P2P connection
   - No data flows through STUN server during the call

2. **Lambda Signaling Server Role**:
   - Manages WebSocket connections
   - Relays initial connection data between peers
   - Handles room management
   - Maintains connection state

3. **Connection Flow**:
```
Peer A                    Lambda                    Peer B
  |                         |                         |
  |-- Join Room ----------->|                         |
  |                         |<-------- Join Room -----|
  |                         |                         |
  |-- STUN Request -------->|                         |
  |<- Public IP:Port -------|                         |
  |                         |                         |
  |-- Offer -------------->|-- Relay Offer --------->|
  |                         |                         |
  |                         |<- Answer --------------|
  |<- Relay Answer ---------|                         |
  |                         |                         |
  |====== Direct P2P Connection Established =======|
```

4. **Scaling Considerations**:
   - Lambda functions auto-scale
   - DynamoDB handles connection storage
   - WebSocket API Gateway manages connections
   - STUN server only needed for connection setup

This implementation provides a scalable, serverless WebRTC signaling solution while utilizing Google's STUN server for NAT traversal.