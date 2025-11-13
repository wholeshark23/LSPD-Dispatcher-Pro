// CAD-MDT Full-stack scaffold (single-file preview for GitHub)
// This code file contains a project README, architecture, example backend snippets (Express + Socket.IO + Discord.js),
// database schema, Docker compose, GitHub Actions workflow, and a React frontend scaffold (Tailwind + shadcn/ui) with
// example components: Login, Dashboard, Radio, LEO Portal, Fire/Rescue Portal, DMV Portal.

/*
README.md (included below)

-- README START --
# CAD-MDT (GitHub Scaffold)

Opinionated starter kit for a multi-department CAD/MDT with:
- Discord integration (bot + OAuth linking)
- In-app radios using Socket.IO rooms (channels)
- Role-based portals: LEO, Fire/Rescue, DMV
- Unit assignment automation hooks
- PostgreSQL database + Prisma ORM
- Auth: JWT + refresh tokens + role-based access
- Docker-compose for local dev
- GitHub Actions for CI

## Tech stack
- Frontend: React (Vite) + TypeScript + TailwindCSS + shadcn/ui
- Backend: Node.js + Express + TypeScript + Socket.IO
- DB: PostgreSQL + Prisma
- Discord integration: discord.js + OAuth2 link flow
- Real-time: Socket.IO for events and radio voice metadata (signaling)

## High-level architecture
- Frontend (React) connects to backend REST and a Socket.IO namespace `/ws`.
- User auth via HTTP login (JWT). Socket.IO attaches JWT on connect.
- Radio channels map to Socket.IO rooms (e.g. `radio:leo`, `radio:fire`, `radio:ems`, `radio:dispatch`).
- Voice is handled off-app (recommended) via WebRTC; this scaffold implements signaling channels.
- Discord bot listens for events, can post incidents, and accepts commands (assign unit, create call).
- DMV portal: CRUD on vehicle registrations and driving records.

## Quick start (dev)
1. Clone repo
2. Copy `.env.example` to `.env` and fill variables
3. `docker-compose up --build` (starts Postgres + Redis + backend + frontend)
4. `pnpm install` in `/web` and `/server` then `pnpm dev`

-- README END --


// ---------- Example file: server/src/index.ts ----------
/*
This is a brief server example showing Express + Socket.IO + Discord integration skeleton.
*/

import express from 'express';
import http from 'http';
import { Server } from 'socket.io';
import { PrismaClient } from '@prisma/client';
import jwt from 'jsonwebtoken';
import Discord from 'discord.js';

const app = express();
const server = http.createServer(app);
const io = new Server(server, { path: '/ws' });
const prisma = new PrismaClient();

app.use(express.json());

// JWT auth middleware example
function authMiddleware(req, res, next) {
  const token = req.headers.authorization?.replace('Bearer ', '');
  if (!token) return res.status(401).send({ error: 'Unauthorized' });
  try {
    const payload = jwt.verify(token, process.env.JWT_SECRET);
    req.user = payload;
    next();
  } catch (err) {
    return res.status(401).send({ error: 'Invalid token' });
  }
}

// Example REST: create incident
app.post('/api/incidents', authMiddleware, async (req, res) => {
  const { type, location, priority } = req.body;
  const incident = await prisma.incident.create({ data: { type, location, priority, createdById: req.user.id } });
  // broadcast to dispatch namespace
  io.of('/dispatch').emit('incident:new', incident);
  return res.json(incident);
});

// Socket.IO auth using JWT token in query
io.use((socket, next) => {
  const token = socket.handshake.auth?.token;
  if (!token) return next(new Error('Authentication error'));
  try {
    const payload = jwt.verify(token, process.env.JWT_SECRET);
    socket.data.user = payload;
    next();
  } catch (err) {
    next(new Error('Authentication error'));
  }
});

// Namespaces: /radio, /dispatch, /dmv
const radioNs = io.of('/radio');
radioNs.on('connection', (socket) => {
  const user = socket.data.user;
  // join default channels based on role
  const roles = user.roles || [];
  if (roles.includes('leo')) socket.join('radio:leo');
  if (roles.includes('fire')) socket.join('radio:fire');
  if (roles.includes('ems')) socket.join('radio:ems');

  socket.on('radio:join', ({ channel }) => {
    socket.join(`radio:${channel}`);
    socket.emit('radio:joined', { channel });
  });

  // signaling for WebRTC (offer/answer/ice)
  socket.on('webrtc:signal', ({ channel, payload }) => {
    // broadcast to room for peer-to-peer negotiation
    socket.to(`radio:${channel}`).emit('webrtc:signal', { from: user.id, payload });
  });
});

// Discord bot integration (minimal)
const discordClient = new Discord.Client({ intents: [Discord.GatewayIntentBits.Guilds, Discord.GatewayIntentBits.GuildMessages] });
discordClient.login(process.env.DISCORD_BOT_TOKEN);

discordClient.on('ready', () => {
  console.log('Discord bot ready', discordClient.user.tag);
});

// Listen for a command (slash command architecture should be used in production)
discordClient.on('messageCreate', async (message) => {
  if (message.author.bot) return;
  if (!message.content.startsWith('!')) return;
  const [cmd, ...rest] = message.content.slice(1).split(' ');
  if (cmd === 'incident') {
    // create public incident in CAD
    const incident = await prisma.incident.create({ data: { type: rest.join(' '), location: 'Discord', priority: 'low' } });
    // push to web clients
    io.of('/dispatch').emit('incident:new', incident);
    message.reply(`Incident created: ${incident.id}`);
  }
});

server.listen(process.env.PORT || 4000, () => console.log('Server listening'));


