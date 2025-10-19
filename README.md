# 🩺 AIDocConnect

**AIDocConnect** is a next-generation **telemedicine platform** that integrates **Generative AI (GenAI)** with **clinical decision support** to bridge the gap between patients, physicians, and health data.

---

## 🌍 Overview

An integrated GenAI-powered telemedicine ecosystem that enables:

- 🩹 Real-time doctor–patient consultations  
- 🧾 Automated clinical documentation  
- 🧠 AI-assisted diagnostics (text, image, and audio)  
- 📱 Cross-device access via web and mobile apps  

---

## 🏗️ System Architecture

             ┌──────────────────────────┐
             │       Flutter App        │
             │   (Patient & Doctor)     │
             └──────────┬───────────────┘
                        │ REST / WebSocket
             ┌──────────┴───────────────┐
             │   React.js Web Portal    │
             │  (Admin / Clinician UI)  │
             └──────────┬───────────────┘
                        │
    ┌───────────────────┴───────────────────┐
    │          Node.js Backend (API)        │
    │ Express.js / NestJS / Socket.IO       │
    │ - Auth (JWT, Keycloak)                │
    │ - FHIR-compliant API (open source)    │
    │ - WebRTC signalling for live consults │
    │ - Queue (RabbitMQ / Redis)            │
    └───────────────────┬───────────────────┘
                        │ gRPC / REST
    ┌───────────────────┴───────────────────┐
    │         Python AI Microservices       │
    │ FastAPI / Flask + PyTorch + HuggingFace│
    │ - LLM (BioGPT, Llama3, Mistral)       │
    │ - Speech (Whisper, Coqui TTS)         │
    │ - Imaging (MONAI, OpenCV)             │
    │ - Time-Series (sktime, tsfresh)       │
    │ - ICD/CPT mapping (UMLS/ICD APIs)     │
    └───────────────────┬───────────────────┘
                        │
    ┌───────────────────┴───────────────────┐
    │     Database & Storage Layer           │
    │ PostgreSQL + MongoDB + MinIO (S3-like) │
    │ Redis cache + Elasticsearch index      │
    └───────────────────────────────────────┘


---

## 🧩 Tech Stack

| Layer | Tech Stack | Purpose |
|-------|-------------|----------|
| **Frontend (Web)** | 🧭 **React.js + Tailwind + Next.js** | Web interface for doctors/admins — dashboard, analytics, consultation panel |
| **Frontend (Mobile)** | 📱 **Flutter** | Cross-platform app for patients/doctors: video call, chat, health records |
| **Backend API** | 🟢 **Node.js (Express/NestJS)** | Core telemedicine logic — authentication, scheduling, user management, and FHIR APIs |
| **AI Microservices** | 🧠 **Python (FastAPI/Flask)** | Runs LLMs, NLP, imaging and speech models; communicates with Node.js over REST/gRPC |
| **Database** | 🗄️ **PostgreSQL + MongoDB** | Stores structured health records, AI logs, and unstructured data |
| **File Storage** | 🗂️ **MinIO** | S3-compatible storage for documents, images, voice notes |
| **Queue/Async** | 🔁 **RabbitMQ / Redis** | Asynchronous task queue between Node and Python services |
| **Authentication** | 🔐 **Keycloak / JWT** | Role-based access control, single sign-on |
| **Monitoring** | 📊 **Prometheus + Grafana + Loki** | Metrics, logs, and real-time service health monitoring |
| **Deployment** | ☸️ **Docker + Kubernetes (K8s)** | Scalable deployment with isolated microservices across pods |

---

## 🧠 AI Model Layer

| Function | Library / Model |
|-----------|----------------|
| Symptom triage & consultation | **Llama-3 / BioGPT / Mistral** via Hugging Face |
| Speech-to-text | **Whisper**, **Vosk** |
| Text-to-speech | **Coqui TTS**, **Piper** |
| Medical entity extraction | **scispaCy**, **MedCAT** |
| ICD-10 / SNOMED lookup | **UMLS Metathesaurus** (research license) |
| Imaging AI | **MONAI**, **BioMedCLIP**, **CheXNet** |
| Time-series anomaly detection | **sktime**, **Kats**, **PyTorch Forecasting** |
| Explainability | **SHAP**, **Grad-CAM**, **Captum** |

---

## 🚀 Development Roadmap

| Sprint | Deliverable | Tools |
|---------|--------------|-------|
| 1️⃣ | Flutter app + React portal + Node.js REST API skeleton | Flutter, React, Node |
| 2️⃣ | User auth, video consultation (WebRTC) | Node.js + Socket.IO + Flutter WebRTC plugin |
| 3️⃣ | Whisper ASR → SOAP note (Python service) | FastAPI + Whisper |
| 4️⃣ | ICD-10 code generator (LLM + UMLS lookup) | BioGPT / Llama-3 |
| 5️⃣ | Image diagnostic module (MONAI) | FastAPI + PyTorch |
| 6️⃣ | Dashboard + logs + AI explainability | React + Grafana |
| 7️⃣ | Deployment on Kubernetes (MinIO, PostgreSQL, MongoDB) | Docker + K8s |

---

## ☸️ Kubernetes Architecture (Mermaid Diagram)

```mermaid
flowchart TD
subgraph Frontend["🧭 Frontend Layer"]
  R[React.js Web Portal] 
  F[Flutter App]
end

subgraph Backend["🟢 Node.js Backend"]
  API[REST / GraphQL API]
  AUTH[Keycloak / JWT Auth]
  WS[WebRTC / Socket.IO]
  QUEUE[Redis / RabbitMQ]
end

subgraph AI["🧠 Python AI Microservices"]
  LLM[LLM Engine<br/>BioGPT / Llama3 / Mistral]
  ASR[Speech-to-Text<br/>Whisper / Vosk]
  IMG[Imaging AI<br/>MONAI / BioMedCLIP]
end

subgraph Data["💾 Data & Storage Layer"]
  DB[(PostgreSQL)]
  MDB[(MongoDB)]
  FS[(MinIO)]
  LOGS[(Grafana / Loki / Prometheus)]
end

subgraph K8s["☸️ Kubernetes Cluster"]
  N1[Node.js Pods]
  P1[Python AI Pods]
  D1[Database StatefulSets]
end

R --> API
F --> API
API --> AUTH
API --> DB
API --> MDB
QUEUE --> AI
AI --> DB
AI --> MDB
API --> FS
AI --> FS
API --> LOGS
AI --> LOGS

API -.deployed on.-> N1
AI -.deployed on.-> P1
DB -.StatefulSet.-> D1
MDB -.StatefulSet.-> D1

🔒 Compliance & Security

✅ HIPAA / GDPR / NDHM aligned
🔐 Data encryption (AES-256, TLS 1.3)
🧾 Audit trails via Loki
🧑‍⚕️ Role-based access for clinicians, patients, and admins

Future Enhancements

🌐 Federated learning for on-device personalization
🧬 Integration with genomics APIs
🤖 LLM fine-tuning for medical summarization
🩻 Edge-AI inference for diagnostic imaging

aidocconnect/
 ├── frontend/
 │   ├── react-portal/
 │   └── flutter-app/
 ├── backend/
 │   ├── node-api/
 │   └── auth-service/
 ├── ai-services/
 │   ├── llm-engine/
 │   ├── speech-service/
 │   └── imaging-ai/
 ├── database/
 │   ├── postgres/
 │   └── mongodb/
 ├── deployment/
 │   ├── docker/
 │   └── kubernetes/
 └── docs/
     ├── architecture.md
     └── roadmap.md
