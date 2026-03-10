# Enterprise Contact Management System

> This repository is not just a project record; it serves as a foundational knowledge base for my personal "Digital Twin"—documenting the architecture decisions, engineering trade-offs, and lessons learned while building an enterprise-grade CRM from scratch in 45 days.

---

## 🏗️ Core Engineering Challenges & Solutions

Rather than just building CRUD features, this system was designed to resolve deep-rooted operational bottlenecks for an engineering consulting firm managing 3,000+ contacts and 2,000+ projects.

### 1. Refactoring a PoC into a Production AI Pipeline (LINE Bot)
Field engineers needed a way to digitize business cards instantly. I took over an experimental Google Apps Script PoC built by a colleague, completely refactored it, and integrated it into our new Python (FastAPI) + Firestore architecture. 
- Transformed raw GPT-4o Vision OCR outputs into highly structured JSON (18 fields).
- Engineered a robust sanitization layer specifically tuned for Taiwan's complex address and phone number formats.
- Implemented a pre-write duplicate detection shield, effectively cutting entry time from minutes down to seconds per card.

### 2. Resolving "Schrödinger's Duplicates" with a Traceable Merge Engine
Decades of disjointed Excel entry had polluted the database with duplicate entities (e.g., "John Doe" vs. "John Doe (Manager)").
- Built a multi-entity merge engine that automatically categorizes field conflicts vs. inheritable data.
- Handled complex relationship transfers (Project-to-Contact) during the merge.
- Implemented soft-deletion and strict audit logging, adhering strictly to the **[Data Cleaning with Traceability](https://github.com/gurry0927/data-cleaning-with-traceability)** design pattern to ensure no historical data is permanently lost.

### 3. Taming Unstructured Data: The Excel Import Engine
Working alongside a colleague who designed the 4-step frontend UI wizard, I built the heavy-lifting backend engine.
- Engineered algorithms to aggressively parse and normalize heterogeneous Excel data (e.g., splitting `"04-2213-8848 #851\n0918-774-526"` into respective semantic fields).
- Automated the creation of N:M association graphs between newly imported Contacts, Companies, and Projects.

### 4. Zero-Friction Web Console (Optimistic UI & Stateful Interceptors)
To provide a native-app feel within a React SPA, I bypassed traditional loading spinners.
- **Optimistic UI:** State updates are instantly reflected in the DOM while API requests resolve in the background.
- **Self-Healing Cache:** Engineered a global Axios interceptor that automatically purges "phantom records" from the frontend state if a 404 is returned, avoiding rigid page reloads.

---

## 🏛️ Architecture & Data Model

### The "Project-Centric" Relation Graph
Traditional CRMs enforce a rigid 1:1 relationship between a contact and a company. In the engineering and construction industry, one person frequently represents multiple sub-contractors or consulting firms depending on the specific project. 

I designed a **Project-Centric bridging model** where individuals map to specific corporate entities strictly within the scope of a given Project, accurately reflecting real-world dynamics.

![Project-Centric Architecture](./images/Project_Centric.png)

### System Architecture Snapshot
- **Frontend**: React 19, Ant Design 6, Zustand
- **Backend API**: FastAPI (Python 3.10+)
- **Primary Database**: Google Firestore (Selected over PostgreSQL for real-time reactivity and managed zero-maintenance overhead)
- **Security & Identity**: Firebase Authentication (OAuth), Role-Based Access Control (RBAC), Global Audit Logging
- **Infrastructure**: Docker Compose, Caddy Reverse Proxy, GCP VM

![System Architecture](./images/System_Architecture.png)

---

## ⚙️ Development Highlights & GitFlow Adoption

Building the features was only half the battle. Coming from a background of standalone scripts and rapid prototyping, I deliberately used this project to institutionalize enterprise engineering practices into my workflow.

I systematically audited my own codebase, identifying 78 architectural and logic debt items. I transitioned to a rigorous **Feature Branch / Pull Request workflow**, writing unit tests and utilizing squash-and-merge strategies to build a pristine commit history. 

| Metric | Value |
|--------|-------|
| Development period | 45 days |
| Total commits | 630+ |
| Scope | Full-stack (API, SPA, Bot) |

---

## 📖 Deep Dive & Documentation

Want to see the thought process behind these architecture decisions? Check out my detailed Chinese documentation:

- 🧱 **[Technical Architecture & Implementation Details](./docs/Enterprise專案技術思維與實作細節.md)** (My Tech Lead perspective & decision matrix)
- 🎯 **[Engineering Case Studies / STAR Method](./docs/Enterprise專案面試_STAR案例集.md)** (Deep dives into how I solved specific business pain points)
- 🔍 **[Code Review & GitFlow Experience](./docs/Code_Review_and_GitFlow_Experience.md)** (My transition to structured CI/CD workflows)

> *Note: This is a private enterprise system. Architecture diagrams and technical descriptions are shared for portfolio and educational purposes only.*

---
*(Screenshots for reference)*

### Intelligent Merge Engine
![Intelligent Merge Engine](./images/Intelligent_Merge_Engine.png)

### Smart Import Wizard (Frontend UI design by colleague)
![Smart Import Wizard](./images/Smart_Import_Wizard.png)

### Comprehensive Audit Logging
![Audit Log](./images/Cover_AuditLog_JSON.jpeg)
