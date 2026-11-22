# Interview AI — Class Diagrams & System Architecture

> **Architecture Style**: Layered MVC & Service-Oriented Backend with Component-Hook-Context Frontend Architecture.

---

## 1. Backend Class & Module Architecture

```mermaid
classDiagram
    direction TB

    class AuthRouter {
        +post("/register", registerUserController)
        +post("/login", loginUserController)
        +get("/logout", logoutUserController)
        +get("/get-me", authUser, getMeController)
    }

    class InterviewRouter {
        +post("/", authUser, uploadSingleResume, generateInterViewReportController)
        +get("/", authUser, getAllInterviewReportsController)
        +get("/report/:interviewId", authUser, getInterviewReportByIdController)
        +post("/resume/pdf/:interviewReportId", authUser, generateResumePdfController)
    }

    class AuthMiddleware {
        +authUser(req: Request, res: Response, next: NextFunction)
    }

    class FileMiddleware {
        -storage: MemoryStorage
        -limits: Object
        +single(fieldName: String)
    }

    class AuthController {
        +registerUserController(req: Request, res: Response)
        +loginUserController(req: Request, res: Response)
        +logoutUserController(req: Request, res: Response)
        +getMeController(req: Request, res: Response)
    }

    class InterviewController {
        +generateInterViewReportController(req: Request, res: Response)
        +getInterviewReportByIdController(req: Request, res: Response)
        +getAllInterviewReportsController(req: Request, res: Response)
        +generateResumePdfController(req: Request, res: Response)
    }

    class AIService {
        -ai: GoogleGenAI
        -interviewReportSchema: ZodSchema
        -resumePdfSchema: ZodSchema
        +generateInterviewReport(data: Object) Object
        +generateResumePdf(data: Object) Buffer
        +generatePdfFromHtml(htmlContent: String) Buffer
    }

    class UserModel {
        +username: String
        +email: String
        +password: String
        +create()
        +findOne()
    }

    class TokenBlacklistModel {
        +token: String
        +createdAt: Date
        +create()
        +findOne()
    }

    class InterviewReportModel {
        +user: ObjectId
        +title: String
        +jobDescription: String
        +resume: String
        +selfDescription: String
        +matchScore: Number
        +technicalQuestions: TechnicalQuestion[]
        +behavioralQuestions: BehavioralQuestion[]
        +skillGaps: SkillGap[]
        +preparationPlan: PreparationPlan[]
        +create()
        +find()
        +findById()
        +findOne()
    }

    class TechnicalQuestion {
        +question: String
        +intention: String
        +answer: String
    }

    class BehavioralQuestion {
        +question: String
        +intention: String
        +answer: String
    }

    class SkillGap {
        +skill: String
        +severity: String
    }

    class PreparationPlan {
        +day: Number
        +focus: String
        +tasks: String[]
    }

    class PDFParseWrapper {
        +getText() Object
    }

    class PuppeteerEngine {
        +launch() Browser
        +newPage() Page
        +setContent(html: String)
        +pdf(options: Object) Buffer
        +close()
    }

    %% Relationships
    AuthRouter ..> AuthController : delegates to
    AuthRouter ..> AuthMiddleware : guards routes
    InterviewRouter ..> InterviewController : delegates to
    InterviewRouter ..> AuthMiddleware : guards routes
    InterviewRouter ..> FileMiddleware : parses file buffer

    AuthController ..> UserModel : reads / writes
    AuthController ..> TokenBlacklistModel : writes revoked token
    AuthMiddleware ..> TokenBlacklistModel : queries revoked token

    InterviewController ..> AIService : orchestrates GenAI
    InterviewController ..> InterviewReportModel : reads / writes
    InterviewController ..> PDFParseWrapper : extracts text

    AIService ..> PuppeteerEngine : compiles HTML to PDF

    InterviewReportModel *-- TechnicalQuestion : embeds
    InterviewReportModel *-- BehavioralQuestion : embeds
    InterviewReportModel *-- SkillGap : embeds
    InterviewReportModel *-- PreparationPlan : embeds
```

---

## 2. Frontend React Architecture

```mermaid
classDiagram
    direction TB

    class AuthContext {
        +user: Object
        +loading: Boolean
        +login(credentials: Object)
        +register(userData: Object)
        +logout()
    }

    class InterviewContext {
        +reports: InterviewReport[]
        +currentReport: InterviewReport
        +loading: Boolean
        +createReport(formData: FormData)
        +fetchReports()
        +fetchReportById(id: String)
        +downloadResumePdf(reportId: String)
    }

    class UseAuthHook {
        +useAuth() AuthContext
    }

    class UseInterviewHook {
        +useInterview() InterviewContext
    }

    class AuthApi {
        +register(data: Object)
        +login(data: Object)
        +logout()
        +getMe()
    }

    class InterviewApi {
        +generateInterviewReport(data: Object)
        +getAllInterviewReports()
        +getInterviewReportById(id: String)
        +generateResumePdf(params: Object)
    }

    class ProtectedRoute {
        +children: ReactNode
        +render()
    }

    class LoginPage {
        +handleLogin(event: Event)
    }

    class RegisterPage {
        +handleRegister(event: Event)
    }

    class HomePage {
        +resumeFile: File
        +jobDescription: String
        +selfDescription: String
        +handleSubmit()
    }

    class InterviewPage {
        +interviewId: String
        +handleDownloadResume()
    }

    %% Frontend Relationships
    LoginPage ..> UseAuthHook : consumes
    RegisterPage ..> UseAuthHook : consumes
    ProtectedRoute ..> UseAuthHook : verifies auth

    HomePage ..> UseInterviewHook : triggers AI analysis
    InterviewPage ..> UseInterviewHook : loads report & downloads PDF

    UseAuthHook ..> AuthContext : connects
    UseInterviewHook ..> InterviewContext : connects

    AuthContext ..> AuthApi : executes HTTP requests
    InterviewContext ..> InterviewApi : executes HTTP requests
```

---

## 3. Design Patterns Employed

1. **Layered Architecture (Separation of Concerns)**:
   - Routes handle URL matching and route-level middlewares.
   - Controllers handle HTTP validation, status codes, and request/response coordination.
   - Services contain third-party external integrations (Google GenAI, Puppeteer, PDF-Parse).
   - Models handle database schemas, validations, and Mongoose query executions.

2. **Schema-Enforced Generative AI Pattern**:
   - `GoogleGenAI` prompts are mapped with `Zod` models converted via `zod-to-json-schema`.
   - Forces deterministic, parseable, and strongly-typed JSON outputs matching internal application schemas.

3. **In-Memory File Processing**:
   - Resumes are handled in-memory via `multer.memoryStorage()`, parsed on the fly with `pdf-parse`, and never written to temporary disk space, eliminating disk I/O latency and security risks.

4. **React Context & Custom Hook Pattern**:
   - Encapsulates state, caching, and network requests cleanly inside decoupled context providers (`AuthContext`, `InterviewContext`) consumed via standard hooks (`useAuth`, `useInterview`).
