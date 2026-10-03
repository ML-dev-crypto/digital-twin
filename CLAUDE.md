# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

RuraLens is a village digital twin platform with:
- **Frontend**: React 18 + TypeScript + Vite + Tailwind CSS
- **Backend**: Node.js + Express + MongoDB
- **AI Features**: RAG (Retrieval-Augmented Generation) with Pathway MCP, GNN (Graph Neural Network) for infrastructure impact prediction
- **Mobile**: Capacitor for Android deployment
- **Key Technologies**: MapLibre GL for 3D maps, Zustand for state management, WebSocket for real-time data

The main application code is in `_RURALENS/` directory.

## Development Commands

### Frontend (run from `_RURALENS/`)
```bash
npm run dev              # Start Vite dev server (port 3000)
npm run build            # TypeScript compile + Vite production build
npm run preview          # Preview production build locally
npm run build:mobile     # Build + sync to Android Capacitor
npm run mobile:open      # Open Android Studio
npm run mobile:dev       # Build, sync, and open Android Studio
```

### Backend (run from `_RURALENS/backend/`)
```bash
npm start                # Start Express server (port 3001)
npm run dev              # Start with --watch flag (auto-reload)
npm run clear-db         # Clear MongoDB database
npm run cleanup-feedback # Clean up feedback collection
npm run export-pathway   # Export data to Pathway RAG system
npm run pathway-server   # Start Python Pathway API server
```

### GNN Service (run from `_RURALENS/backend/python-gnn/`)
```bash
python api_server.py     # Start GNN prediction API server
python gradio_app.py     # Launch interactive Gradio UI for testing
```

## Architecture

