# DDS — Distributed Database and Storage System
### Full Build Specification for AI Code Generation

> **Purpose of this document:** This is a complete, self-contained specification you can hand to an AI coding assistant (Claude, Claude Code, ChatGPT, etc.) in a single prompt to generate the **entire DDS application** — frontend and backend — end to end. It contains the architecture, tech stack, folder structure, API contracts, data models, security rules, build order, and acceptance tests. Nothing here should need to be re-explained mid-build.

---

## 0. Prompt to Paste to the AI

> Use this exact instruction when handing the spec to a coding AI:

```
You are building a complete, production-quality educational project called DDS
(Distributed Database and Storage System), described in full below.

Follow the spec exactly. Do not skip sections. Do not use placeholder or
"TODO" code for core functionality. Generate real, working, complete code for
every file listed in the folder structures. Follow the Development Order
(Section 15) step by step, and only declare the project complete once it
passes the Final Acceptance Test (Section 20).

Ask me only if something is truly ambiguous — otherwise make the most
sensible engineering decision and proceed.
```

---

## 1. Project Overview

DDS is an **educational / prototype distributed storage platform** that simulates a distributed database and storage system using GitHub repositories as logical storage nodes.

| Layer | Technology |
|---|---|
| Frontend | React (Create React App, **not** Vite), JavaScript, JSX, Material UI |
| Backend | Node.js + Express.js |
| Auth | Firebase Authentication (Email/Password + Google) |
| Storage nodes | GitHub repositories, accessed via GitHub REST API |
| Token custody | GitHub Personal Access Token — **backend only, never sent to the frontend** |

Firestore and Firebase Storage are **not** used — Firebase is for authentication only.

The system must demonstrate, functionally and visibly in the UI:
- Distributed storage across multiple nodes
- Deterministic data sharding
- Configurable replication
- Node management and health monitoring
- File metadata and version history (via Git commits)
- Fault tolerance and automatic recovery
- Audit logging of every operation

This is explicitly a **prototype/demo**, not a production replacement for a real database or object store — the README and UI copy should make this clear.

---

## 2. System Architecture

```
                         DDS WEB APPLICATION
                                  |
                    +-------------+-------------+
                    |                           |
                React JSX                 Firebase Auth
                    |                           |
              Create React App          Google / Email
                    |                           |
                    +-------------+-------------+
                                  |
                             REST API
                                  |
                         Node.js + Express
                                  |
                    +-------------+-------------+
                    |          DDS CORE          |
                    |                            |
                    | Node Manager               |
                    | Sharding Engine            |
                    | Replication Engine         |
                    | Metadata Manager           |
                    | Hashing Engine             |
                    | Fault Recovery             |
                    | Health Monitoring          |
                    | Audit Logger               |
                    +-------------+-------------+
                                  |
                 +----------------+----------------+
                 |                |                |
                 v                v                v
             GitHub Node 01   GitHub Node 02   GitHub Node 03
                 |                |                |
                 +----------------+----------------+
                                  |
                         Distributed Storage
```

**Golden rule:** React never talks to GitHub directly. Every storage operation flows `Frontend → Backend → DDS Engine → GitHub Service → GitHub API`.

```
WRONG:                          CORRECT:
React                           React
  |                                |
GitHub Token                    Express Backend
  |                                |
GitHub API                      GitHub Token
                                    |
                                 GitHub API
```

---

## 3. Technology Stack

### Frontend
```
React
Create React App
JavaScript (no TypeScript)
JSX
Material UI (@mui/material, @mui/icons-material)
React Router
Firebase Authentication (client SDK)
Axios
```
Do **not** use: Vite, Next.js, TypeScript, Redux (unless truly necessary — prefer Context + hooks).

### Backend
```
Node.js
Express.js
JavaScript
GitHub REST API (via Axios or Octokit-style calls)
Firebase Admin SDK
Multer (file uploads)
CORS
dotenv
Helmet
express-rate-limit
crypto (built-in, for SHA-256)
```

---

## 4. Authentication

Firebase Authentication is used **only** for identity — no Firestore, no Firebase Storage, no Realtime Database.

Enabled providers:
- Email / Password (with **mandatory email verification** before accessing protected DDS functionality)
- Google Sign-In

### Email/Password Flow
```
Register → Firebase creates account → Verification email sent →
User verifies email → Login → Firebase ID Token →
Backend verifies token → DDS API access granted
```

