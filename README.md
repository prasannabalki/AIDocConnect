# 🩺 AIDocConnect

**AIDocConnect** is a next-generation **telemedicine platform** that integrates **Generative AI (GenAI)** with **clinical decision support** to bridge the gap between patients, physicians, and health data.


<p align="center">
  <img src="a1.png" width="720" alt="AIDocConnect Logo"/>
</p>

<h1 align="center">🩺 AIDocConnect</h1>
<h3 align="center">AI-Powered Telemedicine & Clinical Decision Support Platform</h3>

---

## 🌍 Overview

An integrated GenAI-powered telemedicine ecosystem that en63ables:

- 🩹 Real-time doctor–patient consultations  
- 🧾 Automated clinical documentation  
- 🧠 AI-assisted diagnostics (text, image, and audio)  
- 📱 Cross-device access via web and mobile apps  
633
---

## 🏗️ System Architecture3

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

------------------------------------------------------------------

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


| **Folder / File**        | **Description**                                                        |
| ------------------------ | ---------------------------------------------------------------------- |
| **`aidocconnect/`**      | 🩺 *Root directory for the AIDocConnect Telemedicine + GenAI platform* |
| ├── **`frontend/`**      | Contains all client-facing interfaces (web + mobile)                   |
| │ ├── `react-portal/`    | Web portal for doctors & admins (React.js + Next.js)                   |
| │ └── `flutter-app/`     | Mobile app for patients & doctors (Flutter)                            |
| ├── **`backend/`**       | Core backend microservices                                             |
| │ ├── `node-api/`        | Node.js backend (Express/NestJS REST & GraphQL APIs)                   |
| │ └── `auth-service/`    | Authentication service (Keycloak / JWT)                                |
| ├── **`ai-services/`**   | All AI-related microservices and model APIs                            |
| │ ├── `llm-engine/`      | Generative AI models (BioGPT, Llama3, Mistral)                         |
| │ ├── `speech-service/`  | Speech AI (Whisper/Vosk ASR + Coqui TTS)                               |
| │ └── `imaging-ai/`      | Imaging AI (MONAI / OpenCV diagnostics)                                |
| ├── **`database/`**      | Databases and schema definitions                                       |
| │ ├── `postgres/`        | SQL schema, migrations, and FHIR data models                           |
| │ └── `mongodb/`         | NoSQL collections (chat logs, AI responses, metadata)                  |
| ├── **`deployment/`**    | Deployment automation and environment setup                            |
| │ ├── `docker/`          | Dockerfiles & Compose for local development                            |
| │ └── `kubernetes/`      | Helm charts / YAMLs for production deployment on K8s                   |
| └── **`docs/`**          | Project documentation and planning                                     |
|    ├── `architecture.md` | System design, diagrams, and API mappings                              |
|    └── `roadmap.md`      | Feature roadmap, milestones, and sprint planning                       |
