# ⚙️ Workflow Pipeline Orchestrator

<div align="center">

![Status](https://img.shields.io/badge/Status-Active-brightgreen?style=for-the-badge)
![FastAPI](https://img.shields.io/badge/FastAPI-Python-009688?style=for-the-badge&logo=fastapi)
![React](https://img.shields.io/badge/React-Flow-61DAFB?style=for-the-badge&logo=react)
![n8n](https://img.shields.io/badge/n8n-Webhook-EA4B71?style=for-the-badge&logo=n8n)

**A full-stack visual workflow automation system: design, validate and execute pipelines using a node-based interface.**

Inspired by **n8n** and **Apache Airflow**, built from scratch with a focus on DAG-based execution, reliability and system design.

[🎥 Watch Demo](https://drive.google.com/file/d/1yR2eBKbOcjJFWWRYophAGfVax0seq1s9/view?usp=sharing) · [🚀 Getting Started](#-getting-started) · [🔗 API Reference](#-api-reference)

</div>

---

## 📽️ Demo

> ▶️ [**Watch the full project walkthrough**](https://drive.google.com/file/d/1yR2eBKbOcjJFWWRYophAGfVax0seq1s9/view?usp=sharing)

The demo covers:
- Drag-and-drop node creation and pipeline design
- DAG validation with instant feedback
- Pipeline execution with a log for every node
- An n8n webhook triggering the same execution engine

---

## 🧠 What Is This?

Most workflow tools are black boxes. This project builds one from the ground up.

Users visually connect nodes to form a **Directed Acyclic Graph (DAG)**. The backend validates the graph, works out the execution order using **Kahn's topological sort**, and runs each node in sequence, passing the output of one node as the input to the next.

The result: a working mini-Airflow, built with React, FastAPI and n8n.

---

## 🏗️ System Architecture

```
┌──────────────────────────────────────────┐
│        Frontend (React + React Flow)     │
│   Drag-and-drop visual pipeline builder  │
└────────────────────┬─────────────────────┘
                     │ Workflow JSON (nodes + edges)
                     ▼
┌──────────────────────────────────────────┐
│             FastAPI Backend              │
│                                          │
│  ┌────────────────────────────────────┐  │
│  │  Validation Engine                 │  │
│  │  → Kahn's algorithm (cycle check)  │  │
│  └────────────────────────────────────┘  │
│                     ↓                    │
│  ┌────────────────────────────────────┐  │
│  │  Execution Engine                  │  │
│  │  → Topological order execution     │  │
│  │  → Node-to-node data passing       │  │
│  └────────────────────────────────────┘  │
│                     ↓                    │
│  ┌────────────────────────────────────┐  │
│  │  Logger                            │  │
│  │  → Per-node success/failure logs   │  │
│  └────────────────────────────────────┘  │
└──────────────────────────────────────────┘
                     ▲
         ┌───────────┴───────────┐
         │                       │
  Manual UI Trigger        n8n Webhook Trigger
  (Run Pipeline button)    (HTTP Request node)
```

---

## ⚡ Core Features

### 🎨 Visual Workflow Editor
- Drag nodes onto a canvas and draw connections to define the data flow
- Graph editing powered by **React Flow**, with state managed in **Zustand**
- **Submit** validates the pipeline; **Run** executes it and shows a log for every node

### 🧩 8 Node Types

| Node | What it does |
| :--- | :--- |
| **Input** | Injects starting data into the pipeline |
| **Text** | Holds a text template; each `{{variable}}` you type adds an input handle to the node |
| **Transform** | Changes text: uppercase, lowercase, reverse or trim |
| **Math** | Arithmetic on the incoming value: add, subtract, multiply, divide (safe divide-by-zero) |
| **API** | Calls a real external REST API (GET or POST) and passes the JSON response on |
| **Condition** | Checks whether a value equals a target and flags the result (`condition_passed`) |
| **Database** | Simulated database query, a stand-in for a real connection |
| **Output** | Captures and returns the final result |

Most nodes are built on a shared **BaseNode** component that renders fields and connection handles from a simple config, so adding a new node type takes only a few lines.

### 🔍 DAG Validation Engine
- Detects cycles using **Kahn's topological sorting algorithm**
- Blocks invalid pipelines before execution (`400: Pipeline contains a cycle`)
- Returns node count, edge count and DAG validity

### 🔥 Pipeline Execution Engine

Runs nodes in topological order. Data flows automatically from node to node.

```
Input("Hello World!")  →  Transform(uppercase)  →  Output("HELLO WORLD!")
```

Each node runs in its own error boundary: if one node fails, it is logged as `failed` and the rest of the pipeline keeps running.

### ⚡ Two Trigger Modes

**Mode 1: Manual UI**
```
User clicks "Run Pipeline"
        ↓
Frontend POSTs the workflow JSON
        ↓
POST /pipelines/execute
        ↓
Execution log shown for every node
```

**Mode 2: n8n Webhook**
```
n8n Webhook node (trigger)
        ↓
n8n HTTP Request node
        ↓
POST http://127.0.0.1:8000/pipelines/execute
        ↓
Same execution engine, log returned to n8n
```

Both modes share the **same backend execution engine**.

---

## 🔗 API Reference

| Method | Endpoint | Description |
| :--- | :--- | :--- |
| `GET` | `/` | Health check |
| `POST` | `/pipelines/parse` | Validate the DAG structure |
| `POST` | `/pipelines/execute` | Execute the pipeline node by node |

### Example request

```json
POST /pipelines/execute

{
  "nodes": [
    { "id": "1", "type": "input",     "position": { "x": 0,   "y": 0 }, "data": { "value": "Hello World!" } },
    { "id": "2", "type": "transform", "position": { "x": 200, "y": 0 }, "data": { "operation": "uppercase" } },
    { "id": "3", "type": "output",    "position": { "x": 400, "y": 0 }, "data": {} }
  ],
  "edges": [
    { "id": "e1", "source": "1", "target": "2" },
    { "id": "e2", "source": "2", "target": "3" }
  ]
}
```

### Example response

```json
{
  "status": "completed",
  "num_nodes": 3,
  "num_edges": 2,
  "execution_log": [
    { "node_id": "1", "node_type": "input",     "status": "success", "output": { "output": "Hello World!" } },
    { "node_id": "2", "node_type": "transform", "status": "success", "output": { "output": "HELLO WORLD!" } },
    { "node_id": "3", "node_type": "output",    "status": "success", "output": { "output": { "output": "HELLO WORLD!" }, "final": true } }
  ]
}
```

---

## 🚀 Getting Started

### Prerequisites
- Python 3.9+
- Node.js 16+
- n8n (optional, for the webhook trigger): `npm install -g n8n`

### 1. Clone the repo

```bash
git clone https://github.com/Mahiisss/Workflow-Pipeline-Orchestrator.git
cd Workflow-Pipeline-Orchestrator
```

### 2. Start the backend

```bash
cd backend
python -m venv venv

# Windows
venv\Scripts\activate
# macOS/Linux
source venv/bin/activate

pip install -r requirements.txt
uvicorn main:app --reload
```

| | URL |
| :--- | :--- |
| 🖥️ Backend | http://localhost:8000 |
| 📖 API docs | http://localhost:8000/docs |

### 3. Start the frontend

```bash
cd frontend
npm install
npm start
```

| | URL |
| :--- | :--- |
| 🌐 Frontend | http://localhost:3000 |

### 4. Start n8n (optional, for the webhook trigger)

```bash
n8n start
```

Open http://localhost:5678 and create a workflow:
1. Add a **Webhook** node and set the method to POST
2. Add an **HTTP Request** node that POSTs to `http://127.0.0.1:8000/pipelines/execute`
3. Pass your pipeline JSON as the request body

---

## 🛠️ Tech Stack

| Layer | Technology |
| :--- | :--- |
| Frontend | React, React Flow, Zustand |
| Backend | FastAPI, Pydantic, httpx, Uvicorn |
| Automation | n8n (webhook trigger) |
| Algorithms | Kahn's topological sort, DAG traversal |

---

## 🧠 What This Project Demonstrates

- **Graph theory in practice:** DAG modelling, cycle detection, topological ordering
- **Execution engine design:** node-to-node data passing with isolated error handling per node
- **Event-driven architecture:** the same engine triggered by the UI and by external webhooks
- **Component architecture:** a reusable BaseNode pattern shared across node types
- **Full-stack integration:** React ↔ FastAPI ↔ n8n working together end to end

---

## ⚠️ Known Limitations

- The **Condition** node checks equality and flags the result, but does not yet route data down separate true/false paths
- `{{variables}}` in the **Text** node create input handles in the editor, but are not yet filled in during execution
- The **Database** node is simulated
- When a node has more than one incoming connection, it uses the output of the last one
- The **API** node supports GET and POST

---

## 🔮 Roadmap

- [ ] True/false branching for the Condition node
- [ ] Variable substitution for Text node templates
- [ ] Workflow scheduling (cron-style triggers)
- [ ] Retries and failure recovery per node
- [ ] Persistent storage (PostgreSQL / MongoDB)
- [ ] Authentication and user workspaces
- [ ] LLM nodes (prompting, embeddings, vector search)

---

<div align="center">

Built with ⚡ by [Mahi](https://github.com/Mahiisss)

</div>