### Google Flow
```
Google Sign-In → Firebase Authentication → Firebase ID Token →
Backend verifies token → DDS Dashboard
```

### Backend Token Verification
```
Authorization: Bearer <FIREBASE_ID_TOKEN>
        |
Express middleware extracts token
        |
firebase-admin verifyIdToken()
        |
req.user populated
        |
Route handler executes
```

All protected routes must run through this middleware. The backend must always derive the user's identity from the verified token — **never trust a user ID sent from the frontend.**

---

## 5. GitHub Storage Architecture

Create three (or more) GitHub repositories acting as logical storage nodes:

```
dds-node-01
dds-node-02
dds-node-03
```

Each node repository layout:

```
dds-node-01/
├── metadata/
│   ├── files.json
│   └── node.json
├── storage/
│   ├── documents/
│   ├── images/
│   └── archives/
├── transactions/
├── versions/
├── health/
└── README.md
```

Prefer keeping metadata co-located with each node plus a lightweight system-wide index, rather than a separate `dds-metadata` repo (optional, not required).

### Node Configuration (backend)

```js
const nodes = [
  { id: "node-01", repository: "dds-node-01", branch: "main", enabled: true },
  { id: "node-02", repository: "dds-node-02", branch: "main", enabled: true },
  { id: "node-03", repository: "dds-node-03", branch: "main", enabled: true }
];
```

All node logic must be centralized in a `NodeManager` — never hard-coded elsewhere.

---

## 6. Security Requirements (Non-Negotiable)

- The GitHub Personal Access Token lives **only** in the backend `.env`.
- It is **never** returned in any API response, logged, or placed in a frontend environment variable.
- `.env` files are never committed (must be `.gitignore`d, with `.env.example` provided).
- Apply the principle of least privilege when generating the GitHub PAT.
- Backend errors must never leak tokens, private keys, or stack traces to the client.

```env
# server/.env — backend only
GITHUB_TOKEN=github_pat_xxxxxxxxxxxxxxxxx
GITHUB_OWNER=your-github-username-or-organization
```

---

## 7. DDS Core Engine

Backend service modules (one responsibility each):

```
NodeManager          — register/remove/enable/disable/health/list nodes
ShardingEngine        — deterministic file→node assignment
ReplicationEngine     — replica creation, verification, repair
MetadataManager       — CRUD for file metadata, lookup by hash/owner
StorageManager        — high-level store/read/update/delete/find, UI-agnostic
GitHubService         — the ONLY module that talks to the GitHub API
HashService           — SHA-256 generation & verification
HealthMonitor         — periodic/on-demand node health checks
RecoveryManager        — detect failure, serve replicas, rebuild, rebalance
AuditLogger           — structured logs for every operation
```

### 7.1 Node Manager

Responsibilities: register, remove, enable, disable, check, list active nodes, list healthy nodes, get statistics.

```
GET /api/nodes
GET /api/nodes/health
GET /api/nodes/:nodeId
```

Example node response:
```json
{
  "id": "node-01",
  "repository": "dds-node-01",
  "status": "online",
  "enabled": true
}
```

### 7.2 Sharding Engine (deterministic — never random)

```
fileId → SHA-256 → numeric hash → hash % activeNodeCount → selected node
```

```js
// ShardingEngine.js
getNodeForFile(fileId)
calculateShard(fileId)
getShardDistribution()
```

A given file ID should resolve to the same primary node consistently, unless a deliberate rebalance occurs.

### 7.3 Replication Engine

Default `replicationFactor = 2` (configurable via env).

```json
{
  "fileId": "file_001",
  "primaryNode": "node-01",
  "replicas": ["node-02", "node-03"],
  "replicationFactor": 3
}
```

```js
// replicationEngine.js
replicateFile()
removeReplica()
verifyReplica()
repairReplica()
getReplicaStatus()
```

All replication operations must go through `GitHubService` — no direct GitHub calls elsewhere.

### 7.4 Fault Tolerance & Recovery

```
File requested → Primary node OFFLINE → DDS checks replica metadata →
Replica found on Node 02 → File served from Node 02
```

```js
// RecoveryManager.js
detectFailedNode()
findReplicas()
serveReplica()
rebuildMissingReplica()
markNodeUnhealthy()
recoverNode()
```

