# 🏥 AI-Powered Hospital Email Assistant

An AI-powered assistant that streamlines routine hospital email communication by generating draft responses for administrative queries while intelligently routing sensitive cases to hospital staff.

## 📖 Overview

Hospital appointment desks spend considerable time responding to repetitive emails related to appointments, doctor availability, test preparation, insurance enquiries, and document requests.

This project automates those routine interactions by retrieving information from an approved hospital knowledge base and preparing draft responses for staff review, allowing teams to focus on higher-value patient interactions.

## ⚙️ How It Works

1. Reads incoming emails using the Gmail API.
2. Identifies the user's intent.
3. Retrieves relevant information from the hospital knowledge base stored in Google Drive.
4. Generates a context-aware draft using the Groq LLM.
5. Creates a Gmail draft for staff review.

For requests involving medical advice, report interpretation, legal concerns, privacy-related issues, or queries outside the knowledge base, the system flags the email for manual handling instead of generating a response.

## 🏗️ Architecture

```text
Incoming Email
      │
 Gmail API
      │
 Intent Detection
      │
Hospital Knowledge Base
(Google Drive)
      │
 Groq LLM
      │
 Draft Response
      │
 Human Review
```

## 🛠️ Tech Stack

- Java
- Gmail API
- Google Drive API
- Groq API
- GitHub

## ✨ Key Features

- Automated email processing
- Intent classification
- Knowledge-grounded response generation
- Gmail draft creation
- Intelligent escalation for sensitive queries
- Human-in-the-loop approval workflow

## 🎥 Live Demo

Watch the project in action here:

## 👩‍💻 Author

**Biya Rocky**

📧 Passionate about Data Analytics, AI & Business Intelligence.

- **LinkedIn:** https://www.linkedin.com/in/your-linkedin-profile/

If you found this project interesting, feel free to connect with me on LinkedIn.
