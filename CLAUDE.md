# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Build
./gradlew build

# Run locally
./gradlew bootRun

# Run tests
./gradlew test

# Run a single test class
./gradlew test --tests "com.b1a4.cafeOn.SomeTest"
```

Swagger UI is available at `http://localhost:8080/api-docs` when running locally.

## Environment Variables

The app uses `spring-dotenv` to load a `.env` file at project root. Required variables:

| Variable | Purpose |
|---|---|
| `DB_URL` | MySQL JDBC connection string |
| `DB_USERNAME` / `DB_PASSWORD` | Database credentials |
| `JWT_SECRET_KEY` | JWT signing key |
| `KAKAO_API_KEY` | Kakao Maps REST API key |
| `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` | S3 credentials |
| `AWS_REGION` / `AWS_S3_BUCKET` | S3 region (`ap-northeast-2`) and bucket (`cafeon-b1a4`) |
| `MAIL_USERNAME` / `MAIL_PASSWORD` | Gmail SMTP (app password) |
| `CDN_BASE_URL` | Base URL for S3-served images |

JPA DDL is set to `validate` — schema must already exist and match entities.

## Architecture

**Stack:** Spring Boot 3.5.6 · Java 17 · MySQL (AWS RDS) · AWS S3 · WebSocket (STOMP) · JWT auth

The codebase is organized by domain, each with its own `controller/`, `service/`, `entity/`, `repository/`, `dto/`, `enums/`, and `exception/` sub-packages under `com.b1a4.cafeOn`:

| Package | Responsibility |
|---|---|
| `user/` | Email/password auth, JWT refresh token rotation, soft delete |
| `cafe/` | Cafe search, Kakao Maps integration, scheduled enrichment |
| `community/post/` | Community posts with images, likes, view counts |
| `community/comment/` | Threaded comments on posts |
| `review/` | Cafe reviews (rating 1–5) that update `CafeEntity.avgRating` |
| `chat/` | Real-time 1:1 and group chat via STOMP WebSocket |
| `image/` | Polymorphic image storage (posts, reviews, profiles) backed by S3 |
| `mypage/wishlist/` | User-bookmarked cafes |
| `qna/` | Customer Q&A with admin answers |
| `report/` | Content reporting and moderation |
| `admin/` | Admin-only views for users, reports, penalties |
| `config/` | Security, JWT, WebSocket, S3, OpenAPI config beans |
| `common/` | `ApiResponse<T>` wrapper, `GlobalExceptionHandler`, base exceptions |

## Key Patterns

**Authentication:** JWT access + refresh token pair. `JwtAuthenticationFilter` validates tokens before each request. Roles are `ROLE_USER` and `ROLE_ADMIN`.

**WebSocket:** STOMP endpoint at `/stomp/chats`. Inbound prefix `/pub/`, broadcast to `/sub/`, per-user delivery via `/queue/` and `/user/`. `WebSocketAuthInterceptor` authenticates connections using JWT.

**Images:** `ImageEntity` is polymorphic — a single table stores images for posts, reviews, and profiles, linked by nullable FK columns and an `ImageCategory` enum. Uploads go through `S3Service`; deletes must call `S3Service.delete()` before removing the entity.

**Soft Delete:** `UserEntity` uses Hibernate `@SQLDelete` / `@Where` — use `userRepository.delete()` and JPA queries will automatically filter out deleted rows.

**Cafe Enrichment:** `CafeScheduler` runs periodic jobs to refresh view-count windows and trigger AI-based review summarization. The `CafeOnApplication` is annotated with `@EnableScheduling`.

**DTOs:** Request/response DTOs live alongside their domain (`dto/` sub-package). Entities are never returned directly from controllers.

**Exception Handling:** Throw domain-specific exceptions (e.g., `PostNotFoundException`, `ChatRoomNotFoundException`); `GlobalExceptionHandler` maps them to HTTP responses using `ApiResponse`.