Replica repair flow:
```
Primary → Replica missing → Read primary → Create replica →
Verify hash → Mark replica healthy
```

Primary-down flow:
```
Primary unavailable → Find replica → Read replica → Serve file
```

### 7.5 Node Rebalancing

```
Node 02 disabled → Find files assigned to Node 02 →
Select new healthy node → Replicate files → Update metadata
```
No data should ever be silently lost when a node is disabled.

### 7.6 Health Monitoring

Track per node: `nodeId, repository, status, lastChecked, responseTime, fileCount, storageUsed, errorCount`.

Status values: `ONLINE | OFFLINE | DEGRADED | DISABLED | UNKNOWN`

```
GET /api/nodes/health
```

---

## 8. File Model & Operations

### File Metadata Schema

```json
{
  "id": "file_001",
  "ownerId": "firebase-user-id",
  "name": "report.pdf",
  "originalName": "report.pdf",
  "mimeType": "application/pdf",
  "size": 245760,
  "hash": "sha256...",
  "primaryNode": "node-01",
  "replicas": ["node-02"],
  "path": "storage/documents/file_001.pdf",
  "createdAt": "2026-09-26T00:00:00.000Z",
  "updatedAt": "2026-09-26T00:00:00.000Z",
  "version": 1
}
```

Use generated IDs (e.g. `file_01J...`), never rely on raw filenames as identifiers.

### Upload Flow

```
Dashboard → Upload → Select file → POST /api/files/upload
   ↓
Authenticate user → Validate file → Generate file ID →
SHA-256 hash → Select primary node → Select replica nodes →
Upload to GitHub → Write metadata → Create commit → Return result
```

Response:
```json
{
  "success": true,
  "file": {
    "id": "file_001",
    "name": "report.pdf",
    "size": 245760,
    "primaryNode": "node-01",
    "replicas": ["node-02"],
    "hash": "sha256..."
  }
}
```

### File Operation Routes

```
POST   /api/files/upload
GET    /api/files
GET    /api/files/:id
GET    /api/files/:id/download
PUT    /api/files/:id
DELETE /api/files/:id
GET    /api/files/:id/versions
POST   /api/files/:id/restore
```

### Versioning (via Git history)

```
Version 1 → Commit A
Version 2 → Commit B
Version 3 → Commit C
```

```
GET /api/files/:id/versions
```
```json
[
  { "version": 3, "commit": "abc123", "message": "Updated file", "date": "2026-09-26T10:00:00Z" },
  { "version": 2, "commit": "def456", "message": "Updated file", "date": "2026-09-25T10:00:00Z" }
]
```

### Hashing & Duplicate Detection

- SHA-256 computed from actual file contents.
- Used for: integrity verification, duplicate detection, download validation.
- If a file with an identical hash already exists for the same user, show: *"A file with identical content already exists."* — do not auto-overwrite or auto-delete.

### Storage Limits & Validation

```env
MAX_FILE_SIZE_MB=25
```
Validate: file size, MIME type, filename, extension. Respect GitHub API/repo size constraints — this is a prototype, not unlimited object storage.

---

## 9. Authorization & Roles

- Every file has an `ownerId`, always derived from the verified Firebase token — never trusted from the client.
- Regular users may only upload/download/delete/rename/view **their own** files.
- Roles: `user`, `admin` (admin flag configurable on the backend for this prototype).

Admin can additionally: view all nodes, view all file metadata, view operations, enable/disable nodes, run health checks, view system statistics.

---

## 10. Audit Logging

Log every significant operation:

```json
{
  "id": "operation_001",
  "userId": "firebase-user-id",
  "type": "UPLOAD",
  "fileId": "file_001",
  "nodeId": "node-01",
  "status": "SUCCESS",
  "timestamp": "2026-09-26T00:00:00Z"
}
```

Operation types: `UPLOAD, DOWNLOAD, DELETE, UPDATE, REPLICATE, RECOVER, NODE_FAILURE, NODE_RECOVERY, LOGIN`

### Operation Status Machine

```
PENDING → PROCESSING → SUCCESS | FAILED | PARTIAL
```
Example: primary succeeds, replica fails → status `PARTIAL`, and the UI must clearly surface this.

---

## 11. API Reference (Full)

Base URL: `/api`

