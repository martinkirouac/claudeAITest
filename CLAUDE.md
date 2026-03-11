# CLAUDE.md

We're building the app described in @SPEC.MD. Read that file for general architectural tasks or to double-check the exact database structure, tech stack or application architecture.

Keep your replies extremely concise and focus on conveying the key information. No unnecessary fluff, no long code snippets.

Whenever working with any third-party library or something similar, you MUST look up the official documentation to ensure that you're working with up-to-date information.
Use the DocsExplorer subagent for efficient documentation lookup.

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
mvn spring-boot:run          # Start development server (http://localhost:8080)
mvn package                  # Build production JAR
mvn package -DskipTests      # Build without running tests
java -jar target/*.jar       # Run production JAR
mvn test                     # Run tests
```

## Architecture

This is a Spring Boot 3 / Java 21 application with:

- **Build Tool**: Maven
- **Language**: Java 21
- **Templating**: Thymeleaf (server-rendered HTML)
- **Styling**: TailwindCSS via CDN
- **Rich Text**: Quill.js via CDN

### Key Dependencies

- `spring-boot-starter-web` - Spring MVC
- `spring-boot-starter-thymeleaf` - Thymeleaf templating
- `spring-boot-starter-security` - Spring Security (form login, BCrypt)
- `spring-boot-starter-jdbc` - JdbcTemplate for raw SQL
- `sqlite-jdbc` - SQLite driver
- `thymeleaf-extras-springsecurity6` - Thymeleaf Security integration

### Project Structure

- `src/main/java/com/app/notetaking/` - Java source (config, controller, model, repository, service)
- `src/main/resources/templates/` - Thymeleaf HTML templates
- `src/main/resources/static/` - Static assets (JS, CSS)
- `src/main/resources/schema.sql` - SQLite schema (auto-applied on startup)
- `data/app.db` - SQLite database file (gitignored)
