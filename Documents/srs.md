# Software Requirements Specification (SRS) for Interview AI

**Project Name**: Interview AI — GenAI Interview Preparation & ATS Resume Builder  
**Version**: 1.0.0  
**Status**: Release-Ready  
**Date**: November 2026  

---

## 1. Introduction

### 1.1 Purpose
This document specifies the software requirements for **Interview AI**, a full-stack Generative AI application engineered to help job seekers prepare for technical and behavioral interviews and generate tailored, ATS-compliant resumes based on specific job postings.

### 1.2 Scope
Interview AI bridges the gap between candidate qualifications and employer expectations. Candidates upload their existing PDF resumes and provide a target job description and self-description. Powered by **Google Gemini AI**, the system parses the resume in-memory, computes a profile match score, generates interview questions with evaluation criteria, detects critical skill gaps with severity ratings, plans a day-by-day study curriculum, and automatically compiles an ATS-optimized A4 PDF resume via headless **Puppeteer**.

### 1.3 Definitions, Acronyms, and Abbreviations
* **SRS**: Software Requirements Specification
* **ATS**: Applicant Tracking System
* **LLM / GenAI**: Large Language Model / Generative Artificial Intelligence (Google Gemini `gemini-3-flash-preview`)
* **JWT**: JSON Web Token
* **Puppeteer**: Headless Chromium browser automation library for PDF generation
* **Zod**: TypeScript/JavaScript schema declaration and validation library

---

## 2. Overall Description

### 2.1 Product Perspective
Interview AI operates on a modern, decoupled client-server architecture:
* **Frontend**: Single Page Application (SPA) built with React 19, React Router, Vite, and Sass.
* **Backend**: RESTful API server built with Node.js and Express 5, connecting to MongoDB via Mongoose.
* **External AI Engine**: Google GenAI API (`@google/genai`) providing structured JSON responses via Zod schemas.
* **Rendering Engine**: Headless Puppeteer compiling dynamically tailored HTML into print-ready A4 PDF resumes.

```
[ React 19 Frontend SPA ]
          │ (HTTP / JSON / Multipart + Cookies)
          ▼
[ Express 5 REST API Server ] ────▶ [ MongoDB Database (Mongoose) ]
          │
          ├──▶ [ pdf-parse Engine (In-Memory Buffer) ]
          ├──▶ [ Google Gemini GenAI (Structured Zod Output) ]
          └──▶ [ Headless Puppeteer (A4 PDF Compilation) ]
```

### 2.2 User Characteristics
* **Candidates / Job Seekers**: Users who register, upload resumes, receive personalized AI interview coaching, track report history, and download customized resumes.
* **System / Background Engine**: Automated processes handling in-memory PDF parsing, AI prompt orchestration, and headless PDF rendering.

### 2.3 Operating Environment
* **Server**: Node.js runtime environment (v18.0.0+) on Windows/Linux.
* **Database**: MongoDB 7.0+ (Local or MongoDB Atlas).
* **Browser Compatibility**: Chrome, Firefox, Safari, Edge (Evergreen modern browsers).
* **Client Device**: Responsive desktop and mobile web browsers.

---

## 3. Specific Requirements

### 3.1 Functional Requirements

#### Module 1: Authentication & User Management
* **REQ-1 (Registration)**: Candidates must be able to register with a unique `username`, unique `email`, and a `password`.
* **REQ-2 (Password Hashing)**: User passwords must be securely salted and hashed using `bcrypt` (10 salt rounds) prior to database persistence.
* **REQ-3 (Session Issuance)**: On successful registration or login, the server must issue a signed JWT token containing user ID and username, with a 24-hour expiration (`1d`), stored in an `httpOnly` cookie (`token`).
* **REQ-4 (Token Blacklisting)**: When logging out (`GET /api/auth/logout`), the current token must be persisted in the `blacklistTokens` collection and the client-side cookie must be cleared.
* **REQ-5 (Session Guard)**: Protected endpoints must pass through the `authUser` middleware, verifying the JWT signature against `JWT_SECRET` and asserting that the token does not exist in `blacklistTokens`.
* **REQ-6 (Current User Query)**: Authenticated users must be able to verify their session and retrieve user details via `GET /api/auth/get-me`.

