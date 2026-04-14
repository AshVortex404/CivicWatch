/**
 * Step-by-Step Deployment Guide for CivicWatch
 * 
 * --- BACKEND (Render.com) ---
 * 1. Create a "Web Service" on Render.
 * 2. Connect your GitHub and select the 'backend' folder (or root if using monorepo).
 * 3. Root Directory: backend
 * 4. Build Command: npm install
 * 5. Start Command: node server.js
 * 6. Add Environment Variables:
 *    - MONGO_URI: (Your Atlas URI)
 *    - JWT_SECRET: (A random string)
 *    - PORT: 5000
 * 
 * --- FRONTEND (Vercel.com) ---
 * 1. Create a new Project on Vercel.
 * 2. Select the 'frontend' folder.
 * 3. Framework Preset: Vite
 * 4. Add Environment Variable:
 *    - VITE_API_URL: (The URL Render gave you) + /api
 *      Example: https://civicwatch-api.onrender.com/api
 * 5. Deploy.
 */
