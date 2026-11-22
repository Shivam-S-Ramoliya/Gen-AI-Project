# Interview AI — System Sequence Diagrams

> Visualizing end-to-end communication flows across Client (React 19), Express Backend, Middlewares, MongoDB, Google Gemini AI, and Puppeteer.

---

## 1. User Registration & Cookie Initialization

```mermaid
sequenceDiagram
    autonumber
    actor User as Candidate / User
    participant Frontend as React Client
    participant AuthRouter as Express Router (/api/auth)
    participant AuthController as Auth Controller
    participant UserModel as MongoDB (users)
    participant JWT as JWT Engine

    User->>Frontend: Enter username, email, password & submit
    Frontend->>AuthRouter: POST /api/auth/register { username, email, password }
    AuthRouter->>AuthController: registerUserController(req, res)
    AuthController->>UserModel: findOne({ $or: [ { username }, { email } ] })
    
    alt User Already Exists
        UserModel-->>AuthController: Existing user document
        AuthController-->>Frontend: 400 Bad Request ("Account already exists...")
        Frontend-->>User: Display error message
    else New User
        UserModel-->>AuthController: null
        AuthController->>AuthController: bcrypt.hash(password, 10)
        AuthController->>UserModel: create({ username, email, password: hash })
        UserModel-->>AuthController: Saved user instance
        AuthController->>JWT: sign({ id: user._id, username }, JWT_SECRET, { expiresIn: '1d' })
        JWT-->>AuthController: Generated token string
        AuthController-->>Frontend: 201 Created + Set-Cookie: token=<jwt>; HttpOnly
        Frontend-->>User: Redirect to Dashboard / Home
    end
```

---

## 2. User Login & Session Verification

```mermaid
sequenceDiagram
    autonumber
    actor User as Candidate / User
    participant Frontend as React Client
    participant AuthRouter as Express Router (/api/auth)
    participant AuthController as Auth Controller
    participant UserModel as MongoDB (users)
    participant JWT as JWT Engine

    User->>Frontend: Enter credentials & submit
    Frontend->>AuthRouter: POST /api/auth/login { email, password }
    AuthRouter->>AuthController: loginUserController(req, res)
    AuthController->>UserModel: findOne({ email })

    alt User Not Found
        UserModel-->>AuthController: null
        AuthController-->>Frontend: 401 Unauthorized ("Invalid credentials")
    else User Exists
        UserModel-->>AuthController: User document with password hash
        AuthController->>AuthController: bcrypt.compare(password, user.password)
        alt Password Mismatch
            AuthController-->>Frontend: 401 Unauthorized ("Invalid credentials")
        else Password Valid
            AuthController->>JWT: sign({ id: user._id, username }, JWT_SECRET, { expiresIn: '1d' })
            JWT-->>AuthController: Generated token
            AuthController-->>Frontend: 200 OK + Set-Cookie: token=<jwt>; HttpOnly
            Frontend-->>User: Authenticated view rendered
        end
    end
```

---

## 3. User Logout & Token Blacklisting

```mermaid
sequenceDiagram
    autonumber
    actor User as Candidate / User
    participant Frontend as React Client
    participant AuthRouter as Express Router (/api/auth)
    participant AuthController as Auth Controller
    participant BlacklistModel as MongoDB (blacklistTokens)

    User->>Frontend: Click "Logout"
    Frontend->>AuthRouter: GET /api/auth/logout (Cookie: token)
    AuthRouter->>AuthController: logoutUserController(req, res)
    AuthController->>BlacklistModel: create({ token: req.cookies.token })
    BlacklistModel-->>AuthController: Blacklist entry saved
    AuthController-->>Frontend: 200 OK + Clear-Cookie: token
    Frontend-->>User: Redirect to Login Page
```

---

## 4. Protected Route Interception (Auth Middleware)

```mermaid
sequenceDiagram
    autonumber
    participant Frontend as React Client
    participant AuthMiddleware as authUser Middleware
    participant BlacklistModel as MongoDB (blacklistTokens)
    participant JWT as JWT Engine
    participant TargetController as Protected Controller

    Frontend->>AuthMiddleware: Protected Request (Cookie: token)
    
    alt Token Missing
        AuthMiddleware-->>Frontend: 401 Unauthorized ("Token not provided.")
    else Token Present
        AuthMiddleware->>BlacklistModel: findOne({ token })
        alt Token Is Blacklisted
            BlacklistModel-->>AuthMiddleware: Blacklist entry found
            AuthMiddleware-->>Frontend: 401 Unauthorized ("token is invalid")
        else Token Is Active
            BlacklistModel-->>AuthMiddleware: null
            AuthMiddleware->>JWT: verify(token, JWT_SECRET)
            alt Signature Invalid or Expired
                JWT-->>AuthMiddleware: Throws error
                AuthMiddleware-->>Frontend: 401 Unauthorized ("Invalid token.")
            else Valid Signature
                JWT-->>AuthMiddleware: Decoded user payload { id, username }
                AuthMiddleware->>TargetController: next() with req.user = decoded
                TargetController-->>Frontend: Processed Response
            end
        end
    end
```

