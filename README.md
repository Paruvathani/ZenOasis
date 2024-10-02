mkdir emotional-chatbot
cd emotional-chatbot
npm init -y
npm install express socket.io mongoose dotenv twilio axios
emotional-chatbot/
├── .env
├── Dockerfile
├── server.js
├── models/
│   └── Message.js
├── routes/
│   └── chat.js
└── config/
    └── db.js
const mongoose = require('mongoose');

const messageSchema = new mongoose.Schema({
  user: { type: String, required: true },
  message: { type: String, required: true },
  timestamp: { type: Date, default: Date.now }
});

module.exports = mongoose.model('Message', messageSchema);
const mongoose = require('mongoose');
require('dotenv').config();

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGO_URI, {
      useNewUrlParser: true,
      useUnifiedTopology: true
    });
    console.log('MongoDB Connected');
  } catch (error) {
    console.error(error.message);
    process.exit(1);
  }
};

module.exports = connectDB;
const express = require('express');
const http = require('http');
const socketio = require('socket.io');
const mongoose = require('mongoose');
const Message = require('./models/Message');
const connectDB = require('./config/db');
const axios = require('axios');
const twilio = require('twilio');
require('dotenv').config();

const app = express();
const server = http.createServer(app);
const io = socketio(server);

// Twilio API Setup
const twilioClient = twilio(process.env.TWILIO_SID, process.env.TWILIO_AUTH_TOKEN);

// Connect to MongoDB
connectDB();

app.use(express.json());

// Socket.io Chat Functionality
io.on('connection', (socket) => {
  console.log('User connected');

  socket.on('sendMessage', async (messageData) => {
    const { user, message } = messageData;

    // Save message to MongoDB
    const newMessage = new Message({ user, message });
    await newMessage.save();

    // Analyze emotional tone using Gemini API (replace with actual endpoint)
    const emotionalResponse = await axios.post('https://gemini-api.com/analyze', {
      text: message
    }, {
      headers: { Authorization: `Bearer ${process.env.GEMINI_API_KEY}` }
    });

    const emotionalTone = emotionalResponse.data.emotion;
    io.emit('receiveMessage', { user, message, emotionalTone });

    // Send SMS via Twilio if distressed
    if (emotionalTone === 'distressed') {
      await twilioClient.messages.create({
        body: `User ${user} seems distressed. Message: ${message}`,
        from: process.env.TWILIO_PHONE_NUMBER,
        to: process.env.NOTIFICATION_PHONE_NUMBER
      });
    }
  });

  socket.on('disconnect', () => {
    console.log('User disconnected');
  });
});

const PORT = process.env.PORT || 5000;
server.listen(PORT, () => console.log(`Server running on port ${PORT}`));
MONGO_URI=mongodb+srv://<your-mongodb-connection-string>
GEMINI_API_KEY=your_gemini_api_key
TWILIO_SID=your_twilio_sid
TWILIO_AUTH_TOKEN=your_twilio_auth_token
TWILIO_PHONE_NUMBER=your_twilio_phone_number
NOTIFICATION_PHONE_NUMBER=your_target_phone_number
npx create-react-app emotional-chatbot-frontend
cd emotional-chatbot-frontend
npm install socket.io-client axios
import React, { useState, useEffect } from 'react';
import io from 'socket.io-client';

const socket = io('http://localhost:5000');

const Chat = () => {
  const [message, setMessage] = useState('');
  const [messages, setMessages] = useState([]);

  useEffect(() => {
    socket.on('receiveMessage', (data) => {
      setMessages((prevMessages) => [...prevMessages, data]);
    });
  }, []);

  const sendMessage = () => {
    const user = 'Student'; // Replace with dynamic user data
    socket.emit('sendMessage', { user, message });
    setMessage('');
  };

  return (
    <div>
      <h2>Emotional Support Chat</h2>
      <div className="chat-box">
        {messages.map((msg, index) => (
          <div key={index}>
            <strong>{msg.user}:</strong> {msg.message} <em>({msg.emotionalTone})</em>
          </div>
        ))}
      </div>
      <input
        type="text"
        value={message}
        onChange={(e) => setMessage(e.target.value)}
        placeholder="Type your message..."
      />
      <button onClick={sendMessage}>Send</button>
    </div>
  );
};

export default Chat;
import React from 'react';
import './App.css';
import Chat from './Chat';

function App() {
  return (
    <div className="App">
      <Chat />
    </div>
  );
}

export default App;
FROM node:14

WORKDIR /app

COPY package.json /app/package.json
RUN npm install

COPY . /app

EXPOSE 5000

CMD ["npm", "start"]
docker build -t emotional-chatbot .
docker run -p 5000:5000 emotional-chatbot