### Frontend Structure (`_RURALENS/src/`)
- **components/**: React components organized by feature
  - `Auth/`: Login pages (desktop + mobile variants)
  - `Dashboard/`: Admin/Citizen/Mobile dashboards, KPI cards, live charts
  - `Map3D/`: MapLibre GL 3D map with pipe layer visualization
  - `Rag/`: RAG query modal for AI-powered Q&A
  - `Views/`: Full-page views (Alerts, Analytics, Schemes, Impact Predictor, etc.)
  - `Sidebar/`: Navigation sidebar
  - `Layout/`: TopNav, StatusBar, MobileNav, MobileHeader
- **store/**: Zustand state management (`villageStore.ts`)
- **services/**: API client functions
- **hooks/**: Custom React hooks (e.g., `useRagQuery.ts`)
- **data/**: Demo/mock data (`demoVillageData.ts`)
- **utils/**: Helper functions (map highlighting, data formatters)
- **types/**: TypeScript type definitions

### Backend Structure (`_RURALENS/backend/`)
- **server.js**: Main Express server, WebSocket setup, route mounting
- **routes/**: API endpoints
  - `auth.js`: JWT authentication (login, register)
  - `schemes.js`: Government schemes CRUD
  - `rag.js`: RAG query endpoint with PII filtering, caching, citation enrichment
  - `gnn.js`: GNN impact prediction proxy to Python service
  - `anonymousReports.js`: Citizen feedback with image uploads (Multer)
  - `ringg.js`: Ringg AI integration for voice call handling
  - `llmStatus.js`: Check LLM service availability
- **models/**: MongoDB Mongoose schemas (Scheme, User, Feedback, Tank, Pump, Pipe, AnonymousReport, etc.)
- **utils/**: 
  - `dataGenerator.js`: Generates mock sensor data
  - `localLLMService.js`: Local LLM processing for feedback
  - `pathwayClient.js`: Pathway MCP API client
  - `piiSanitizer.js`: Redacts PII (Aadhaar, PAN, emails, phones)
  - `ragCache.js`: In-memory cache for RAG responses (120s TTL)
- **config/database.js**: MongoDB connection + seeding logic
- **python-gnn/**: PyTorch-based Graph Neural Network for cascading failure simulation

### State Management Pattern
Uses Zustand (`villageStore.ts`) for global state:
- Village infrastructure data (tanks, pumps, pipes, consumer clusters)
- Active view, sidebar/panel states
- Authentication (user, role, token)
- Real-time sensor data
- GNN graph nodes/edges for impact prediction

### API Communication
- Frontend communicates with backend via REST APIs (`/api/*`)
- Backend proxies requests to:
  - **Pathway MCP server** (Python/Rust RAG): `PATHWAY_MCP_URL` (default port 8000)
  - **GNN Python service**: `http://localhost:5000/api/gnn/predict-structured`
  - **MongoDB**: Stores schemes, users, reports, infrastructure graph
- WebSocket at `/ws` for real-time sensor updates (not yet fully implemented)

## Key Features & Workflows

### 1. RAG (AI Q&A System)
- **User Action**: Click "Ask AI" button → Modal opens → Type question → Get answer with citations
- **Backend Flow**: 
  1. `POST /api/rag-query` receives question
  2. PII sanitizer redacts sensitive data
  3. Check cache (120s TTL)
  4. Query Pathway MCP server at `PATHWAY_MCP_URL`
  5. Enrich citations with geo-coordinates (4-level fallback)
  6. Return answer + citations with map integration
- **Files**: `routes/rag.js`, `utils/pathwayClient.js`, `utils/piiSanitizer.js`, `components/Rag/RagQueryModal.tsx`

### 2. GNN Impact Predictor
- **User Action**: Navigate to "Village Analyzer" → Click infrastructure node → Select failure type/severity → Watch cascading impacts
- **Backend Flow**:
  1. Frontend sends node failure event to `POST /api/gnn/predict-structured`
  2. Backend proxies to Python GNN service
  3. GNN calculates impact propagation via graph edges
  4. Returns impact scores (0-100%) for connected nodes
  5. Frontend updates MapLibre GL visualization with color-coded nodes (green → yellow → orange → red)
- **Features**: Accumulated damage from multiple failures, right-click to add new nodes with auto-connections
- **Files**: `routes/gnn.js`, `python-gnn/api_server.py`, `components/Views/ImpactPredictorView.tsx`

### 3. Mobile Support (Capacitor)
- Detects native platform via `Capacitor.isNativePlatform()`
- Separate mobile components: `MobileLandingPage`, `MobileLoginPage`, `MobileDashboard`, `MobileNav`, `MobileHeader`
- Build mobile: `npm run build:mobile` → syncs to `android/` directory
- Mobile-specific views route to simplified UIs

### 4. Authentication
- JWT-based authentication with `jsonwebtoken`
- Login: `POST /api/auth/login` → returns `{ token, user: { name, email, role } }`
- Roles: `admin`, `citizen`, `field_worker`
- Default admin: `admin@village.com` / `admin123`
- Token stored in Zustand state + localStorage (for mobile persistence)

## Environment Variables

### Frontend (`.env` in `_RURALENS/`)
```
VITE_API_URL=http://localhost:3001  # Backend URL (auto-detected in dev)
```

### Backend (`.env` in `_RURALENS/backend/`)
- **Required**:
  - `MONGODB_URI`: MongoDB connection string
  - `JWT_SECRET`: Secret for JWT signing (change in production!)
  - `GEMINI_API_KEY`: Google Gemini API key for document analysis
- **RAG Configuration**:
  - `PATHWAY_MCP_URL`: Pathway server URL (default: `http://localhost:8000/v1/pw_ai_answer`)
  - `PATHWAY_MCP_TOKEN`: Auth token for Pathway
  - `RAG_CACHE_TTL_SECONDS`: Cache duration (default: 120)
  - `PATHWAY_TIMEOUT_MS`: Request timeout (default: 20000)
- **Optional**:
  - `RINGG_API_KEY`, `RINGG_WEBHOOK_TOKEN`: For voice call integration
  - `PORT`: Backend port (default: 3001)

## Common Development Patterns

### Adding a New View
1. Create component in `src/components/Views/MyNewView.tsx`
2. Add route in `src/App.tsx` switch statement
3. Add sidebar menu item in `src/components/Sidebar/Sidebar.tsx`
4. Update `villageStore.ts` if new state needed

### Adding a New API Endpoint
1. Create route file in `backend/routes/myNewRoute.js`
2. Define Express router with endpoints
3. Mount in `backend/server.js`: `app.use('/api/my-new', myNewRoutes);`
4. Create Mongoose model if new data type: `backend/models/MyModel.js`

### Working with the Map
- Map instance lives in `Map3D.tsx` using MapLibre GL
- To add markers/layers: Use MapLibre GL API in `useEffect` after map loads
- Infrastructure nodes are rendered as circle layers with click handlers
- Color-coding: Use map style expressions based on node `health` property (0-100)

### GNN Model Changes
- Model architecture: `python-gnn/infrastructure_gnn.py`
- Training: `python-gnn/train.py` (uses PyTorch Geometric)
- Fine-tuning: `python-gnn/fine_tune.py`
- Inference: `python-gnn/api_server.py` loads trained model and serves predictions
- Model weights: Stored in `python-gnn/saved_models/`

## Testing

### RAG Feature Test
```bash
cd _RURALENS/backend
powershell -File test-rag.ps1  # Tests login + RAG query
```

### GNN Demo
```bash
cd _RURALENS/backend
node demo-gnn.js  # Demonstrates GNN predictions with sample data
```

### Manual Testing
1. Start backend: `cd _RURALENS/backend && npm start`
2. Start frontend: `cd _RURALENS && npm run dev`
3. Visit `http://localhost:3000`
4. Login: `admin@village.com` / `admin123`
5. Test features: Dashboard → Ask AI, Village Analyzer → Trigger failures

## Troubleshooting

### Frontend won't build
- Check TypeScript errors: `npx tsc --noEmit`
- Clear Vite cache: `rm -rf node_modules/.vite`

### Backend can't connect to MongoDB
- Check `MONGODB_URI` in `.env`
- Ensure MongoDB is running: `mongod` or MongoDB Atlas connection
- Backend falls back to in-memory data if DB unavailable

### RAG queries fail
- Check Pathway server is running on `PATHWAY_MCP_URL`
- For quick testing, use mock server: `cd llm-app/templates/question_answering_rag && python mock_pathway_server.py`
- Check `PATHWAY_MCP_TOKEN` matches between backend and Pathway

### GNN predictions return errors
- Ensure Python GNN server is running: `cd backend/python-gnn && python api_server.py`
- Check Python dependencies: `pip install -r requirements.txt` (in python-gnn folder)
- GNN service runs on port 5000 by default

### Mobile build fails
- Run `npm run mobile:sync` to sync Capacitor
- Ensure Android SDK is installed for Android builds
- Check `capacitor.config.json` for correct paths

## Important Notes

- **Do not commit `.env` files** - Use `.env.example` as template
- **PII Protection**: All RAG queries auto-redact sensitive info before sending to LLM
- **Rate Limiting**: RAG endpoint has 10 queries/minute per user limit
- **Caching**: RAG responses cached for 120 seconds to reduce LLM API costs
- **Mobile Detection**: Use `Capacitor.isNativePlatform()` to detect mobile, not user agent
- **Map Performance**: Limit number of nodes rendered on map (<500 for smooth performance)
- **GNN Accuracy**: Model trained on synthetic data; production use requires retraining with real infrastructure data
- **Database Seeding**: Backend auto-seeds demo data on first run if collections are empty

## Project Context
This was built for a hackathon to demonstrate AI-powered rural infrastructure monitoring with:
- Real-time IoT sensor simulation
- Natural language Q&A over government documents
- Graph neural network for failure prediction
- 3D visualization of village infrastructure
- Mobile-first citizen engagement
