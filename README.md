# 🍎 Poly Apple Game

Welcome to the **Poly Apple** root workspace. This directory serves as a container for the decoupled microservice architecture of the Poly Apple game. 

The game has been split into two entirely independent projects to ensure scalability, ease of deployment, and clean code management.

## 🏗 Project Architecture

This workspace contains two main folders, each acting as its own standalone repository:

1. **[`/frontend`](./frontend/README.md):** 
   - The Client-Side application where the actual gameplay happens.
   - Built using **Vanilla JavaScript, HTML5 Canvas, and Vite**.
   - Handles the visual rendering, graphing mathematical equations, and collision detection.

2. **[`/backend`](./backend/README.md):**
   - The Authoritative Server and Real-time Engine.
   - Built using **Node.js, Express, and Socket.io**.
   - Handles matchmaking, room lifecycles, and real-time state synchronization between players and spectators to prevent cheating.

*(Please navigate into each folder to read their specific `README.md` for detailed technical stacks and logic).*

## 🚀 How to Run the Full Project Locally

Because the project is decoupled, you will need to run the Frontend and Backend on two separate terminal instances simultaneously.

**Terminal 1: Start the Backend Server**
```bash
cd backend
npm install
npm run dev
```
*The backend API and Websocket server will start on `http://localhost:3000`.*

**Terminal 2: Start the Frontend UI**
```bash
cd frontend
npm install
npm run dev
```
*Vite will start the client on `http://localhost:5173`. Open this URL in your browser to play.*

## 🌐 Deployment Strategy
- **Frontend:** Should be deployed to a static CDN provider like **Vercel** or **Netlify**.
- **Backend:** Should be deployed to a Node.js hosting service like **Render.com** or **Railway.app**. 
*(Remember to update the `BACKEND_URL` in `frontend/script.js` to point to your live backend URL before deploying).*
