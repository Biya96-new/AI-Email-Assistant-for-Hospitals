# 🏥 AI-Powered Hospital Email Assistant

An AI-powered email assistant that helps hospitals streamline routine administrative email enquiries using **Retrieval-Augmented Generation (RAG)** and **Large Language Models (LLMs)**.

The assistant drafts responses for common administrative requests while identifying emails that require human review. Every response is saved as a **draft**, ensuring hospital staff remain in complete control before anything is sent.

---

## ✨ Features

- 📧 Reads incoming emails from Gmail
- 🧠 Identifies the intent of patient enquiries
- 📚 Retrieves relevant information from a hospital knowledge base using RAG
- ✍️ Drafts context-aware responses for routine administrative emails
- 🚩 Flags sensitive or complex emails for human review
- 📨 Creates Gmail drafts instead of sending emails automatically
- ⚙️ Supports workflow automation through GitHub Actions

---

## 🛠️ Tech Stack

- Python
- Groq LLM API
- Gmail API
- Google Drive API
- Google OAuth 2.0
- Retrieval-Augmented Generation (RAG)
- Prompt Engineering
- GitHub Actions

---

## 🚀 Getting Started

### 1. Clone the repository

```bash
git clone https://github.com/Biya96-new/AI-Email-Assistant-for-Hospitals.git
cd AI-Email-Assistant-for-Hospitals
```

### 2. Install dependencies

Python

```bash
pip install -r requirements.txt
```

Node (if applicable)

```bash
npm install
```

### 3. Configure Environment Variables

This repository does **not** include API keys or OAuth credentials.

Create a `.env` file using the provided `.env.example` file and add your own credentials.

Example:

```env
GROQ_API_KEY=your_groq_api_key

GOOGLE_CLIENT_ID=your_google_client_id

GOOGLE_CLIENT_SECRET=your_google_client_secret

GOOGLE_REFRESH_TOKEN=your_google_refresh_token

GOOGLE_DRIVE_FOLDER_ID=your_google_drive_folder_id

GMAIL_USER=your_email@example.com
```

> **Note:** This project requires your own API keys and OAuth credentials. It will not run until valid credentials are configured.

---

## 📖 Development Process

This project was developed using an AI-assisted workflow.

1. Defined the project requirements in a **Product Requirements Document (PRD)**.
2. Converted the PRD into a phased implementation plan.
3. Implemented each phase using **Google Antigravity**.
4. Iteratively tested and refined the solution.

The repository includes both the **PRD** and the **Implementation Plan** to document the engineering process.

---

## 🎥 Project Demo

Watch the project in action:

> **📹 Demo:** *[Add your LinkedIn / YouTube / Google Drive video link here](https://www.linkedin.com/posts/biya-rocky-dataanalyst_digitalhealth-healthcareinnovation-hospitalmanagement-ugcPost-7485921814838149120-ck1d/?utm_source=share&utm_medium=member_desktop&rcm=ACoAACeEowYByIUu1ymdXneEMbHii85a1lJyZSo)*

## 💡 Key Takeaway

One of the biggest lessons from this project was:

> **Building an AI agent isn't just about teaching it what to do,it's equally about defining what it should never do.**

---

## 📄 License

This project is licensed under the **MIT License**.