```
GET    /api/health

POST   /api/files/upload
GET    /api/files
GET    /api/files/:id
GET    /api/files/:id/download
PUT    /api/files/:id
DELETE /api/files/:id
GET    /api/files/:id/versions
POST   /api/files/:id/restore

GET    /api/nodes
GET    /api/nodes/health
GET    /api/nodes/:id
POST   /api/nodes/:id/health-check
POST   /api/nodes/:id/enable
POST   /api/nodes/:id/disable

GET    /api/operations
GET    /api/operations/:id

GET    /api/statistics
```

### Standard Response Shapes

Success:
```json
{ "success": true, "data": {}, "message": "Operation successful" }
```

Error:
```json
{ "success": false, "data": null, "message": "Operation failed", "code": "ERROR_CODE" }
```

Never expose GitHub tokens, private keys, or internal stack traces in any response.

---

## 12. Frontend Structure

### Routes

```
/
/auth/sign-in
/auth/sign-up
/auth/forgot-password
/auth/verify-email

/dashboard
/files
/files/:id
/nodes
/nodes/:id
/operations
/profile
/settings
```

Unauthenticated users redirect to `/auth/sign-in`. Include a 404 page.

### Folder Structure

```
client/
├── public/
└── src/
    ├── components/
    │   ├── common/
    │   ├── layout/
    │   ├── dashboard/
    │   ├── files/
    │   └── nodes/
    ├── pages/
    │   ├── Home.jsx
    │   ├── Dashboard.jsx
    │   ├── Files.jsx
    │   ├── FileDetails.jsx
    │   ├── Nodes.jsx
    │   ├── Operations.jsx
    │   ├── Profile.jsx
    │   └── Settings.jsx
    ├── auth/
    │   ├── SignIn.jsx
    │   ├── SignUp.jsx
    │   ├── ForgotPassword.jsx
    │   └── VerifyEmail.jsx
    ├── firebase/
    │   └── firebase.js
    ├── services/
    │   ├── api.js
    │   ├── authService.js
    │   ├── fileService.js
    │   └── nodeService.js
    ├── hooks/
    ├── context/
    │   └── AuthContext.jsx
    ├── routes/
    │   ├── AppRoutes.jsx
    │   └── ProtectedRoute.jsx
    ├── utils/
    ├── App.jsx
    └── index.js
```

### Frontend API Service Pattern

```
Firebase currentUser → getIdToken() → Axios Authorization header → Express
```
Attach the token automatically in an Axios interceptor — don't copy tokens manually into every component.

### State Management

Use React Context + hooks + local component state. Avoid Redux unless genuinely necessary. `AuthContext` holds auth state.

### UX Requirements

- Every network call shows loading feedback (e.g. "Uploading...", "Checking node...", "Replicating...") and disables buttons while in flight.
- Empty states: *"No files found. Upload your first file to start using DDS."* / *"No storage nodes configured."*
- Confirmation dialogs for destructive actions: delete file, disable node, remove replica.

### UI Design Requirements (Material UI)

Professional, minimal, clean, responsive (desktop + tablet + mobile), no excessive gradients or unnecessary animation, good spacing, accessible controls, clear typography. Use `AppBar, Drawer, Cards, Tables, Dialogs, Tabs, Chips, Buttons, Snackbar, CircularProgress, LinearProgress`.

Navigation: sidebar on desktop, bottom navigation on mobile. Sections: Dashboard, Files, Nodes, Operations, Profile.

### Key Pages

**Dashboard** — cards for Total Files, Total Storage, Active Nodes, Offline Nodes, Replication Status, Total Operations, Failed Operations.

**Node Dashboard** — Node ID, Repository, Status, Branch, File Count, Storage Used, Response Time, Last Health Check, Replication Count; actions: Enable, Disable, Health Check, View Files.

**Storage Explorer** — search, sort, filter, upload, download, delete, rename, file details, node info, replication info, version history. Columns: Name, Type, Size, Primary Node, Replica Nodes, Version, Created, Updated, Status.

**File Details** — File name, ID, Owner, MIME type, Size, SHA-256, Primary node, Replica nodes, Version, Created/Updated dates. Actions: Download, Delete, Rename, Restore Version.

**Home / Landing Page** — explains DDS ("Distributed storage built on Git-based infrastructure"), covers Distributed Nodes, Data Sharding, Replication, Versioning, Fault Tolerance, Secure Authentication. CTAs: Get Started, Sign In.

