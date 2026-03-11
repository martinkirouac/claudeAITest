# Technical Specification – Note Taking Web App

## 1. Overview

A web application where authenticated users can create, view, edit, delete, and publicly share rich-text notes. Notes are created with Quill.js, stored as JSON in a SQLite database, and rendered in the browser with formatting.

### Core Features

- User authentication (sign up, login, logout) via Spring Security
- Authenticated note management (CRUD)
- Rich text editor using Quill.js with:
  - Bold, Italic
  - Heading levels (H1–H3) + normal text
  - Inline code + code blocks
  - Bullet lists
  - Horizontal rules
- Public sharing of notes via a public URL (toggle on/off)

### Tech

- Java 21 + Spring Boot 3
- Thymeleaf (server-rendered HTML templates)
- TailwindCSS (via CDN) + prose styling for read-only content
- SQLite via `org.xerial:sqlite-jdbc` with Spring JDBC (raw SQL)
- Maven build

## 2. Architecture

### 2.1 High-Level Architecture

- **Frontend & Backend:** Spring Boot (Spring MVC)
  - Thymeleaf templates for server-rendered pages
  - REST API controllers for JSON endpoints consumed by client-side JS
  - Quill.js editor loaded from CDN in Thymeleaf templates
- **Runtime:** JVM (Java 21)
- **Database:** Single SQLite file (e.g., `data/app.db`) accessed via Spring JDBC
- **Auth:** Spring Security with form login, BCrypt password encoding, session-based auth

### 2.2 Application Layers

**Presentation layer**
- Thymeleaf HTML templates
- TailwindCSS for styling (CDN)
- Quill.js editor in browser

**API layer**
- REST JSON endpoints (`@RestController`) for notes CRUD & sharing (consumed by JS)
- MVC controllers (`@Controller`) for page rendering

**Data access layer**
- Raw SQL queries via `JdbcTemplate`
- Repository classes per domain entity

## 3. Functional Requirements

### 3.1 Authentication

Users can:
- Register (email + password, minimum validation)
- Log in / log out

Authentication state is managed by Spring Security via HTTP session.

Unauthenticated users:
- Can access public shared note URLs (read-only)
- Cannot access dashboard or personal notes (redirected to `/login`)

### 3.2 Notes Management (Authenticated)

**Create a new note:**
- Default title: "Untitled note"
- Default empty Quill Delta document

**View a list of own notes:**
- Show title, last updated at, shared status

**View a single note:**
- Load editor with stored Quill Delta JSON document

**Update note:**
- Change title
- Change content (Quill Delta JSON)
- Auto-update `updated_at`

**Delete note:**
- Hard delete

### 3.3 Note Sharing

Users can toggle note "public sharing":

**When enabled:**
- Note gets a unique public slug (e.g. `abcdef1234abcdef`, 16 hex chars)
- Accessible via `/p/{slug}` for anonymous users

**When disabled:**
- Public URL returns 404 / "Note not found"

**Public page rendering:**
- Reads note from DB by `public_slug`
- Shows title and content in read-only mode (Quill viewer)
- No editing or owner information necessary

## 4. Non-Functional Requirements

**Performance**
- Notes list & note view should load under ~300 ms for typical DB sizes

**Security**
- All note operations are scoped to authenticated user's `user_id`
- Public notes are read-only; no leaked private data in API responses
- Passwords hashed with BCrypt

**Reliability**
- Graceful handling of DB errors (Spring exception handling)

**Maintainability**
- Type-safe Java models
- Modularized repository and service classes

**UX**
- Simple, minimal UI with keyboard-friendly editor

## 5. Data Model & Database Schema (SQLite)

### 5.1 Tables

#### users

```sql
CREATE TABLE IF NOT EXISTS users (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  email TEXT NOT NULL UNIQUE,
  password TEXT NOT NULL,
  created_at TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);
```

#### notes

```sql
CREATE TABLE IF NOT EXISTS notes (
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL,
  title TEXT NOT NULL,
  content_json TEXT NOT NULL,
  is_public INTEGER NOT NULL DEFAULT 0,
  public_slug TEXT UNIQUE,
  created_at TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at TEXT NOT NULL DEFAULT (datetime('now')),
  FOREIGN KEY (user_id) REFERENCES users(id)
);
```

### 5.2 Indexes

```sql
CREATE INDEX IF NOT EXISTS idx_notes_user_id ON notes(user_id);
CREATE INDEX IF NOT EXISTS idx_notes_public_slug ON notes(public_slug);
CREATE INDEX IF NOT EXISTS idx_notes_is_public ON notes(is_public);
```

### 5.3 Schema Initialization

Schema SQL lives in `src/main/resources/schema.sql` and is applied on startup via `spring.sql.init.mode=always`.

## 6. Backend: DB & API Layer

### 6.1 Database Configuration

