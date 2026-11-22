# Interview AI — REST API Documentation

> **Base URL**: `http://localhost:3000`  
> **Protocol**: HTTP/1.1 / JSON / Multipart Form-Data  
> **Authentication**: HTTP-Only Cookie containing JSON Web Token (`token`)

---

## 1. Authentication & Security Architecture

The API uses **JWT (JSON Web Tokens)** stored securely inside `httpOnly` cookies (`token`).
- Tokens expire in **1 day** (`1d`).
- On logout, tokens are persisted to a **MongoDB blacklist collection** (`blacklistTokens`).
- The `authUser` middleware intercepts incoming requests, verifies token signature using `JWT_SECRET`, and rejects any tokens found on the blacklist or expired tokens with `401 Unauthorized`.

```
Client ──[ Cookie: token ]──> Auth Middleware ──[ Check Blacklist & Verify ]──> Route Controller
```

---

## 2. API Endpoints Summary

### Authentication Routes (`/api/auth`)

| Method | Endpoint | Access | Content-Type | Description |
| :--- | :--- | :--- | :--- | :--- |
| `POST` | `/api/auth/register` | Public | `application/json` | Register new user with username, email, and password. |
| `POST` | `/api/auth/login` | Public | `application/json` | Authenticate user and issue JWT token in HTTP-only cookie. |
| `GET` | `/api/auth/logout` | Public | N/A | Invalidate current JWT, append to blacklist, and clear cookie. |
| `GET` | `/api/auth/get-me` | Private | N/A | Fetch profile information for the authenticated user. |

### Interview & AI Generation Routes (`/api/interview`)

| Method | Endpoint | Access | Content-Type | Description |
| :--- | :--- | :--- | :--- | :--- |
| `POST` | `/api/interview/` | Private | `multipart/form-data` | Parse resume PDF, execute Gemini AI evaluation, and save report. |
| `GET` | `/api/interview/` | Private | N/A | Get all saved interview reports belonging to the user. |
| `GET` | `/api/interview/report/:interviewId` | Private | N/A | Retrieve complete report details by interview report ID. |
| `POST` | `/api/interview/resume/pdf/:interviewReportId` | Private | N/A | Generate and download an ATS-friendly tailored resume PDF. |

---

## 3. Detailed Route Specifications

### 3.1 Authentication Endpoints

---

#### 3.1.1 Register User
- **Route**: `POST /api/auth/register`
- **Access**: Public
- **Request Body**:
  ```json
  {
    "username": "shivam_ramoliya",
    "email": "shivam@example.com",
    "password": "SecurePassword123!"
  }
  ```
- **Responses**:
  - `201 Created`: User registered successfully. Sets `token` cookie.
    ```json
    {
      "message": "User registered successfully",
      "user": {
        "id": "674128f092b3a1d987654321",
        "username": "shivam_ramoliya",
        "email": "shivam@example.com"
      }
    }
    ```
  - `400 Bad Request`: Missing required fields or duplicate email/username.
    ```json
    {
      "message": "Account already exists with this email address or username"
    }
    ```

---

#### 3.1.2 Login User
- **Route**: `POST /api/auth/login`
- **Access**: Public
- **Request Body**:
  ```json
  {
    "email": "shivam@example.com",
    "password": "SecurePassword123!"
  }
  ```
- **Responses**:
  - `200 OK`: Authentication successful. Sets `token` cookie.
    ```json
    {
      "message": "Logged in successfully",
      "user": {
        "id": "674128f092b3a1d987654321",
        "username": "shivam_ramoliya",
        "email": "shivam@example.com"
      }
    }
    ```
  - `400 Bad Request`: Missing email or password.
  - `401 Unauthorized`: Invalid credentials.

---

#### 3.1.3 Logout User
- **Route**: `GET /api/auth/logout`
- **Access**: Public / Authenticated
- **Description**: Blacklists the current JWT cookie in MongoDB and clears the cookie from client.
- **Responses**:
  - `200 OK`:
    ```json
    {
      "message": "Logged out successfully"
    }
    ```

---

#### 3.1.4 Get Current User Profile
- **Route**: `GET /api/auth/get-me`
- **Access**: Private (Requires valid `token` cookie)
- **Responses**:
  - `200 OK`:
    ```json
    {
      "user": {
        "id": "674128f092b3a1d987654321",
        "username": "shivam_ramoliya"
      }
    }
    ```
  - `401 Unauthorized`: Token missing, invalid, or blacklisted.

---

### 3.2 Interview & AI Preparation Endpoints

---

#### 3.2.1 Generate Interview Report
- **Route**: `POST /api/interview/`
- **Access**: Private (Authenticated)
- **Content-Type**: `multipart/form-data`
- **File Limits**: Max 3MB (`.pdf` format parsed in-memory via `multer.memoryStorage()`)
- **Request Parameters**:
  - `resume`: File (`binary/pdf`) — Candidate resume document.
  - `jobDescription`: Text (`string`) — Target job opening description & requirements.
  - `selfDescription`: Text (`string`) — Candidate personal intro, background, and goals.
