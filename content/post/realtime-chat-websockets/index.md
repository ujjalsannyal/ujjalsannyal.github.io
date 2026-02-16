---
title: "Building a Real-Time Chat Application with WebSockets"
description: "A deep dive into creating a scalable real-time chat application using WebSockets, Node.js, and React"
date: 2024-02-10
slug: realtime-chat-websockets
image: cover.jpg
categories:
    - Web Development
    - Backend
tags:
    - WebSockets
    - Node.js
    - React
    - Real-time
    - Socket.io
---

## Introduction

Real-time communication has become an essential feature in modern web applications. In this post, I'll walk you through building a scalable real-time chat application using WebSockets, Node.js, and React.

## Why WebSockets?

Traditional HTTP follows a request-response model, which isn't ideal for real-time applications. WebSockets provide:

- **Full-duplex communication**: Both client and server can send messages independently
- **Low latency**: Persistent connection eliminates handshake overhead
- **Efficient**: Less bandwidth compared to polling

## Architecture Overview

Our chat application consists of:

1. **Backend Server** (Node.js + Socket.io)
2. **Frontend Client** (React + Socket.io-client)
3. **Message Store** (Redis for pub/sub)

### Backend Implementation

```javascript
const express = require('express');
const http = require('http');
const socketIo = require('socket.io');
const Redis = require('ioredis');

const app = express();
const server = http.createServer(app);
const io = socketIo(server, {
  cors: {
    origin: process.env.CLIENT_URL,
    methods: ["GET", "POST"]
  }
});

const redis = new Redis();
const pub = new Redis();
const sub = new Redis();

// Subscribe to chat channel
sub.subscribe('chat-messages');

io.on('connection', (socket) => {
  console.log('New client connected:', socket.id);

  socket.on('join-room', (roomId) => {
    socket.join(roomId);
    socket.to(roomId).emit('user-joined', socket.id);
  });

  socket.on('send-message', async (data) => {
    const message = {
      id: Date.now(),
      userId: socket.id,
      username: data.username,
      text: data.text,
      roomId: data.roomId,
      timestamp: new Date().toISOString()
    };

    // Publish to Redis for horizontal scaling
    await pub.publish('chat-messages', JSON.stringify(message));
  });

  socket.on('disconnect', () => {
    console.log('Client disconnected:', socket.id);
  });
});

// Listen for messages from Redis
sub.on('message', (channel, message) => {
  if (channel === 'chat-messages') {
    const data = JSON.parse(message);
    io.to(data.roomId).emit('receive-message', data);
  }
});

server.listen(3001, () => {
  console.log('Server running on port 3001');
});
```

### Frontend Implementation

```javascript
import React, { useState, useEffect, useRef } from 'react';
import io from 'socket.io-client';

const ChatRoom = ({ roomId, username }) => {
  const [messages, setMessages] = useState([]);
  const [inputMessage, setInputMessage] = useState('');
  const socketRef = useRef();

  useEffect(() => {
    socketRef.current = io('http://localhost:3001');

    socketRef.current.emit('join-room', roomId);

    socketRef.current.on('receive-message', (message) => {
      setMessages((prev) => [...prev, message]);
    });

    socketRef.current.on('user-joined', (userId) => {
      console.log('User joined:', userId);
    });

    return () => {
      socketRef.current.disconnect();
    };
  }, [roomId]);

  const sendMessage = (e) => {
    e.preventDefault();
    if (inputMessage.trim()) {
      socketRef.current.emit('send-message', {
        username,
        text: inputMessage,
        roomId
      });
      setInputMessage('');
    }
  };

  return (
    <div className="chat-container">
      <div className="messages">
        {messages.map((msg) => (
          <div key={msg.id} className="message">
            <strong>{msg.username}:</strong> {msg.text}
            <span className="timestamp">
              {new Date(msg.timestamp).toLocaleTimeString()}
            </span>
          </div>
        ))}
      </div>
      <form onSubmit={sendMessage}>
        <input
          type="text"
          value={inputMessage}
          onChange={(e) => setInputMessage(e.target.value)}
          placeholder="Type a message..."
        />
        <button type="submit">Send</button>
      </form>
    </div>
  );
};

export default ChatRoom;
```

## Scaling Considerations

### Horizontal Scaling with Redis

When running multiple server instances, we use Redis pub/sub to broadcast messages across all instances:

```javascript
// Each server instance subscribes to the same channel
sub.subscribe('chat-messages');

// Messages are published to Redis instead of directly to clients
pub.publish('chat-messages', JSON.stringify(message));

// All instances receive the message and broadcast to their connected clients
sub.on('message', (channel, message) => {
  io.to(roomId).emit('receive-message', JSON.parse(message));
});
```

### Connection Management

Implement reconnection logic on the client:

```javascript
const socket = io('http://localhost:3001', {
  reconnection: true,
  reconnectionDelay: 1000,
  reconnectionAttempts: 5
});

socket.on('reconnect', (attemptNumber) => {
  console.log('Reconnected after', attemptNumber, 'attempts');
  // Re-join rooms and sync state
});
```

## Performance Optimization

1. **Message Batching**: Group multiple messages to reduce network overhead
2. **Compression**: Enable WebSocket compression for large payloads
3. **Rate Limiting**: Prevent spam and abuse
4. **Connection Pooling**: Reuse database connections

## Security Best Practices

- **Authentication**: Verify user identity before establishing WebSocket connection
- **Input Validation**: Sanitize all user inputs
- **Rate Limiting**: Implement per-user rate limits
- **CORS Configuration**: Restrict allowed origins

```javascript
io.use((socket, next) => {
  const token = socket.handshake.auth.token;
  if (isValidToken(token)) {
    next();
  } else {
    next(new Error('Authentication error'));
  }
});
```

## Conclusion

Building a real-time chat application with WebSockets is straightforward with the right tools. The combination of Socket.io and Redis provides a solid foundation for scalable real-time applications.

Key takeaways:
- WebSockets enable efficient bidirectional communication
- Redis pub/sub allows horizontal scaling
- Proper error handling and reconnection logic are essential
- Security should be a priority from the start

## Resources

- [Socket.io Documentation](https://socket.io/docs/)
- [WebSocket Protocol RFC](https://tools.ietf.org/html/rfc6455)
- [Redis Pub/Sub](https://redis.io/topics/pubsub)

---

*Have questions or suggestions? Feel free to reach out on [Twitter](https://twitter.com/ujjalsannyal) or [GitHub](https://github.com/ujjalsannyal)!*
