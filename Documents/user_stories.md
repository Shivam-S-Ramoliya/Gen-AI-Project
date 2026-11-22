# Interview AI — User Stories & Acceptance Criteria

> Agile user stories with explicit acceptance criteria, business rationales, and validation checks.

---

## Epic 1: User Authentication & Account Security

### US-1.1: User Registration
* **User Story**: As a job candidate, I want to create an account with my username, email, and password, so that I can securely generate and preserve my interview reports.
* **Priority**: Must Have (P0)
* **Acceptance Criteria**:
  - The system must require `username`, `email`, and `password`.
  - Both `username` and `email` must be validated for uniqueness in the database.
  - The password must be securely hashed using `bcrypt` (10 rounds) before storing.
  - Upon successful registration, the server must issue a JWT valid for 24 hours (`1d`) inside a secure `httpOnly` cookie (`token`).
  - The system must respond with HTTP `201 Created` and the created user profile (omitting password hash).
  - Attempting to register an existing email or username must return HTTP `400 Bad Request`.

---

### US-1.2: User Login
* **User Story**: As a registered candidate, I want to log in using my email and password, so that I can access my saved interview preparation dashboards.
* **Priority**: Must Have (P0)
* **Acceptance Criteria**:
  - Validating user credentials must compare plain text password against the stored bcrypt hash.
  - Successful login must generate a signed JWT with claims `{ id, username }` and set it in the `httpOnly` cookie.
  - The response must return HTTP `200 OK` with user identification details.
  - Submitting invalid credentials (incorrect password or unregistered email) must return HTTP `401 Unauthorized` without revealing which field was incorrect.

---

### US-1.3: Secure Logout with Token Invalidation
* **User Story**: As a logged-in user, I want to securely log out of my account, so that subsequent users on a shared device cannot access my private data.
* **Priority**: Must Have (P0)
* **Acceptance Criteria**:
  - Calling `GET /api/auth/logout` must retrieve the JWT from the request cookie.
  - The server must persist the token into the `blacklistTokens` collection in MongoDB.
  - The server must clear the `token` cookie on the client response.
  - Any subsequent request utilizing the blacklisted token must be rejected with HTTP `401 Unauthorized` by the `authUser` middleware.

---

### US-1.4: Persistent Session Verification (`/get-me`)
* **User Story**: As a user refreshing the browser, I want my active session to be verified automatically, so that I don't have to log in on every page reload.
* **Priority**: High (P1)
* **Acceptance Criteria**:
  - On app launch, the client calls `GET /api/auth/get-me`.
  - The `authUser` middleware validates token presence, checks blacklist status, and verifies JWT signature.
  - If valid, the server returns HTTP `200 OK` with `{ user: { id, username } }`.
  - If missing, expired, or blacklisted, the server returns HTTP `401 Unauthorized`, prompting the frontend to transition to `/login`.

---

## Epic 2: Resume Ingestion & Parsing

### US-2.1: In-Memory Resume PDF Upload
* **User Story**: As a candidate, I want to upload my PDF resume directly through the web form without saving files to the server's hard drive, so that my personal information is processed safely and quickly.
* **Priority**: Must Have (P0)
* **Acceptance Criteria**:
  - The backend must intercept the file using `multer.memoryStorage()`.
  - File size must be capped at 3 MB; files exceeding this limit must be rejected.
  - The server must extract plaintext from the PDF buffer using `pdf-parse`.
  - Uploaded buffers must exist only in RAM during the lifecycle of the request.

---

## Epic 3: Generative AI Assessment & Preparation Roadmap

### US-3.1: Profile-Job Compatibility Match Scoring
* **User Story**: As a candidate, I want to see an objective match score between my resume and the job description, so that I can understand how qualified I am for the position.
* **Priority**: Must Have (P0)
* **Acceptance Criteria**:
  - The AI evaluation must compute a `matchScore` between 0 and 100 based on the alignment of skills, experience, and job requirements.
  - The match score must be stored as an integer on the report and prominently displayed on the frontend report dashboard.

---