- **Processing Pipeline**:
  1. `pdfParse`: Reads in-memory buffer and extracts plaintext.
  2. `GoogleGenAI`: Evaluates resume + job description using model `gemini-3-flash-preview` and structured JSON schema validated with Zod.
  3. `MongoDB`: Stores generated report linked to `user: req.user.id`.
- **Responses**:
  - `201 Created`:
    ```json
    {
      "message": "Interview report generated successfully.",
      "interviewReport": {
        "_id": "67412ab192b3a1d987654399",
        "title": "Full Stack Software Engineer",
        "matchScore": 85,
        "jobDescription": "Looking for a full stack engineer proficient in React, Node.js...",
        "selfDescription": "Passionate full-stack developer with 2 years of React experience...",
        "resume": "Extracted text content from resume PDF...",
        "technicalQuestions": [
          {
            "question": "How do you handle state management across complex React component trees?",
            "intention": "To test understanding of React Context API vs external state libraries like Redux.",
            "answer": "Explain React 19 Context patterns, useActionState, and performance trade-offs."
          }
        ],
        "behavioralQuestions": [
          {
            "question": "Describe a scenario where you resolved a merge conflict or technical disagreement.",
            "intention": "To evaluate collaboration, communication, and Git conflict resolution skills.",
            "answer": "Use the STAR method: explain the situation, technical disagreement, resolution, and result."
          }
        ],
        "skillGaps": [
          {
            "skill": "Docker & Containerization",
            "severity": "medium"
          }
        ],
        "preparationPlan": [
          {
            "day": 1,
            "focus": "Core React Patterns and Performance",
            "tasks": [
              "Review React 19 features and useMemo/useCallback hooks",
              "Practice building custom state hooks"
            ],
            "_id": "67412ab192b3a1d98765439a"
          }
        ],
        "user": "674128f092b3a1d987654321",
        "createdAt": "2026-11-22T19:40:00.000Z",
        "updatedAt": "2026-11-22T19:40:00.000Z"
      }
    }
    ```
  - `401 Unauthorized`: User is unauthenticated.
  - `500 Server Error`: PDF parsing or Gemini API error.

---

#### 3.2.2 Get All Interview Reports
- **Route**: `GET /api/interview/`
- **Access**: Private (Authenticated)
- **Description**: Returns all reports created by the logged-in user, sorted by newest first (`createdAt: -1`), excluding heavy text fields for fast listing.
- **Responses**:
  - `200 OK`:
    ```json
    {
      "message": "Interview reports fetched successfully.",
      "interviewReports": [
        {
          "_id": "67412ab192b3a1d987654399",
          "title": "Full Stack Software Engineer",
          "matchScore": 85,
          "user": "674128f092b3a1d987654321",
          "createdAt": "2026-11-22T19:40:00.000Z",
          "updatedAt": "2026-11-22T19:40:00.000Z"
        }
      ]
    }
    ```

---

#### 3.2.3 Get Report By ID
- **Route**: `GET /api/interview/report/:interviewId`
- **Access**: Private (Authenticated)
- **Parameters**: `interviewId` (MongoDB ObjectId)
- **Responses**:
  - `200 OK`: Returns full `interviewReport` object including questions, skill gaps, and preparation plan.
  - `404 Not Found`:
    ```json
    {
      "message": "Interview report not found."
    }
    ```

---

#### 3.2.4 Generate Tailored ATS Resume PDF
- **Route**: `POST /api/interview/resume/pdf/:interviewReportId`
- **Access**: Private (Authenticated)
- **Parameters**: `interviewReportId` (MongoDB ObjectId)
- **Processing Pipeline**:
  1. Finds interview report by ID.
  2. Prompts Gemini AI (`gemini-3-flash-preview`) to craft an ATS-compliant, single/double-page HTML resume tailored specifically for the stored job description.
  3. Spawns a headless **Puppeteer** instance.
  4. Sets page HTML and compiles to an **A4 PDF** (`format: "A4"`, `margin: { top: "20mm", bottom: "20mm", left: "15mm", right: "15mm" }`).
  5. Sends the PDF buffer with `Content-Type: application/pdf` and `Content-Disposition: attachment; filename=resume_<id>.pdf`.
- **Responses**:
  - `200 OK`: Binary PDF stream (`Content-Type: application/pdf`).
  - `404 Not Found`: Report does not exist.
  - `500 Server Error`: PDF generation or Puppeteer launch error.

---

## 4. Standard HTTP Error Codes

| Code | Status | Meaning |
| :--- | :--- | :--- |
| `200` | OK | Request processed successfully. |
| `201` | Created | Resource successfully created (User, Report). |
| `400` | Bad Request | Missing required parameters or duplicate key in database. |
| `401` | Unauthorized | Missing, invalid, or blacklisted token cookie. |
| `404` | Not Found | Target resource or report ID does not exist. |
| `500` | Internal Server Error | Unexpected server, Gemini AI, or Puppeteer failure. |
