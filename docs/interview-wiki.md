# cafeOn 백엔드 면접 준비 위키

## 목차
1. [프로젝트 한 줄 소개](#1-프로젝트-한-줄-소개)
2. [기술 스택](#2-기술-스택)
3. [담당 도메인 전체 정리](#3-담당-도메인-전체-정리)
4. [JWT 인증 시스템 (심화)](#4-jwt-인증-시스템-심화)
5. [카페 검색 & 카카오 API 병합 (심화)](#5-카페-검색--카카오-api-병합-심화)
6. [리뷰 시스템](#6-리뷰-시스템)
7. [위시리스트 시스템](#7-위시리스트-시스템)
8. [공통 아키텍처 패턴](#8-공통-아키텍처-패턴)
9. [면접 예상 질문 TOP 50](#9-면접-예상-질문-top-50-코드-기반-답변)

---

## 1. 프로젝트 한 줄 소개

> 카페 정보 검색 + 커뮤니티 + 실시간 채팅을 결합한 카페 전용 소셜 플랫폼의 **백엔드 REST API 서버**

- 팀 구성: 백엔드 3명 (나: User/Cafe/Review/Wishlist 담당)
- Spring Boot 3.5.6 + Java 17 + MySQL (AWS RDS) + AWS S3
- JWT 기반 무상태(Stateless) 인증, WebSocket 실시간 채팅

---

## 2. 기술 스택

| 분류 | 기술 |
|------|------|
| 언어/프레임워크 | Java 17, Spring Boot 3.5.6 |
| 인증/보안 | Spring Security, JWT (jjwt 0.9.1, HS512 알고리즘) |
| ORM | Spring Data JPA + Hibernate |
| DB | MySQL (AWS RDS) |
| 파일 저장 | AWS S3 (SDK v2) |
| 실시간 통신 | WebSocket + STOMP |
| 외부 API | 카카오맵 REST API |
| 이메일 | Spring Mail (Gmail SMTP) |
| 문서화 | SpringDoc OpenAPI 3.0 (Swagger) |
| 빌드 | Gradle |
| 배포 | AWS EC2 + Docker + Nginx |

---

## 3. 담당 도메인 전체 정리

### 3-1. User (인증)

| API | Method | 인증 필요 |
|-----|--------|----------|
| `/api/auth/signup` | POST | X |
| `/api/auth/login` | POST | X |
| `/api/auth/refresh` | POST | X (Refresh Token) |
| `/api/auth/logout` | POST | O (Access Token) |
| `/api/auth/password` | PUT | O |
| `/api/auth/password/reset` | POST | X |
| `/api/users/me` | GET/PUT | O |

### 3-2. Cafe (카페)

| API | 기능 |
|-----|------|
| `GET /api/cafes/search` | 키워드/태그 검색 (카카오 API + DB 병합) |
| `GET /api/cafes/{id}` | 상세 조회 (조회수 +1) |
| `GET /api/cafes/nearby` | 위치 기반 근처 카페 (MySQL ST_Distance_Sphere) |
| `GET /api/cafes/random10` | 랜덤 10개 |
| `GET /api/cafes/hot10` | 인기 10개 (가중치 공식) |
| `GET /api/cafes/wish10` | 찜 많은 10개 |
| `GET /api/cafes/{id}/reviews` | 특정 카페 리뷰 목록 |

### 3-3. Review (리뷰)

| API | Method |
|-----|--------|
| `/api/cafes/{cafeId}/reviews` | POST (작성) |
| `/api/reviews/{reviewId}` | PUT (수정) |
| `/api/reviews/{reviewId}` | DELETE |
| `/api/my/reviews` | GET (내 리뷰 목록) |

### 3-4. Wishlist (위시리스트)

| API | 기능 |
|-----|------|
| `POST /api/my/wishlist/{cafeId}` | 찜 토글 (추가/제거) |
| `DELETE /api/my/wishlist/{cafeId}` | 특정 카테고리 제거 |
| `GET /api/my/wishlist` | 카테고리별 목록 조회 |
| `GET /api/my/wishlist/{cafeId}` | 해당 카페의 담긴 카테고리 확인 |

---

## 4. JWT 인증 시스템 (심화)

### 4-1. 토큰 구조

```
Access Token
  - 알고리즘: HS512 (HMAC-SHA512)
  - 만료: 30분
  - Payload: sub(userId), iss(cafeOn), iat, exp, tokenType(access), role

Refresh Token
  - 알고리즘: HS512
  - 만료: 14일
  - Payload: sub(userId), iss(cafeOn), iat, exp, tokenType(refresh), role
  - DB 저장: users.refresh_token 컬럼
```

### 4-2. 로그인 흐름

```
클라이언트 → POST /api/auth/login {email, password}
  → AuthService.getByCredentials()
      → userRepository.findByEmail(email)
      → passwordEncoder.matches(입력 PW, DB 암호화 PW)
  → TokenProvider.issueTokens(user)
      → Access Token (30분) + Refresh Token (14일) 발급
  → user.setRefreshToken() + userRepository.save()  // DB에 Refresh 저장
  → 응답: {token, refreshToken}
```

### 4-3. 요청 인증 흐름 (JwtAuthenticationFilter)

```
모든 요청 → JwtAuthenticationFilter.doFilterInternal()
  1. Authorization 헤더에서 "Bearer <token>" 추출
  2. /api/auth/refresh 경로는 제외
  3. TokenProvider.validateAndExtractClaims(token, "access")
     → Jwts.parser().setSigningKey(SECRET).parseClaimsJws(token)
     → 만료/위조 시 null 반환
  4. 유효하면 UsernamePasswordAuthenticationToken 생성
     → principal = userId, authorities = ["ROLE_USER"] or ["ROLE_ADMIN"]
  5. SecurityContextHolder에 저장
  → Controller에서 @AuthenticationPrincipal String userId 로 주입받음
```

### 4-4. Token Rotation (토큰 갱신)

```
클라이언트 → POST /api/auth/refresh {refreshToken}
  → AuthService.refreshTokens()
      1. "Bearer " prefix 제거
      2. validateAndExtractClaims(token, "refresh") 로 검증
      3. userId 추출 → DB에서 유저 조회
      4. DB의 refresh_token과 요청값 비교 (재사용 방지)
      5. 새 Access Token + 새 Refresh Token 발급
      6. 새 Refresh Token DB 저장 (이전 것 무효화)
  → 응답: {accessToken, refreshToken}
```

**Token Rotation 전략의 장점:** 기존 Refresh Token을 재사용하면 mismatch → 탈취된 토큰 무효화 가능

### 4-5. 로그아웃

```
클라이언트 → POST /api/auth/logout (헤더: Authorization: Bearer <token>)
  → AuthService.logout()
      1. Access Token 유효성 검증
      2. userId 추출 → DB 유저 조회
      3. user.setRefreshToken(null) + save()
  → 이후 /refresh 호출 시 "Refresh Token mismatch" 로 거부
```

### 4-6. 역할 계층

```
SecurityConfig에서 설정:
ROLE_ADMIN > ROLE_USER
→ ADMIN은 USER의 모든 권한 포함
```

### 4-7. 비밀번호 관련

- 저장: `BCryptPasswordEncoder.encode()` (단방향 해시)
- 검증: `passwordEncoder.matches(rawPw, encodedPw)`
- 임시 비밀번호: 영대소문자+숫자 10자리 랜덤 생성 → BCrypt 암호화 후 DB 저장 → Gmail SMTP 발송

---

## 5. 카페 검색 & 카카오 API 병합 (심화)

### 5-1. 검색 로직 흐름

```
GET /api/cafes/search?query=강남카페&tag=분위기

keyword 있음:
  fetchFromKakao(keyword)
    → 카카오 REST API 첫 페이지 15개만 (성능 최적화)
    → UriComponentsBuilder로 UTF-8 인코딩 보장
  
  searchAndMerge(keyword)
    → 카카오 결과 이름 목록 추출
    → cafeRepository.findByNameIn(names) — DB 배치 조회
    → 병합: DB 존재 → DB 데이터 사용, 없음 → 카카오 데이터 사용
    → 신규 카페: @Async synchronizeCafe() 비동기 DB 저장
  
  N+1 해결:
    → cafeRepository.findAllById(cafeIds) — 카페 배치 조회
    → cafeRepository.findTagNamesByCafeIds(cafeIds) — 태그 배치 조회
    → 결과를 Map<Long, List<String>>으로 관리

keyword 없음:
  → tag 있으면: cafeRepository.findByTag(tag) — JOIN 쿼리
  → 없으면: cafeRepository.findAll()
```

### 5-2. 카카오 API 호출 방식

```java
String url = UriComponentsBuilder
    .fromUriString("https://dapi.kakao.com/v2/local/search/keyword.json")
    .queryParam("query", keyword)
    .queryParam("size", 15)      // 첫 페이지만
    .queryParam("page", 1)
    .encode(StandardCharsets.UTF_8)  // 한글 인코딩
    .toUriString();

headers.add("Authorization", "KakaoAK " + kakaoApiKey);
restTemplate.exchange(url, HttpMethod.GET, entity, new ParameterizedTypeReference<>(){})
```

### 5-3. 인기 카페 Hot Score 공식

```sql
HotScore = (views_last7d * w7d) + (viewCount * wAll) + (avgRating * 50 * wRate) + (reviewCount * 5 * wRev)
```
→ 가중치를 파라미터로 받아 조정 가능 (`?w7d=3&wAll=1&wRate=2&wRev=2`)

### 5-4. 근처 카페 (Haversine 공식)

```sql
ST_Distance_Sphere(POINT(longitude, latitude), POINT(:lon, :lat)) <= :radius
ORDER BY distance ASC
LIMIT 100
```
→ DB 결과 10개 미만이면 카카오 Category API (CE7=카페) 로 보완

### 5-5. 조회수 관리

- `viewCount`: 상세 조회 시마다 +1
- `viewsLast7d`: CafeScheduler가 매일 새벽 3시 → `viewsLast7d = viewCount - lastViewCount`
- 인기 지수 계산 시 `viewsLast7d` 가중치 가장 높게 설정

---

## 6. 리뷰 시스템

### 6-1. Entity 구조

```
ReviewEntity
  - reviewId (BIGINT, AUTO)
  - rating (INT, 1~5)
  - content (TEXT)
  - createdAt (TIMESTAMP, @PrePersist)
  - user → UserEntity (ManyToOne, LAZY)
  - cafe → CafeEntity (ManyToOne, LAZY)
  - images → List<ImageEntity> (OneToMany, cascade=ALL, orphanRemoval=true)
```

### 6-2. 리뷰 작성 (multipart/form-data)

```
POST /api/cafes/{cafeId}/reviews
Content-Type: multipart/form-data
  - review part: {"rating": 5, "content": "좋아요"} (JSON)
  - images part: [파일들] (선택)

Controller → ObjectMapper.readValue(reviewJson, ReviewRequestDTO.class)
Service    → ReviewEntity 생성 + S3 이미지 업로드 + ImageEntity 연결
```

### 6-3. 권한 검증 (수정/삭제)

```java
// 작성자 또는 관리자만 가능
boolean isOwner = review.getUser().getUserId().equals(userId);
boolean isAdmin = SecurityContextHolder.getContext()
    .getAuthentication().getAuthorities()
    .stream().anyMatch(a -> a.getAuthority().equals("ROLE_ADMIN"));
if (!isOwner && !isAdmin) throw new ForbiddenException();
```

---

## 7. 위시리스트 시스템

### 7-1. 카테고리 Enum

```java
HIDEOUT   // 나만의 아지트
WORK      // 작업하기 좋은
ATMOSPHERE // 분위기 좋은
TASTE     // 커피·디저트 맛집
PLANNED   // 방문 예정
```

### 7-2. 중복 방지 설계

```sql
UNIQUE KEY (user_id, cafe_id, category)
-- 같은 카페를 다른 카테고리로는 여러 개 저장 가능
-- 같은 카페 + 같은 카테고리 중복 불가
```

### 7-3. 찜 토글 로직

```
POST /api/my/wishlist/{cafeId}?category=PLANNED

1. em.getReference(CafeEntity.class, cafeId) — 프록시만 생성 (DB 조회 안 함, 성능 최적화)
2. 이미 찜 있으면 → 삭제 (wished: false)
3. 없으면 → 생성 (wished: true)
4. DataIntegrityViolationException → 외래키 위반 처리
```

**`em.getReference()` 쓴 이유:** CafeEntity 실제 데이터는 필요 없고 FK 참조만 필요. DB 조회를 1번 아낄 수 있음.

---

## 8. 공통 아키텍처 패턴

### 8-1. 요청 처리 흐름

```
HTTP 요청
  → JwtAuthenticationFilter (토큰 검증)
  → Controller (@RestController)
      → @AuthenticationPrincipal String userId
      → DTO 받기
  → Service (@Transactional)
      → 비즈니스 로직
      → Repository 호출
  → Repository (JpaRepository)
      → MySQL 쿼리
  → ApiResponse<T> 래퍼로 응답
```

### 8-2. 표준 응답 형식

```json
{
  "message": "로그인 성공",
  "data": { "token": "...", "refreshToken": "..." }
}
```

### 8-3. N+1 문제 해결 패턴

```
문제: 카페 목록 100개 조회 시 각 카페마다 태그 조회 쿼리 발생 = 101번 쿼리

해결:
1. cafeIds 리스트 추출
2. findTagNamesByCafeIds(cafeIds) — IN 쿼리로 한 번에 조회
3. Map<Long, List<String>> 으로 변환
4. 카페별로 map.getOrDefault() 로 태그 매핑
```

### 8-4. 트랜잭션 전략

```java
@Transactional(readOnly = true)  // 조회 전용 (성능 최적화)
@Transactional                   // 쓰기 (기본값)
@Async @Transactional            // 비동기 쓰기 (카카오 신규 카페 저장)
```

### 8-5. Soft Delete (UserEntity)

```java
@SQLDelete(sql = "UPDATE users SET deleted_at = NOW() WHERE user_id = ?")
@Where(clause = "deleted_at IS NULL")

// softDelete() 메서드:
user.setDeletedAt(LocalDateTime.now());
user.setStatus(UserStatus.DELETED);
user.setRefreshToken(null);
user.setNickname("탈퇴회원");
```

---

## 9. 면접 예상 질문 TOP 50 (코드 기반 답변)

> 모든 답변은 실제 구현된 코드(`TokenProvider`, `AuthService`, `CafeService`, `SecurityConfig` 등)를 근거로 작성했습니다.

---

### 🔐 카테고리 1: JWT / 인증 (Q1~Q12)

---

### Q1. JWT란 무엇인가요?

> JWT(JSON Web Token)는 Header.Payload.Signature 세 부분으로 구성된 자가 포함(Self-contained) 토큰입니다. Payload에 사용자 정보(userId, role 등)를 담고 서버 비밀키로 서명해, 서버가 DB를 조회하지 않아도 토큰만으로 사용자를 식별할 수 있습니다. 저는 `TokenProvider` 클래스에서 `Jwts.builder()`로 생성하고 `parseClaimsJws()`로 검증했습니다.

---

### Q2. JWT를 사용한 이유가 뭔가요?

> 세션 방식은 서버 메모리에 세션을 저장해 서버가 여러 대로 늘어날 때 세션 동기화 문제가 생깁니다. JWT는 토큰 자체에 정보를 담아 서버가 **Stateless**로 동작할 수 있어 수평 확장(Scale-out)에 유리합니다. 이 프로젝트는 AWS EC2 배포를 목표로 했고 향후 확장성을 고려해 JWT를 선택했습니다.

---

### Q3. Access Token과 Refresh Token의 역할 차이는 뭔가요?

> `TokenProvider.issueAccessToken()`은 만료 30분짜리 토큰을 발급해 매 API 요청 헤더에 담아 보냅니다. `issueRefreshToken()`은 만료 14일짜리로 Access Token이 만료됐을 때 재발급받는 용도입니다. Access Token은 짧게 유지해 탈취 위험을 줄이고, Refresh Token은 길게 유지해 사용자 재로그인 빈도를 낮춥니다.

---

### Q4. Refresh Token을 왜 DB에 저장했나요?

> JWT는 서버가 상태를 저장하지 않아 발급한 토큰을 취소할 방법이 없습니다. Refresh Token을 `users.refresh_token` 컬럼에 저장해두면 로그아웃 시 null로 만들어 이후 갱신 요청을 차단할 수 있습니다. `AuthService.logout()`에서 Access Token으로 userId를 추출한 뒤 `user.setRefreshToken(null)`을 호출해 구현했습니다.

---

### Q5. Token Rotation이란 무엇이고 왜 적용했나요?

> Refresh Token을 한 번 쓰면 새 토큰으로 교체하는 전략입니다. `AuthService.refreshTokens()`에서 요청값과 DB 저장값을 `cleanedToken.equals(stored.trim())`으로 비교하고, 일치하면 새 Access + Refresh 쌍을 발급한 후 DB를 업데이트합니다. 공격자가 탈취한 이전 Refresh Token으로 재시도하면 DB 값과 달라 `"Refresh Token mismatch"`로 거부됩니다.

---

### Q6. JWT의 단점은 무엇이고 이 프로젝트에서 어떻게 보완했나요?

> JWT는 발급 후 만료 전까지 서버에서 강제 무효화가 어렵다는 단점이 있습니다. 이를 보완하기 위해 두 가지를 적용했습니다. 첫째, Refresh Token을 DB에 저장해 로그아웃 시 무효화했습니다. 둘째, Access Token은 30분으로 짧게 유지해 탈취되더라도 피해 시간을 최소화했습니다. 완벽한 해결책은 아니지만 실용적인 트레이드오프로 판단했습니다.

---

### Q7. 로그아웃을 어떻게 구현했나요?

> `AuthController.logout()`에서 `Authorization` 헤더의 Access Token을 받아 `TokenProvider.validateAndExtractClaims(token, "access")`로 검증 후 userId를 추출합니다. 그런 다음 `userRepository.findById(userId)`로 유저를 조회해 `user.setRefreshToken(null)`을 호출하고 save합니다. 이후 클라이언트가 `/api/auth/refresh`를 요청하면 DB에 null이 저장되어 있어 `"Refresh Token mismatch"`로 거부됩니다.

---

### Q8. JwtAuthenticationFilter는 어떻게 동작하나요?

> `OncePerRequestFilter`를 상속해 모든 요청에 한 번씩 실행됩니다. `Authorization` 헤더에서 `"Bearer "` 뒤 토큰을 추출해 `TokenProvider.validateAndExtractClaims(token, "access")`를 호출합니다. 유효하면 `UsernamePasswordAuthenticationToken(userId, null, authorities)`를 생성해 `SecurityContextHolder`에 저장합니다. `/api/auth/refresh` 경로는 Refresh Token으로 오므로 Access Token 검증에서 제외했습니다.

---

### Q9. @AuthenticationPrincipal은 어떻게 동작하나요?

> `JwtAuthenticationFilter`에서 `SecurityContextHolder`에 저장한 Authentication 객체의 `principal`을 꺼냅니다. `UsernamePasswordAuthenticationToken` 생성 시 `principal = userId`(String)로 설정했기 때문에, `@AuthenticationPrincipal String userId`로 Controller 메서드에서 바로 주입받을 수 있습니다. 비밀번호 변경 API에서는 `Authentication authentication`을 직접 파라미터로 받아 `authentication.getName()`으로 userId를 꺼냈습니다.

---

### Q10. Access Token이 만료됐을 때 클라이언트는 어떻게 처리하나요?

> 서버는 `TokenProvider.validateAndExtractClaims()`에서 `ExpiredJwtException`이 발생하면 null을 반환하고, 이후 인증이 필요한 API는 401을 응답합니다. 클라이언트는 401을 받으면 저장해둔 Refresh Token을 담아 `POST /api/auth/refresh`를 호출합니다. 성공하면 새 Access + Refresh Token을 저장하고 원래 요청을 재시도하도록 프론트와 합의해 설계했습니다.

---

### Q11. CSRF를 비활성화한 이유는?

> CSRF 공격은 브라우저가 세션 쿠키를 자동으로 첨부할 때 발생합니다. 이 프로젝트는 JWT를 `Authorization` 헤더에 담아 전송하는 방식이므로 브라우저가 자동으로 첨부하지 않아 CSRF 위협이 없습니다. `SecurityConfig`에서 `.csrf(CsrfConfigurer::disable)`로 비활성화해 불필요한 처리를 제거했습니다.

---

### Q12. HS512 알고리즘을 선택한 이유는?

> `SignatureAlgorithm.HS512`는 HMAC-SHA512 방식으로, 서버 하나의 비밀키로 서명과 검증을 모두 수행합니다. RS256 같은 비대칭 알고리즘은 공개키/개인키 쌍이 필요해 단일 서버 구조에서는 불필요한 복잡성이 생깁니다. 단일 서버 구조에서는 HS512가 구현이 간단하면서도 충분히 안전합니다.

---

### 🛡 카테고리 2: Spring Security (Q13~Q17)

---

### Q13. Spring Security FilterChain이란?

> HTTP 요청이 Controller에 도달하기 전에 순서대로 거쳐가는 필터들의 체인입니다. `SecurityConfig.filterChain()`에서 `.authorizeHttpRequests()`로 경로별 접근 권한을 설정하고, `http.addFilterAfter(jwtAuthenticationFilter, CorsFilter.class)`로 CORS 처리 이후 JWT 검증이 실행되도록 순서를 지정했습니다. `SessionCreationPolicy.STATELESS`로 세션을 생성하지 않도록 설정했습니다.

---

### Q14. SecurityContextHolder는 무엇인가요?

> 현재 요청 스레드의 인증 정보를 보관하는 컨텍스트입니다. `JwtAuthenticationFilter`에서 토큰 검증 후 `SecurityContextHolder.getContext().setAuthentication(authentication)`으로 저장합니다. 이후 같은 요청 내 어느 레이어에서든 `SecurityContextHolder.getContext().getAuthentication()`으로 꺼낼 수 있습니다. 리뷰 수정/삭제 시 `ROLE_ADMIN` 확인도 이 방식으로 했습니다.

---

### Q15. ROLE_ADMIN > ROLE_USER 역할 계층을 어떻게 구현했나요?

> `SecurityConfig`에서 `RoleHierarchyImpl.fromHierarchy("ROLE_ADMIN > ROLE_USER")` 빈을 등록했습니다. 이렇게 하면 `ROLE_USER` 권한이 필요한 모든 API에 `ROLE_ADMIN`도 접근할 수 있습니다. 리뷰 수정/삭제에서 `ROLE_ADMIN`이 모든 리뷰를 관리할 수 있도록 이 설정을 활용했습니다.

---

### Q16. CORS를 어떻게 설정했나요?

> `SecurityConfig.corsConfigurationSource()`에서 `CorsConfiguration`을 생성해 허용 Origin을 `"*"`, 허용 메서드를 `GET/POST/PUT/DELETE/PATCH/OPTIONS/HEAD`로 설정했습니다. `setAllowCredentials(true)`로 쿠키·인증 헤더를 허용했습니다. 현재는 개발 편의상 와일드카드를 썼지만, 실서비스에서는 프론트엔드 도메인으로 제한해야 합니다.

---

### Q17. BCrypt를 선택한 이유는?

> BCrypt는 단방향 해시에 salt를 자동 부여해 같은 비밀번호라도 매번 다른 해시값을 생성합니다. Rainbow Table 공격을 방어하고 work factor(기본 10)로 연산 비용을 조절할 수 있습니다. Spring Security가 `BCryptPasswordEncoder`를 기본으로 제공해 `passwordEncoder.matches(rawPw, encodedPw)`로 간편하게 검증했습니다.

---

### 🗄 카테고리 3: JPA / Hibernate (Q18~Q27)

---

### Q18. N+1 문제란 무엇이고 어떻게 해결했나요?

> 1번의 쿼리로 N개 결과를 가져온 뒤, 각 결과마다 연관 데이터를 추가 조회해 N+1번 쿼리가 발생하는 문제입니다. `CafeService.searchCafes()`에서 카페 목록을 조회한 뒤 `cafeRepository.findTagNamesByCafeIds(cafeIds)`로 IN 절 배치 조회하고, 결과를 `Map<Long, List<String>>`으로 변환해 카페별로 매핑했습니다. 리뷰도 `getReviewsByCafeIds(cafeIds)`로 동일하게 처리했습니다.

---

### Q19. @Transactional의 역할은?

> 메서드 실행을 하나의 트랜잭션으로 묶어 중간에 예외가 발생하면 전체를 롤백합니다. `AuthService.refreshTokens()`에서 새 Refresh Token을 DB에 저장하다가 오류가 나면 이전 상태로 되돌아갑니다. `@EnableAsync`와 함께 쓴 `synchronizeCafe()`에는 `@Async @Transactional`을 같이 달아 비동기 저장도 트랜잭션을 보장했습니다.

---

### Q20. @Transactional(readOnly=true)를 사용한 이유는?

> `CafeService` 클래스 레벨에 `@Transactional(readOnly = true)`를 선언했습니다. 읽기 전용 트랜잭션은 영속성 컨텍스트의 변경감지(Dirty Checking)를 생략해 약간의 성능 향상이 있고, 실수로 데이터를 수정하는 것을 방지합니다. 데이터 변경이 필요한 `getCafeDetail()`(조회수 증가), `synchronizeCafe()`(저장)에는 별도로 `@Transactional`을 붙여 오버라이드했습니다.

---

### Q21. Lazy Loading을 사용한 이유는?

> `ReviewEntity`의 `user`, `cafe` 필드에 `FetchType.LAZY`를 설정했습니다. Eager Loading이면 리뷰 조회 시 항상 User와 Cafe 전체를 JOIN해서 가져오는데, 대부분의 조회에서 사용자 전체 정보나 카페 전체 정보는 필요 없습니다. Lazy Loading으로 실제 접근 시점에만 추가 쿼리를 발생시켜 불필요한 데이터 로딩을 막았습니다.

---

### Q22. em.getReference()를 사용한 이유는?

> `WishlistService.toggle()`에서 `em.getReference(CafeEntity.class, cafeId)`를 호출했습니다. 위시리스트 생성 시 `CafeEntity`의 실제 데이터는 필요 없고 FK 참조값만 있으면 됩니다. `getReference()`는 프록시 객체만 생성하고 실제 DB 조회를 하지 않아 SELECT 쿼리 1번을 절약합니다. 단 존재하지 않는 cafeId면 `DataIntegrityViolationException`이 발생하므로 catch로 처리했습니다.

---

### Q23. Soft Delete를 어떻게 구현했나요?

> `UserEntity`에 `@SQLDelete(sql = "UPDATE users SET deleted_at = NOW() WHERE user_id = ?")`를 달아 JPA가 delete 호출 시 UPDATE 쿼리를 실행하게 했습니다. `@Where(clause = "deleted_at IS NULL")`로 일반 SELECT에서 탈퇴 유저가 자동 제외됩니다. `softDelete()` 메서드에서 `deletedAt`, `status = DELETED`, `refreshToken = null`, `nickname = "탈퇴회원"`을 일괄 처리했습니다.

---

### Q24. cascade=ALL과 orphanRemoval=true의 차이는?

> `ReviewEntity.images`에 `cascade = CascadeType.ALL, orphanRemoval = true`를 설정했습니다. `cascade=ALL`은 Review 저장/삭제 시 연관 ImageEntity도 같이 저장/삭제합니다. `orphanRemoval=true`는 images 리스트에서 특정 ImageEntity를 제거하면 자동으로 DELETE 쿼리가 나갑니다. 단 ImageEntity를 DB에서 삭제해도 S3의 실제 파일은 따로 `S3Service.delete()`를 호출해야 합니다.

---

### Q25. @PrePersist는 무엇인가요?

> JPA가 Entity를 처음 저장하기 직전에 실행되는 콜백 메서드에 붙이는 어노테이션입니다. `ReviewEntity`에서 `@PrePersist`로 `createdAt = LocalDateTime.now()`를 자동 설정했습니다. 개발자가 createdAt을 직접 지정하는 실수를 막고, 저장 시점을 일관되게 유지할 수 있습니다. `UserEntity`는 `@PrePersist`와 `@PreUpdate` 모두 사용해 createdAt/updatedAt을 관리했습니다.

---

### Q26. Builder 패턴을 Entity에서 사용한 이유는?

> Lombok의 `@Builder`를 붙여 `UserEntity.builder().name().email().password().build()` 방식으로 생성했습니다. 생성자 파라미터 순서를 외울 필요 없이 필드 이름으로 값을 지정하므로 가독성이 높고 실수가 줄어듭니다. `AuthService.signUp()`에서 회원가입 시 빌더로 Entity를 생성하는 코드가 그 예시입니다.

---

### Q27. @EnableAsync와 @Async 동작 원리는?

> `CafeService`에 `@EnableAsync`를 선언하고, `synchronizeCafe()` 메서드에 `@Async`를 붙였습니다. Spring은 `@Async` 메서드를 별도 스레드풀에서 실행하는 프록시를 생성합니다. 프록시를 거쳐야 하므로 **반드시 public 메서드**여야 하고, **같은 클래스 내부에서 직접 호출하면 프록시를 우회**해 동기로 실행됩니다. 때문에 Controller → Service → `synchronizeCafe()` 순으로 외부 호출 구조를 유지했습니다.

---

### 🗃 카테고리 4: DB 설계 (Q28~Q35)

---

### Q28. userId를 UUID로 설계한 이유는?

> `AuthService.signUp()`에서 `UUID.randomUUID().toString()`으로 userId를 생성합니다. Auto-increment 정수 PK는 1, 2, 3처럼 순차적이라 외부에 노출되면 전체 사용자 수 파악이나 순차 탐색 공격에 취약합니다. UUID는 예측 불가능하고 분산 환경에서도 충돌 없이 생성할 수 있어 보안과 확장성이 유리합니다.

---

### Q29. Wishlist에 UNIQUE(user_id, cafe_id, category)를 설정한 이유는?

> 같은 사용자가 같은 카페를 같은 카테고리로 중복 저장하는 것을 DB 레벨에서 방지합니다. 단 같은 카페라도 다른 카테고리(예: WORK, ATMOSPHERE)로는 저장 가능하도록 복합 UNIQUE를 설계했습니다. 애플리케이션 레벨의 중복 체크만으로는 동시 요청 시 Race Condition이 발생할 수 있어 DB 제약조건으로 보완했습니다.

---

### Q30. ST_Distance_Sphere를 사용한 이유는?

> `CafeRepository.findNearbyCafes()`에서 `ST_Distance_Sphere(POINT(longitude, latitude), POINT(:lon, :lat)) <= :radius` 조건으로 반경 내 카페를 조회합니다. 단순 위도/경도 차이 비교는 지구가 구면이라는 사실을 무시해 장거리에서 오차가 커집니다. MySQL의 `ST_Distance_Sphere()`는 하버사인 공식 기반으로 실제 구면 거리를 미터 단위로 계산합니다.

---

### Q31. DDL Auto를 validate로 설정한 이유는?

> `application.yml`에서 `spring.jpa.hibernate.ddl-auto=validate`로 설정했습니다. `create`나 `update`는 실수로 테이블을 재생성하거나 컬럼을 변경해 운영 데이터를 날릴 위험이 있습니다. `validate`는 기존 스키마가 Entity와 일치하는지만 확인하고 변경하지 않아 프로덕션 환경에서 안전합니다. 스키마 변경은 별도의 마이그레이션 스크립트로 관리하는 것이 원칙입니다.

---

### Q32. Enum을 DB에 String으로 저장한 이유는?

> Entity에서 `@Enumerated(EnumType.STRING)`을 사용했습니다. `EnumType.ORDINAL`은 Enum 선언 순서의 정수(0, 1, 2…)를 저장하는데, Enum 중간에 값을 추가하면 기존 데이터와 매핑이 틀어집니다. `EnumType.STRING`은 `"ACTIVE"`, `"USER"` 같은 문자열을 저장해 Enum 순서 변경에 영향받지 않고 DB에서 바로 의미를 파악할 수 있습니다.

---

### Q33. avgRating은 어떻게 갱신하나요?

> 현재 구현에서는 리뷰 작성/수정/삭제 이벤트 시 `CafeEntity.avgRating`을 직접 업데이트하거나, `ReviewRepository.findByCafe_CafeId(cafeId)`로 전체 리뷰를 조회해 평균을 계산하는 방식입니다. 개선 방향으로는 DB의 AVG 집계 쿼리를 사용하거나, 리뷰 수와 합계를 별도 컬럼에 유지하는 증분 업데이트 방식이 더 효율적입니다.

---

### Q34. 외래키 제약조건 위반을 어떻게 처리했나요?

> `WishlistService.toggle()`에서 `em.getReference()`로 프록시 참조를 생성하다가 존재하지 않는 cafeId면 저장 시점에 `DataIntegrityViolationException`이 발생합니다. 이를 `try-catch`로 잡아 클라이언트에 400 에러를 반환했습니다. `GlobalExceptionHandler`에서도 공통으로 처리해 예외별 적절한 HTTP 상태코드가 반환되도록 했습니다.

---

### Q35. 인덱스는 어떻게 고려했나요?

> `WishlistEntity`에 `@Index(columnList = "user_id")`와 `@Index(columnList = "cafe_id")`를 설정해 사용자별 찜 목록 조회와 카페별 찜 수 집계 쿼리의 성능을 높였습니다. `UserEntity`의 `email`은 로그인 시마다 조회되므로 `UNIQUE` 제약이 인덱스 역할을 합니다. `CafeEntity`의 `kakaoId`는 `existsByKakaoId()` 중복 체크에 활용됩니다.

---

### 🌐 카테고리 5: API 설계 / 외부 연동 (Q36~Q43)

---

### Q36. RESTful하게 설계했다고 볼 수 있나요?

> 대체로 REST 원칙을 따랐습니다. 리소스를 명사로 표현하고(`/api/cafes`, `/api/reviews`), HTTP 메서드로 행위를 표현했습니다(GET 조회, POST 생성, PUT 수정, DELETE 삭제). 다만 `/api/auth/logout`이나 `/api/auth/password/reset`처럼 동작이 복잡한 경우 순수 REST보다 RPC에 가까운 URL을 썼습니다. 실용성과 원칙 사이에서 팀과 합의해 결정했습니다.

---

### Q37. ApiResponse\<T\> 래퍼를 사용한 이유는?

> 모든 API 응답을 `{"message": "...", "data": {...}}` 형태로 통일했습니다. 프론트엔드가 응답 형식을 예측할 수 있고, 에러 시에도 같은 구조로 메시지를 전달할 수 있습니다. `ApiResponse.<UserDTO>builder().message("로그인 성공").data(responseDTO).build()` 방식으로 일관되게 사용했습니다.

---

### Q38. multipart/form-data를 어떻게 처리했나요?

> 리뷰 작성 API(`POST /api/cafes/{cafeId}/reviews`)에서 JSON 데이터와 이미지 파일을 함께 전송하기 위해 multipart를 사용했습니다. Controller에서 `@RequestPart String reviewJson`과 `@RequestPart List<MultipartFile> images`를 받아, `ObjectMapper.readValue(reviewJson, ReviewRequestDTO.class)`로 JSON을 파싱합니다. `@RequestBody`로는 파일을 함께 받을 수 없어 이 방식을 선택했습니다.

---

### Q39. 카카오 API 결과를 DB와 어떻게 병합했나요?

> `CafeService.searchAndMerge()`에서 카카오 API로 가져온 결과의 이름 목록을 추출하고, `cafeRepository.findByNameIn(names)`로 DB에서 일치하는 카페를 배치 조회합니다. 카카오 결과를 순회하며 DB에 존재하면 DB 데이터를, 없으면 카카오 데이터로 DTO를 생성합니다. 신규 카페는 `@Async synchronizeCafe(doc, kakaoId)`를 호출해 응답 지연 없이 백그라운드에서 저장합니다.

---

### Q40. @Async의 주의할 점은 무엇인가요?

> 세 가지를 주의했습니다. 첫째, `@Async`는 Spring AOP 프록시를 통해야 동작하므로 **public 메서드**에만 적용됩니다. 둘째, **같은 클래스에서 직접 호출하면 프록시를 우회**해 동기로 실행됩니다. 셋째, `@Async`와 `@Transactional`을 함께 쓰면 별도 스레드에서 트랜잭션이 시작되므로, 호출자의 트랜잭션에 참여하지 않습니다. `synchronizeCafe()`에 `@Transactional`을 별도로 선언한 이유입니다.

---

### Q41. 이미지를 S3에 저장한 이유는?

> 이미지를 서버 로컬 파일시스템에 저장하면 서버가 여러 대로 늘어날 때 파일 동기화 문제가 생기고, 서버 재시작 시 데이터가 날아갈 수 있습니다. AWS S3는 가용성이 높고 CDN과 연동이 쉬우며, SDK v2로 업로드/삭제를 간편하게 구현할 수 있습니다. S3 Key와 URL을 `ImageEntity`에 저장해 나중에 `S3Service.delete(s3Key)`로 파일을 삭제할 수 있습니다.

---

### Q42. Gmail SMTP로 이메일 발송을 어떻게 구현했나요?

> `EmailService.sendTempPasswordEmail()`에서 `JavaMailSender`를 주입받아 `MimeMessage`를 생성하고 HTML 형식의 이메일을 발송합니다. Gmail App Password를 `.env`의 `MAIL_PASSWORD`에 저장하고 `spring-dotenv`로 로드합니다. `AuthService.resetPassword()`에서 10자리 랜덤 임시 비밀번호를 BCrypt로 암호화해 DB에 저장한 뒤 평문을 이메일로 발송합니다.

---

### Q43. Swagger를 도입한 이유는?

> `springdoc-openapi-starter-webmvc-ui`를 의존성에 추가해 `http://localhost:8080/api-docs`에서 API 문서를 확인할 수 있습니다. 프론트엔드 팀과 API 명세를 공유할 때 별도 문서 작성 없이 Swagger UI에서 직접 테스트할 수 있어 협업 효율이 높아졌습니다. Controller에 `@Operation`, `@ApiResponse`, `@ExampleObject`로 요청/응답 예시를 직접 작성했습니다.

---

### ⚡ 카테고리 6: 성능 최적화 (Q44~Q47)

---

### Q44. 조회수를 어떻게 구현했고 개선점은?

> `CafeService.getCafeDetail()`에서 카페 상세 조회 시마다 `entity.setViewCount(entity.getViewCount() + 1)`하고 `cafeRepository.save(entity)`를 호출합니다. 매 조회마다 UPDATE 쿼리가 발생하는 단점이 있습니다. 개선 방향은 Redis에 조회수를 버퍼링했다가 배치로 DB에 반영하는 방식입니다. 현재는 `CafeScheduler`로 `viewsLast7d = viewCount - lastViewCount`를 매일 새벽 3시에 계산합니다.

---

### Q45. Hot Score 공식은 어떻게 설계했나요?

> `CafeRepository.findHotWeightedNative()`에서 네이티브 SQL로 계산합니다. 공식은 `views_last7d × w7d + viewCount × wAll + avgRating × 50 × wRate + reviewCount × 5 × wRev`입니다. 가중치를 파라미터로 받아 `GET /api/cafes/hot10?w7d=3&wAll=1&wRate=2&wRev=2`처럼 외부에서 조정할 수 있게 했습니다. 리뷰 수는 서브쿼리로 집계하며 10개 결과를 반환합니다.

---

### Q46. 카카오 API 검색 성능은 어떻게 최적화했나요?

> 세 가지를 적용했습니다. 첫째, 카카오 API는 최대 45페이지를 조회할 수 있지만 검색 목록에서는 첫 페이지 15개만 가져옵니다(`size=15, page=1`). 둘째, `UriComponentsBuilder.encode(StandardCharsets.UTF_8)`로 한글 인코딩을 안전하게 처리했습니다. 셋째, 결과는 최대 50개로 제한하고 신규 카페 저장은 `@Async`로 비동기 처리해 응답 속도를 보장했습니다.

---

### Q47. CafeScheduler로 어떤 배치 작업을 했나요?

> `@Scheduled(cron = "0 0 3 * * *")`로 매일 새벽 3시에 `cafeRepository.updateViewsLast7d()`를 실행합니다. 이 쿼리는 `views_last7d = viewCount - lastViewCount`를 계산하고 `lastViewCount = viewCount`로 기준점을 업데이트합니다. 결과적으로 `views_last7d`는 항상 최근 7일간의 증분 조회수를 나타내 Hot Score 계산에 활용됩니다. `CafeOnApplication`에 `@EnableScheduling`을 선언했습니다.

---

### 🤝 카테고리 7: 프로젝트 전반 (Q48~Q50)

---

### Q48. 구현하면서 가장 어려웠던 부분은?

> 카카오 API 결과와 DB 데이터를 병합하는 로직이 가장 복잡했습니다. 두 데이터 소스의 카페 식별 기준이 달라(카카오는 `kakaoId`, DB는 이름) 정확히 매칭하기 어려웠고, 신규 카페를 비동기로 저장할 때 트랜잭션 범위와 `@Async` 프록시 동작 방식을 이해하는 데 시간이 걸렸습니다. 결국 `findByNameIn()`으로 이름 기준 매칭하고, `existsByKakaoId()`로 중복 저장을 방지하는 방식으로 해결했습니다.

---

### Q49. 개선하고 싶은 점은?

> 세 가지를 꼽겠습니다. 첫째, **임시 비밀번호 방식**을 UUID 토큰 + 만료시간 DB 저장 방식의 일회용 링크로 교체해 보안을 강화하고 싶습니다. 둘째, 상세 조회마다 발생하는 **조회수 UPDATE를 Redis로 옮겨** DB 부하를 줄이고 싶습니다. 셋째, 시간 부족으로 구현하지 못한 **OAuth2 소셜 로그인**(Google, Kakao)을 Spring Security OAuth2 Client로 추가하고 싶습니다.

---

### Q50. 팀 협업은 어떻게 했나요?

> 백엔드 3명이 도메인 단위로 역할을 분담했습니다. 저는 User(인증), Cafe, Review, Wishlist를 담당했고, 팀원들은 Chat(WebSocket), Community(게시판/댓글), Admin 파트를 맡았습니다. Git Flow 전략으로 `dev` 브랜치를 기준으로 기능별 브랜치를 생성해 PR로 합쳤습니다. API 명세는 공유 문서와 Swagger로 관리해 프론트엔드 팀과 연결했습니다.