**File:** `config/DatabaseConfig.java`
- Configures a `DataSource` using `org.xerial:sqlite-jdbc` pointing to `data/app.db`
- Exposes a `JdbcTemplate` bean injected into repositories

### 6.2 User Repository

**File:** `repository/UserRepository.java`

Methods:
- `Optional<User> findByEmail(String email)`
- `User save(User user)`

### 6.3 Note Repository

**File:** `repository/NoteRepository.java`

**Java model:**

```java
public class Note {
  private String id;
  private String userId;
  private String title;
  private String contentJson;   // stringified Quill Delta
  private boolean isPublic;
  private String publicSlug;    // nullable
  private String createdAt;
  private String updatedAt;
}
```

**Repository methods:**

- `Note create(String userId, String title, String contentJson)`
- `Optional<Note> findById(String userId, String noteId)`
- `List<Note> findAllByUser(String userId)`
- `Optional<Note> update(String userId, String noteId, String title, String contentJson)`
- `void delete(String userId, String noteId)`
- `Optional<Note> setPublic(String userId, String noteId, boolean isPublic)`
- `Optional<Note> findByPublicSlug(String slug)`

Each method enforces `user_id = ?` in SQL where applicable.

## 7. API Design

### 7.1 Authentication

Spring Security handles form login:
- `GET /login` — login page
- `POST /login` — form submit (`username`, `password`)
- `POST /logout`
- `GET /register` + `POST /register`

All `/api/notes/**` handlers check `SecurityContextHolder` for an authenticated principal. Return 401 if not authenticated.

### 7.2 REST Endpoints (`@RestController`)

Base path: `/api/notes`

#### GET /api/notes
List notes for current user.

**Response 200:**
```json
[
  {
    "id": "note-id",
    "title": "My Note",
    "isPublic": true,
    "updatedAt": "2025-01-01T12:00:00"
  }
]
```

#### POST /api/notes
Create a new note.

**Request body:**
```json
{
  "title": "Optional title",
  "contentJson": "{\"ops\":[{\"insert\":\"\\n\"}]}"
}
```

**Response 201:** created Note

#### GET /api/notes/{id}
Get single note. Returns 404 if not found or not owned by user.

#### PUT /api/notes/{id}
Update note title/content. Returns 200 with updated note or 404.

#### DELETE /api/notes/{id}
Delete note. Returns 204 or 404.

#### POST /api/notes/{id}/share
Toggle public sharing.

**Request body:**
```json
{ "isPublic": true }
```

**Behavior:**
- If `isPublic = true` and no `public_slug` exists, generate one: first 16 chars of `UUID.randomUUID().toString().replace("-", "")`
- If `isPublic = false`, set `is_public = 0` and `public_slug = NULL`

**Response 200:**
```json
{
  "id": "note-id",
  "isPublic": true,
  "publicSlug": "abcdef1234abcdef"
}
```

### 7.3 Public Note

**Page:** `GET /p/{slug}` — handled by `@Controller`, renders Thymeleaf template with read-only Quill viewer.
Returns 404 if slug not found or `is_public = 0`.

## 8. Frontend – Pages & Components (Thymeleaf)

### 8.1 Routes

- `/` — Landing page (`index.html`)
- `/login` — Login form (`auth/login.html`)
- `/register` — Registration form (`auth/register.html`)
- `/dashboard` — Notes list (`dashboard.html`)
- `/notes/{id}` — Note editor page (`notes/editor.html`)
- `/p/{slug}` — Public note page (`public/note.html`)

### 8.2 Layout

- Shared Thymeleaf fragment: `fragments/layout.html`
  - Header with app name, login/logout link
  - TailwindCSS CDN in `<head>`
  - Quill CDN loaded where needed

### 8.3 Templates

**templates/dashboard.html**
- Lists user notes: title, last updated, public badge
- Link to `/notes/{id}` for each note
- "New Note" button (POST form or JS call to `POST /api/notes`, redirect to new note)

**templates/notes/editor.html**
- Title `<input>` (auto-saves on blur via `PUT /api/notes/{id}`)
- Quill.js editor div (auto-saves on text change, debounced 1s)
- Share toggle (calls `POST /api/notes/{id}/share`, shows public URL when enabled)
- Delete button (calls `DELETE /api/notes/{id}`, redirects to dashboard on success)

**templates/public/note.html**
- Read-only Quill viewer (`readOnly: true`, `theme: 'bubble'`)
- Displays note title and content

### 8.4 Client-Side JS

**static/js/editor.js**
- Initializes Quill editor with toolbar
- Debounced auto-save (PUT `/api/notes/{id}`)
- Share toggle handler (POST `/api/notes/{id}/share`)
- Delete button with confirmation dialog

## 9. Quill.js Integration

### 9.1 Toolbar Config