// ---------- Example: prisma/schema.prisma ----------
/*
model User {
  id        Int      @id @default(autoincrement())
  discordId String?  @unique
  username  String
  email     String?  @unique
  roles     String[]
  createdAt DateTime @default(now())
}

model Incident {
  id        Int      @id @default(autoincrement())
  type      String
  location  String
  priority  String
  status    String   @default("open")
  createdBy  User?   @relation(fields: [createdById], references: [id])
  createdById Int?
  createdAt DateTime @default(now())
}

model Vehicle {
  id         Int @id @default(autoincrement())
  plate      String @unique
  vin        String
  ownerId    Int
}
*/


// ---------- Docker Compose (snippet) ----------
/*
version: '3.8'
services:
  db:
    image: postgres:15
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: example
    volumes:
      - db-data:/var/lib/postgresql/data
  redis:
    image: redis:alpine
  server:
    build: ./server
    env_file: .env
    depends_on: [db, redis]
  web:
    build: ./web
    env_file: .env
    ports: ["3000:3000"]
volumes:
  db-data:
*/


// ---------- Frontend (React) example components ----------
// /web/src/App.jsx (minimal)
import React from 'react';
import { BrowserRouter, Routes, Route, Link } from 'react-router-dom';
import Login from './components/Login';
import Dashboard from './components/Dashboard';
import Radio from './components/Radio';
import LEO from './components/LEOPortal';
import FirePortal from './components/FirePortal';
import DMV from './components/DMVPortal';

export default function App() {
  return (
    <BrowserRouter>
      <div className="p-4">
        <nav className="mb-4">
          <Link to="/">Dashboard</Link> | <Link to="/radio">Radio</Link> | <Link to="/leo">LEO</Link> | <Link to="/fire">Fire</Link> | <Link to="/dmv">DMV</Link>
        </nav>
        <Routes>
          <Route path="/" element={<Dashboard/>} />
          <Route path="/login" element={<Login/>} />
          <Route path="/radio" element={<Radio/>} />
          <Route path="/leo" element={<LEO/>} />
          <Route path="/fire" element={<FirePortal/>} />
          <Route path="/dmv" element={<DMV/>} />
        </Routes>
      </div>
    </BrowserRouter>
  );
}

// /web/src/components/Radio.jsx (Socket.IO + WebRTC signaling UI)
import React, { useEffect, useState, useRef } from 'react';
import { io } from 'socket.io-client';

export default function Radio(){
  const [channels] = useState(['dispatch','leo','fire','ems']);
  const [joined, setJoined] = useState(null);
  const socketRef = useRef(null);

  useEffect(()=>{
    const token = localStorage.getItem('token');
    const s = io('/radio', { auth: { token } , path: '/ws' });
    socketRef.current = s;
    s.on('connect', ()=> console.log('radio connected'));
    s.on('webrtc:signal', ({ from, payload }) => {
      console.log('signal', from, payload);
      // handle peer signaling (offer/answer/ice) -> integrate RTCPeerConnection
    });
    return ()=> s.disconnect();
  },[]);

  function join(channel){
    socketRef.current.emit('radio:join', { channel });
    setJoined(channel);
  }

  return (
    <div>
      <h2>Radio</h2>
      <div className="flex gap-2">
        {channels.map(c => <button key={c} onClick={()=>join(c)} className="border p-2">{c}</button>)}
      </div>
      {joined && <div className="mt-4">Joined channel: {joined}</div>}
    </div>
  );
}

// /web/src/components/LEOPortal.jsx (summary)
export default function LEOPortal(){
  return (<div><h2>LEO Portal</h2><p>CAD calls, assigned units, warrants DB, suspect DB, arrests, RMS links</p></div>);
}

export function FirePortal(){
  return (<div><h2>Fire / Rescue Portal</h2><p>Apparatus status, station delta, hydrant map, turnout times</p></div>);
}

export function DMVPortal(){
  return (<div><h2>DMV Portal</h2><p>Vehicle registration, suspensions, plate lookups, PDFs and forms</p></div>);
}


// ---------- Auth & Discord OAuth notes ----------
/*
- Provide endpoints for /auth/discord/link and /auth/discord/callback to link user accounts.
- Store discordId on user record. Allow linking via code or OAuth flow.
- In Discord bot, provide slash commands that admins can use to push incidents or query unit status.
*/

// ---------- Unit assignment automation (hook example) ----------
/*
When incident created, run function assignUnits(incident):
- Query available units by status in DB (e.g., "available", within X distance)
- Reserve units, set status to "enroute" and notify radios and Discord
- Example: io.of('/radio').to('radio:dispatch').emit('unit:assigned', { incidentId, units });
*/

// ---------- GitHub Actions (ci.yml) ----------
/*
name: CI
on: [push, pull_request]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v2
        with:
          version: 8
      - run: pnpm install --filter ./server && pnpm install --filter ./web
      - run: pnpm --filter ./server build
      - run: pnpm --filter ./web build
*/

// ---------- Security & privacy notes ----------
/*
- Do NOT transmit audio or PII to third-party services without consent.
- For voice, prefer WebRTC P2P or a self-hosted SFU (Jitsi, Janus, mediasoup) instead of piping audio through Socket.IO.
- Rate-limit Discord interactions and validate commands.
- Use prepared statements and Prisma to avoid SQL injection.
*/

// ---------- Next steps included in repo ----------
/*
- Full TypeScript conversion for server and web
- Add unit tests, E2E tests for critical flows
- Implement WebRTC peer connections and an optional SFU integration guide
- Implement permissions matrix UI and role management
- Create Discord slash commands and register them via Discord API
*/

// End of scaffold file