---

## 13. Backend Structure

```
server/
├── src/
│   ├── config/
│   │   ├── env.js
│   │   ├── firebaseAdmin.js
│   │   └── github.js
│   ├── controllers/
│   │   ├── authController.js
│   │   ├── fileController.js
│   │   ├── nodeController.js
│   │   └── operationController.js
│   ├── routes/
│   │   ├── fileRoutes.js
│   │   ├── nodeRoutes.js
│   │   └── operationRoutes.js
│   ├── middleware/
│   │   ├── authMiddleware.js
│   │   ├── errorMiddleware.js
│   │   └── uploadMiddleware.js
│   ├── services/
│   │   ├── github/
│   │   │   └── githubService.js
│   │   └── dds/
│   │       ├── nodeManager.js
│   │       ├── shardingEngine.js
│   │       ├── replicationEngine.js
│   │       ├── storageManager.js
│   │       ├── metadataManager.js
│   │       ├── recoveryManager.js
│   │       ├── healthMonitor.js
│   │       ├── hashService.js
│   │       └── auditLogger.js
│   ├── utils/
│   ├── app.js
│   └── server.js
└── package.json
```

### GitHub Service (single point of contact with GitHub)

```js
getRepository()
getFile()
createFile()
updateFile()
deleteFile()
listDirectory()
getCommits()
getBranches()
```
No controller or engine may call the GitHub API directly — always through this service.

### Storage Manager
```js
storeFile()
readFile()
updateFile()
deleteFile()
findFile()
```
Must remain UI-agnostic.

### Metadata Manager
```js
createMetadata()
getMetadata()
updateMetadata()
deleteMetadata()
findByHash()
findByOwner()
```

### Git Commit Message Convention

```
DDS: Upload file file_001
DDS: Update file file_001
DDS: Delete file file_001
DDS: Create replica for file_001
DDS: Recover file_001
```

### GitHub API Considerations to Handle Gracefully

Rate limits, repository permissions, file size limits, API errors, network failures, auth failures, repo/branch not found, SHA conflicts. Do not assume unlimited storage.

### Security Middleware

Helmet, CORS, rate limiting, input validation, auth middleware, authorization checks, file validation, size limits, secure env handling, sanitized error output. Never log tokens, private keys, or passwords.

---

## 14. Environment Variables

### Frontend (`client/.env`) — public config only

```env
REACT_APP_FIREBASE_API_KEY=
REACT_APP_FIREBASE_AUTH_DOMAIN=
REACT_APP_FIREBASE_PROJECT_ID=
REACT_APP_FIREBASE_APP_ID=
```

Never put `GITHUB_TOKEN`, `FIREBASE_ADMIN_PRIVATE_KEY`, or `FIREBASE_ADMIN_CLIENT_EMAIL` here.

### Backend (`server/.env`)

```env
PORT=5000
CLIENT_URL=http://localhost:3000

GITHUB_TOKEN=
GITHUB_OWNER=

GITHUB_NODE_01_REPO=dds-node-01
GITHUB_NODE_02_REPO=dds-node-02
GITHUB_NODE_03_REPO=dds-node-03
GITHUB_BRANCH=main

REPLICATION_FACTOR=2
MAX_FILE_SIZE_MB=25
```
Store Firebase Admin credentials securely on the backend (service account JSON or equivalent env vars).

---

## 15. Development Order (Build Sequentially)

1. Create the complete project structure (client + server).
2. Implement Express backend skeleton.
3. Implement GitHub service.
4. Implement Firebase Admin authentication middleware.
5. Implement Node Manager.
6. Implement hashing (SHA-256).
7. Implement sharding.
8. Implement file upload to a single GitHub node.
9. Implement file download.
10. Implement metadata.
11. Implement multiple nodes.
12. Implement replication.
13. Implement fault recovery.
14. Implement health monitoring.
15. Implement React authentication (sign-up, sign-in, verify, forgot password, Google).
16. Implement dashboard.
17. Implement storage explorer.
18. Implement node dashboard.
19. Implement operations dashboard.
20. Implement responsive design.
21. Implement security hardening.
22. Implement tests.
23. Write documentation (README, API docs, setup guides).

---

## 16. Installation & Scripts

### Frontend

