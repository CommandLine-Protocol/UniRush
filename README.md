# UniRush

## Project Structure

- `/frontend` - React.js application
- `/backend` - Node.js & Express API

## Quick Start Guide

### 1. Backend Setup

Open a terminal and navigate to the backend directory:
```bash
cd backend
npm install
```

Start the development server (runs on port 5000):
```bash
npm run dev
```

### 2. Frontend Setup

Open a new terminal window and navigate to the frontend directory:
```bash
cd frontend
npm install
```

Start the React development server (typically runs on port 3000):
```bash
npm start
```

## Health Check

Once both servers are running, navigating to the frontend should display a green "success" banner confirming the frontend is successfully communicating with the `http://localhost:5000/api/health` endpoint.