#### Module 2: Resume Ingestion & Parsing
* **REQ-7 (In-Memory File Upload)**: The backend must process resume PDF uploads using `multer.memoryStorage()`, avoiding temporary disk writes to protect user privacy.
* **REQ-8 (Upload Constraints)**: The uploaded file must be restricted to a maximum size of 3 MB and must accept `.pdf` MIME formats.
* **REQ-9 (Text Extraction)**: The server must extract plaintext from the in-memory PDF buffer using `pdf-parse` without persisting raw files.

#### Module 3: GenAI Assessment & Report Generation
* **REQ-10 (Evaluation Request)**: Submitting `POST /api/interview/` requires candidate self-description, job description, and the uploaded resume buffer.
* **REQ-11 (Structured Gemini Integration)**: The backend must prompt Google Gemini (`gemini-3-flash-preview`) enforcing strict JSON schema output via `zod-to-json-schema`.
* **REQ-12 (Match Score)**: The AI evaluation must compute a match score between 0 and 100 representing candidate compatibility with the target job.
* **REQ-13 (Targeted Q&A)**:
  * Generate technical interview questions tailored to the position with interviewer rationale and recommended answers.
  * Generate behavioral interview questions with evaluation criteria and recommended answers.
* **REQ-14 (Skill Gap Identification)**: Identify missing skills and classify each gap into a severity level: `low`, `medium`, or `high`.
* **REQ-15 (Preparation Plan)**: Formulate a sequential day-by-day preparation roadmap with a core daily focus and actionable tasks.
* **REQ-16 (Persistence)**: The generated report must be saved to MongoDB under `InterviewReport` referencing the authenticated user's ID.

#### Module 4: Report Retrieval & History Browsing
* **REQ-17 (History Listing)**: Candidates must be able to view all previously generated reports (`GET /api/interview/`) sorted in descending order of creation date (`createdAt: -1`).
* **REQ-18 (Payload Optimization)**: Historical listing queries must project only lightweight summary fields (`title`, `matchScore`, `createdAt`, `_id`), excluding large text bodies (`resume`, `jobDescription`).
* **REQ-19 (Detail View)**: Candidates must be able to fetch a specific report by ID (`GET /api/interview/report/:interviewId`), restricted to reports owned by that user.

#### Module 5: Automated ATS Resume PDF Compilation
* **REQ-20 (Job-Tailored HTML Synthesis)**: Given an interview report ID, the AI engine must synthesize an ATS-optimized, professional HTML resume emphasizing the candidate's relevant skills for that specific job posting.
* **REQ-21 (Headless PDF Compilation)**: The backend must spawn a headless **Puppeteer** instance, load the generated HTML into an isolated browser page, and compile it to an A4 PDF document with 20mm top/bottom and 15mm left/right margins.
* **REQ-22 (Resource Cleanup)**: The Puppeteer browser instance must be explicitly closed in all execution paths to prevent zombie Chromium processes.
* **REQ-23 (Binary Stream Download)**: The compiled PDF must be returned with `Content-Type: application/pdf` and `Content-Disposition: attachment; filename=resume_<id>.pdf`.

---

## 4. Non-Functional Requirements

### 4.1 Security Requirements
* **SEC-1**: Passwords must never be stored in plaintext; hashing with `bcrypt` (10 rounds) is mandatory.
* **SEC-2**: Authentication tokens must be transported exclusively via HTTP-only cookies to prevent Cross-Site Scripting (XSS) token theft.
* **SEC-3**: Cross-Origin Resource Sharing (CORS) must be restricted to the verified frontend origin (`http://localhost:5173`) with `credentials: true`.
* **SEC-4**: The token blacklist must prevent replay attacks from revoked JWT sessions.

### 4.2 Performance & Scalability
* **PER-1**: Authentication and historical report queries must resolve in less than 200 ms.
* **PER-2**: End-to-end AI evaluation (PDF parse + Gemini API call + DB write) should complete in less than 10 seconds.
* **PER-3**: In-memory file processing ensures zero disk I/O bottleneck during resume uploads.

### 4.3 Reliability & Maintainability
* **REL-1**: All asynchronous operations (Gemini API, Puppeteer, database transactions) must be protected with structured `try/catch` handlers.
* **MNT-1**: Codebase must adhere to modular Layered Architecture (Routers, Middlewares, Controllers, Models, Services).
