{
  "name": "zenoasis-backend",
  "version": "1.0.0",
  "description": "Backend for ZenOasis Chatbot",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.17.1",
    "mongoose": "^5.11.15",
    "socket.io": "^4.0.0",
    "dotenv": "^8.2.0",
    "twilio": "^3.58.0"
  }
}
MONGODB_URI=your_mongodb_uri
TWILIO_ACCOUNT_SID=your_twilio_account_sid
TWILIO_AUTH_TOKEN=your_twilio_auth_token
TWILIO_PHONE_NUMBER=your_twilio_phone_number
const mongoose = require('mongoose');

const messageSchema = new mongoose.Schema({
  userId: String,
  message: String,
  timestamp: { type: Date, default: Date.now },
});

module.exports = mongoose.model('Message', messageSchema);
const express = require('express');
const mongoose = require('mongoose');
const http = require('http');
const socketIo = require('socket.io');
const dotenv = require('dotenv');
const Message = require('./models/Message');
const twilio = require('twilio');

dotenv.config();
const app = express();
const server = http.createServer(app);
const io = socketIo(server);

mongoose.connect(process.env.MONGODB_URI, { useNewUrlParser: true, useUnifiedTopology: true });

io.on('connection', (socket) => {
  console.log('New client connected');

  socket.on('sendMessage', async (data) => {
    const newMessage = new Message(data);
    await newMessage.save();
    io.emit('newMessage', newMessage);
  });

  socket.on('disconnect', () => {
    console.log('Client disconnected');
  });
});

server.listen(5000, () => {
  console.log('Server is running on port 5000');
});
{
  "name": "zenoasis-frontend",
  "version": "1.0.0",
  "description": "Frontend for ZenOasis Chatbot",
  "main": "index.js",
  "scripts": {
    "start": "react-scripts start",
    "build": "react-scripts build",
    "test": "react-scripts test"
  },
  "dependencies": {
    "react": "^17.0.2",
    "react-dom": "^17.0.2",
    "socket.io-client": "^4.0.0"
  },
  "eslintConfig": {
    "extends": [
      "react-app",
      "react-app/jest"
    ]
  },
  "browserslist": {
    "production": [
      ">0.2%",
      "not dead",
      "not op_mini all"
    ],
    "development": [
      "last 1 chrome version",
      "last 1 firefox version",
      "last 1 safari version"
    ]
  }
}
import React, { useEffect, useState } from 'react';
import io from 'socket.io-client';
import Chat from './components/Chat';

const socket = io('http://localhost:5000');

function App() {
  return (
    <div>
      <h1>ZenOasis Chatbot</h1>
      <Chat socket={socket} />
    </div>
  );
}

export default App;
import React, { useEffect, useState } from 'react';

function Chat({ socket }) {
  const [messages, setMessages] = useState([]);
  const [input, setInput] = useState('');

  useEffect(() => {
    socket.on('newMessage', (message) => {
      setMessages((prevMessages) => [...prevMessages, message]);
    });
  }, [socket]);

  const sendMessage = () => {
    socket.emit('sendMessage', { message: input });
    setInput('');
  };

  return (
    <div>
      <div>
        {messages.map((msg, index) => (
          <div key={index}>{msg.message}</div>
        ))}
      </div>
      <input 
        type="text" 
        value={input} 
        onChange={(e) => setInput(e.target.value)} 
        onKeyPress={(e) => e.key === 'Enter' ? sendMessage() : null} 
      />
      <button onClick={sendMessage}>Send</button>
    </div>
  );
}

export default Chat;
FROM node:14
WORKDIR /usr/src/app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 5000
CMD ["node", "server.js"]
FROM node:14
WORKDIR /usr/src/app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build
CMD ["npm", "start"]
version: '3'
services:
  backend:
    build: ./backend
    ports:
      - "5000:5000"
  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
