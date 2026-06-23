# passport-ai-assistant
🛂 PassportAI – AI-Powered Passport Form Assistant

PassportAI is a modern AI-powered web application that automates passport-style application form filling using OCR, Large Language Models (LLMs), and a conversational AI agent. Instead of manually entering personal information, users simply upload an identity document such as an Aadhaar Card, PAN Card, Passport, or Driving License. The system extracts relevant information, verifies it through an AI assistant, and automatically populates the passport application form.

The application combines Tesseract.js OCR for on-device text extraction with Claude Sonnet 4.6 for intelligent document understanding and structured data extraction. An interactive AI agent guides users throughout the process, asks follow-up questions for missing information, verifies low-confidence fields, maintains conversation history, and enables manual corrections before final submission.

✨ Features
🤖 Conversational AI passport assistant
📄 OCR-based text extraction using Tesseract.js
🧠 AI-powered document understanding with Claude Sonnet 4.6
📑 Automatic extraction of Name, DOB, Gender, Address, Nationality, ID Number, and more
⚡ Intelligent form auto-filling
📊 Confidence score for extracted fields
✏️ User verification and editable extracted data
💬 ChatGPT-style conversation interface
📷 Drag-and-drop image upload
🔒 Browser-based OCR for improved privacy
📝 Automatic passport form completion
📱 Responsive and modern UI
🔄 Regex-based fallback extraction when AI parsing is unavailable
🛠️ Tech Stack

Frontend

HTML5
CSS3
JavaScript (ES6)

AI & OCR

Claude Sonnet 4.6 (LLM)
Tesseract.js (OCR)
Prompt Engineering
Structured JSON Extraction

Concepts Demonstrated

Optical Character Recognition (OCR)
Large Language Models (LLMs)
AI Agents
Prompt Engineering
Information Extraction
Human-in-the-Loop Verification
Confidence-Based Validation
Form Automation
Conversational AI
🔄 Workflow
User Uploads Identity Document
            │
            ▼
      Tesseract.js OCR
            │
            ▼
      Raw Text Extraction
            │
            ▼
 Claude Sonnet 4.6 Analysis
            │
            ▼
 Structured JSON Generation
            │
            ▼
 Confidence Verification
            │
            ▼
 AI Conversation & User Confirmation
            │
            ▼
 Automatic Passport Form Filling
            │
            ▼
      Final Submission
🎯 Learning Outcomes

This project demonstrates the practical implementation of Generative AI, AI Agents, OCR, Document Intelligence, LLM Integration, Prompt Engineering, Structured Data Extraction, and Human-in-the-Loop AI Systems to automate real-world government form-filling workflows.

This description is suitable for a portfolio project and clearly communicates the technologies, architecture, and AI concepts used.