### US-3.2: Role-Specific Technical & Behavioral Q&A Generation
* **User Story**: As a candidate, I want AI to generate probable technical and behavioral interview questions along with the interviewer's intent and best answers, so that I can practice effectively.
* **Priority**: Must Have (P0)
* **Acceptance Criteria**:
  - The Gemini prompt must return an array of `technicalQuestions` containing:
    - `question`: The specific technical query.
    - `intention`: The underlying reasoning behind why an interviewer asks this question.
    - `answer`: Best-practice guidelines, structure, and points to cover in the response.
  - The prompt must return an array of `behavioralQuestions` adhering to the same three-field structure.
  - Structured output must be strictly validated against the Zod schema before saving to MongoDB.

---

### US-3.3: Skill Gap Analysis with Severity Tagging
* **User Story**: As a candidate, I want to know what skills or technologies I am missing for the job, so that I can focus my learning on high-impact areas.
* **Priority**: High (P1)
* **Acceptance Criteria**:
  - The system must identify missing requirements from the job description not present in the candidate's profile.
  - Each skill gap must be classified into a severity level: `low`, `medium`, or `high`.
  - Gaps must be displayed on the UI with visual severity badges to highlight high-priority learning objectives.

---

### US-3.4: Day-by-Day Preparation Roadmap
* **User Story**: As a candidate preparing for an upcoming interview, I want a structured daily study plan, so that I can systematically prepare day by day.
* **Priority**: High (P1)
* **Acceptance Criteria**:
  - The AI must output a sequential array of daily targets (`day: 1`, `day: 2`, etc.).
  - Each day must have a clear `focus` (e.g., "System Design", "Core React Hooks").
  - Each day must include a `tasks` array with concrete actionable steps (articles to read, problems to solve, mock topics).

---

## Epic 4: Report History & Dashboard Management

### US-4.1: Historical Report Exploration
* **User Story**: As a returning candidate, I want to view a list of all my previous interview evaluations, so that I can track my preparation across multiple applications.
* **Priority**: High (P1)
* **Acceptance Criteria**:
  - Calling `GET /api/interview/` must return all reports associated with the logged-in user.
  - Results must be sorted in descending order of creation date (`createdAt: -1`).
  - The query must project only summary fields (`_id`, `title`, `matchScore`, `createdAt`) to keep responses lightweight and fast.

---

### US-4.2: Comprehensive Report Inspection
* **User Story**: As a candidate, I want to view the complete details of a specific historical report, so that I can revisit practice questions and study plans at any time.
* **Priority**: High (P1)
* **Acceptance Criteria**:
  - Calling `GET /api/interview/report/:interviewId` must fetch the full report.
  - The endpoint must strictly enforce user ownership (`user: req.user.id`).
  - Attempting to access another user's report ID or a non-existent ID must return HTTP `404 Not Found`.

---

## Epic 5: Automated ATS Resume Compilation & Download

### US-5.1: Job-Tailored ATS Resume Synthesis
* **User Story**: As a candidate, I want the AI to rewrite and tailor my resume specifically for the target job description, so that my resume passes Applicant Tracking Systems (ATS).
* **Priority**: Must Have (P0)
* **Acceptance Criteria**:
  - Given a stored report, Google Gemini generates clean, professional, semantic HTML.
  - The generated resume must highlight qualifications that directly map to the job requirements while maintaining candidate authenticity.
  - The layout must follow standard 1–2 page ATS formatting without complex multi-column graphics that confuse parsers.

---

### US-5.2: Headless PDF Compilation & One-Click Download
* **User Story**: As a candidate, I want to download the tailored resume as a print-ready A4 PDF file with one click, so that I can immediately apply to the job.
* **Priority**: Must Have (P0)
* **Acceptance Criteria**:
  - Calling `POST /api/interview/resume/pdf/:interviewReportId` triggers a headless Puppeteer browser instance.
  - Puppeteer compiles the HTML to an A4 PDF (`format: "A4"`, `margin: { top: "20mm", bottom: "20mm", left: "15mm", right: "15mm" }`).
  - The browser instance must close cleanly after buffer generation to prevent resource leaks.
  - The response must set `Content-Type: application/pdf` and `Content-Disposition: attachment; filename=resume_<id>.pdf`.
  - The frontend initiates an automatic file download in the user's browser.