```javascript
const toolbarOptions = [
  ['bold', 'italic'],
  [{ 'header': [1, 2, 3, false] }],
  [{ 'list': 'bullet' }],
  ['code'],
  ['code-block']
];

const quill = new Quill('#editor', {
  theme: 'snow',
  modules: { toolbar: toolbarOptions }
});
```

Load via CDN in Thymeleaf:
```html
<link href="https://cdn.quilljs.com/1.3.7/quill.snow.css" rel="stylesheet">
<script src="https://cdn.quilljs.com/1.3.7/quill.js"></script>
```

### 9.2 Content Storage

Content stored as `JSON.stringify(quill.getContents())` (Quill Delta format).

When loading: `quill.setContents(JSON.parse(contentJson))`.

### 9.3 Read-Only Viewer

```javascript
const quill = new Quill('#viewer', {
  readOnly: true,
  theme: 'bubble'
});
quill.setContents(JSON.parse(contentJson));
```

## 10. Styling (TailwindCSS)

- TailwindCSS loaded via CDN `<script>` tag in Thymeleaf base layout
- Minimal design: neutral background, card-like note container
- Prose-style utility classes for read-only content rendering

## 11. Security Considerations

**Auth enforcement**
- Spring Security `HttpSecurity` secures `/dashboard/**` and `/notes/**`
- Redirects unauthenticated users to `/login`
- CSRF protection enabled (Spring Security default)

**Authorization**
- Every note SQL query filters by `user_id = ?` matching the authenticated principal's ID

**Public notes**
- Slug is 16 random hex chars — sufficiently unguessable
- Public endpoint is read-only, response contains no user data

**XSS**
- Content stored as Quill Delta JSON, not raw HTML
- Quill handles rendering in a sandboxed iframe-like context; no arbitrary `innerHTML` of unsanitized input

**Password security**
- BCryptPasswordEncoder for all password hashing

## 12. Project Structure

```
notetaking/
├── src/
│   ├── main/
│   │   ├── java/com/app/notetaking/
│   │   │   ├── NoteApp.java
│   │   │   ├── config/
│   │   │   │   ├── SecurityConfig.java
│   │   │   │   └── DatabaseConfig.java
│   │   │   ├── controller/
│   │   │   │   ├── PageController.java          (Thymeleaf page routes)
│   │   │   │   └── api/
│   │   │   │       └── NotesApiController.java  (REST JSON)
│   │   │   ├── model/
│   │   │   │   ├── User.java
│   │   │   │   └── Note.java
│   │   │   ├── repository/
│   │   │   │   ├── UserRepository.java
│   │   │   │   └── NoteRepository.java
│   │   │   └── service/
│   │   │       ├── UserService.java
│   │   │       └── NoteService.java
│   │   └── resources/
│   │       ├── templates/
│   │       │   ├── fragments/layout.html
│   │       │   ├── index.html
│   │       │   ├── auth/login.html
│   │       │   ├── auth/register.html
│   │       │   ├── dashboard.html
│   │       │   ├── notes/editor.html
│   │       │   └── public/note.html
│   │       ├── static/
│   │       │   └── js/editor.js
│   │       ├── schema.sql
│   │       └── application.properties
│   └── test/
│       └── java/com/app/notetaking/
└── pom.xml
```

## 13. Key Dependencies (pom.xml)

```xml
<!-- Spring Boot Starters -->
<dependency>spring-boot-starter-web</dependency>
<dependency>spring-boot-starter-thymeleaf</dependency>
<dependency>spring-boot-starter-security</dependency>
<dependency>spring-boot-starter-jdbc</dependency>

<!-- Thymeleaf Spring Security integration -->
<dependency>thymeleaf-extras-springsecurity6</dependency>

<!-- SQLite JDBC Driver -->
<dependency>
  <groupId>org.xerial</groupId>
  <artifactId>sqlite-jdbc</artifactId>
  <version>3.45.3.0</version>
</dependency>
```

## 14. Development Workflow

1. Initialize Spring Boot project via Spring Initializr (Web, Thymeleaf, Security, JDBC)
2. Add SQLite JDBC dependency to `pom.xml`
3. Configure SQLite `DataSource` in `application.properties` and `DatabaseConfig.java`
4. Create `schema.sql` with all table and index definitions
5. Implement `UserRepository` and `NoteRepository` using `JdbcTemplate`
6. Configure Spring Security in `SecurityConfig.java` (form login, BCrypt, route rules)
7. Implement `UserService` with registration logic (hash password, save user)
8. Implement `NoteService` and `NotesApiController` (all REST endpoints)
9. Build `PageController` for Thymeleaf page routes
10. Create Thymeleaf templates with TailwindCSS and Quill.js
11. Implement `static/js/editor.js` for auto-save, share toggle, and delete
12. Test end-to-end: register → create note → edit → share → view public URL