---

## 5. End-to-End Interview Report Generation

```mermaid
sequenceDiagram
    autonumber
    actor User as Candidate / User
    participant Frontend as React Client (Home Page)
    participant FileMiddleware as Multer (Memory Storage)
    participant Controller as Interview Controller
    participant PDFParse as pdfParse Engine
    participant AIService as Gemini AI Service
    participant Gemini as Google GenAI (gemini-3-flash-preview)
    participant DB as MongoDB (InterviewReport)

    User->>Frontend: Select Resume PDF, enter Job & Self Description, click "Generate"
    Frontend->>FileMiddleware: POST /api/interview/ (multipart/form-data)
    FileMiddleware->>FileMiddleware: Buffer resume in-memory (max 3MB)
    FileMiddleware->>Controller: generateInterViewReportController(req, res)
    
    Controller->>PDFParse: new PDFParse(Uint8Array.from(req.file.buffer)).getText()
    PDFParse-->>Controller: Extracted resume text content

    Controller->>AIService: generateInterviewReport({ resume, selfDescription, jobDescription })
    AIService->>Gemini: models.generateContent({ model: "gemini-3-flash-preview", prompt, responseSchema: ZodSchema })
    Gemini-->>AIService: Structured JSON { matchScore, technicalQuestions, behavioralQuestions, skillGaps, preparationPlan, title }
    AIService-->>Controller: Parsed AI evaluation data

    Controller->>DB: create({ user: req.user.id, resume, selfDescription, jobDescription, ...interViewReportByAi })
    DB-->>Controller: Persisted InterviewReport document
    Controller-->>Frontend: 201 Created + { message, interviewReport }
    Frontend-->>User: Navigate to /interview/:id and display full report
```

---

## 6. Report Listing and Detailed View Retrieval

```mermaid
sequenceDiagram
    autonumber
    actor User as Candidate / User
    participant Frontend as React Client
    participant Controller as Interview Controller
    participant DB as MongoDB (InterviewReport)

    %% Flow A: All Reports
    Note over User, DB: Fetching All Historical Reports
    User->>Frontend: Open Dashboard
    Frontend->>Controller: GET /api/interview/
    Controller->>DB: find({ user: req.user.id }).sort({ createdAt: -1 }).select("-resume ...")
    DB-->>Controller: Compact report summaries
    Controller-->>Frontend: 200 OK + { interviewReports: [...] }
    Frontend-->>User: Display report history cards

    %% Flow B: Specific Report
    Note over User, DB: Fetching Single Report Details
    User->>Frontend: Click report card
    Frontend->>Controller: GET /api/interview/report/:interviewId
    Controller->>DB: findOne({ _id: interviewId, user: req.user.id })
    alt Report Not Found
        DB-->>Controller: null
        Controller-->>Frontend: 404 Not Found
    else Report Found
        DB-->>Controller: Complete report document
        Controller-->>Frontend: 200 OK + { interviewReport: {...} }
        Frontend-->>User: Render Q&A, skill gaps, and day-wise roadmap
    end
```

---

## 7. ATS Resume PDF Tailoring & Compilation

```mermaid
sequenceDiagram
    autonumber
    actor User as Candidate / User
    participant Frontend as React Client
    participant Controller as Interview Controller
    participant DB as MongoDB (InterviewReport)
    participant AIService as Gemini AI Service
    participant Gemini as Google GenAI (gemini-3-flash-preview)
    participant Puppeteer as Puppeteer Headless Engine

    User->>Frontend: Click "Download Tailored Resume"
    Frontend->>Controller: POST /api/interview/resume/pdf/:interviewReportId
    Controller->>DB: findById(interviewReportId)
    DB-->>Controller: Stored report with resume & job description

    Controller->>AIService: generateResumePdf({ resume, jobDescription, selfDescription })
    AIService->>Gemini: models.generateContent({ prompt: "tailor resume for job description", responseSchema: { html } })
    Gemini-->>AIService: Structured JSON with HTML resume layout

    AIService->>Puppeteer: launch()
    Puppeteer-->>AIService: Browser instance
    AIService->>Puppeteer: newPage() -> setContent(html, { waitUntil: "networkidle0" })
    AIService->>Puppeteer: page.pdf({ format: "A4", margin: {...} })
    Puppeteer-->>AIService: PDF Binary Buffer
    AIService->>Puppeteer: browser.close()
    AIService-->>Controller: Compiled PDF Buffer

    Controller-->>Frontend: 200 OK (Content-Type: application/pdf, Content-Disposition: attachment)
    Frontend-->>User: Prompt PDF download in browser ("resume_<id>.pdf")
```