```bash
npx create-react-app client
cd client
npm install @mui/material @emotion/react @emotion/styled @mui/icons-material react-router-dom firebase axios
```

```json
{
  "scripts": {
    "start": "react-scripts start",
    "build": "react-scripts build",
    "test": "react-scripts test",
    "eject": "react-scripts eject"
  }
}
```

### Backend

```bash
mkdir server && cd server
npm init -y
npm install express cors dotenv axios multer helmet express-rate-limit firebase-admin
npm install --save-dev nodemon
```

```json
{
  "scripts": {
    "start": "node src/server.js",
    "dev": "nodemon src/server.js"
  }
}
```

---

## 17. Firebase Setup Checklist

1. Create a Firebase project.
2. Register a web app.
3. Enable Email/Password sign-in method.
4. Enable Google sign-in method.
5. Configure authorized domains.
6. Configure email verification templates.
7. Copy the Firebase web config into `client/.env`.
8. Generate a service account and configure Firebase Admin SDK on the backend.

## 18. GitHub Setup Checklist

1. Create/choose a GitHub account or organization.
2. Create repos: `dds-node-01`, `dds-node-02`, `dds-node-03`.
3. Generate a GitHub Personal Access Token with only the scopes actually required (repo contents read/write).
4. Store the token only in `server/.env`.
5. Never expose it to the frontend or commit it.
6. Route all GitHub calls through `githubService.js`.

---

## 19. Testing Requirements

Cover with backend tests:
```
Authentication middleware
Node selection
Hashing
Sharding
Replication
File metadata
GitHub service
Fault recovery
Authorization
```

Scenarios to test:
```
User uploads file
User downloads file
User deletes file
User uploads duplicate
Node failure
Replica recovery
Unauthorized access
Invalid Firebase token
GitHub API failure
```

---

## 20. Final Acceptance Test

The project is complete only when this end-to-end flow works:

```
Open DDS → Sign Up → Email verification → Sign In → Dashboard →
Upload file → Firebase auth verified → Express API →
SHA-256 generated → Primary node selected →
File stored in GitHub Node 01 → Metadata created →
Replica created in Node 02 → Dashboard shows successful upload →
User downloads file → DDS finds primary/replica →
File returned → SHA-256 verified → Download succeeds
```

Then, fault tolerance:
```
Node 01 OFFLINE → User requests file → DDS detects primary unavailable →
DDS finds Node 02 replica → File returned from Node 02 →
System reports successful fallback
```

Then, recovery:
```
Node 01 recovered → DDS health check → Node marked ONLINE →
Missing replica repaired → Hash verified → System returns to healthy state
```

---

## 21. README Contents (Required Deliverable)

The generated README must include:
```
Project Overview          Firebase Setup           API Documentation
Architecture               Google Auth Setup        DDS Algorithm
Features                   GitHub Token Setup        Sharding Explanation
Technology Stack           GitHub Repository Setup   Replication Explanation
Folder Structure           Environment Variables     Fault Tolerance
Installation               Running Frontend          Security
                            Running Backend           Limitations / Future Work
```

---

## 22. Code Quality Rules

- Write real, complete, working code — no `// TODO` or `// implement later` for core functionality.
- Every imported function must exist; every route must have a controller; every controller must have a service.
- Avoid circular dependencies.
- Use `async/await` consistently with proper `try/catch` error handling.
- Keep all secrets out of source code — `.env.example` files provided for both client and server.
- Keep frontend and backend fully separate deployable applications.
- No god-files: don't put all backend logic in `server.js` or all frontend logic in `App.jsx` — use the modular structure above.

---

## 23. Hard Constraints Recap (Do Not Violate)

1. Create React App only — no Vite, no Next.js.
2. JavaScript/JSX only — no TypeScript.
3. Firebase Auth only — no Firestore, no Firebase Storage, no Realtime DB.
4. Email/password **and** Google auth supported; email verification required for password accounts.
5. GitHub token never leaves the backend.
6. All GitHub calls go through `githubService.js` on the backend.
7. Sharding is deterministic (hash-based), never random.
8. Replication factor is configurable (default 2).
9. Every file has SHA-256 integrity verification.
10. Users can only access their own files; ownership always derived from the verified Firebase token.
11. Consistent API response envelope (`success/data/message/code`) across all endpoints.
12. Responsive, professional Material UI — desktop, tablet, and mobile.
