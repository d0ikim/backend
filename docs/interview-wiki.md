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
9. [면접 예상 질문 & 모범 답변](#9-면접-예상-질문--모범-답변)

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

## 9. 면접 예상 질문 & 모범 답변

### Q1. JWT를 사용한 이유가 뭔가요?

> 세션 방식은 서버에 상태를 저장해서 서버가 여러 대일 때 세션 동기화 문제가 생깁니다. JWT는 토큰 자체에 사용자 정보를 담아서 서버가 무상태(Stateless)로 동작할 수 있어 수평 확장에 유리합니다. 저희 서비스는 AWS EC2에 배포할 예정이었고, 확장성을 고려해 JWT를 선택했습니다.

### Q2. Refresh Token을 DB에 저장하는 이유가 뭔가요?

> Refresh Token을 DB에 저장하지 않으면, 토큰이 탈취됐을 때 서버에서 무효화할 방법이 없습니다. 로그아웃 시 DB의 refresh_token을 null로 만들면, 이후 갱신 요청을 모두 거부할 수 있습니다. 또한 Token Rotation 전략으로 갱신할 때마다 새 Refresh Token을 발급하고 이전 것은 무효화해서 탈취된 토큰 재사용을 방지했습니다.

### Q3. Token Rotation이 무엇이고 왜 적용했나요?

> Refresh Token을 한 번 사용하면 새로운 Refresh Token으로 교체하는 전략입니다. 만약 Refresh Token이 탈취됐다면, 정상 사용자가 먼저 갱신 요청을 하면 토큰이 교체되고, 공격자가 나중에 같은 Refresh Token으로 요청하면 DB 값과 다르다는 것을 감지해 거부할 수 있습니다.

### Q4. 카카오 API 결과를 DB와 병합한 이유는요?

> 카카오 API만 쓰면 서비스 고유 데이터(리뷰 요약, 태그, 평점)가 없고, DB만 쓰면 카카오에만 있는 신규 카페가 누락됩니다. 두 데이터를 이름 기준으로 병합해서 DB에 있는 카페는 내부 데이터를, 없는 카페는 카카오 데이터를 보여주고 백그라운드에서 비동기로 DB에 저장해 다음 검색 시 내부 데이터를 활용할 수 있게 했습니다.

### Q5. @Async로 비동기 저장한 이유가 뭔가요?

> 검색 요청마다 신규 카페를 동기로 DB에 저장하면 응답 지연이 생깁니다. 사용자 입장에서는 저장 완료를 기다릴 필요가 없으니 @Async로 별도 스레드에서 처리해 응답 속도를 개선했습니다. 단 @Async 메서드는 프록시를 통해야 작동해서 반드시 public 메서드로 분리했습니다.

### Q6. N+1 문제를 어떻게 해결했나요?

> 카페 목록 100개에서 각각 태그를 조회하면 101번 쿼리가 발생합니다. `findTagNamesByCafeIds(cafeIds)` 쿼리를 만들어 카페 ID 리스트를 IN 절로 한 번에 조회하고, 결과를 `Map<Long, List<String>>`으로 변환해 카페별로 매핑했습니다. 리뷰도 동일한 방식으로 `getReviewsByCafeIds(cafeIds)`를 사용했습니다.

### Q7. Soft Delete를 적용한 이유가 뭔가요?

> 사용자 데이터를 물리적으로 삭제하면 해당 유저가 작성한 리뷰, 게시글 등과의 관계가 끊어져 데이터 정합성이 깨집니다. Hibernate의 @SQLDelete와 @Where를 사용해 deleted_at 컬럼으로 논리 삭제를 구현했습니다. 탈퇴한 사용자는 닉네임을 '탈퇴회원'으로 변경해 게시글/댓글에서 표시할 수 있게 했습니다.

### Q8. BCrypt를 선택한 이유는요?

> BCrypt는 단방향 해시 + salt를 자동 적용해서 같은 비밀번호도 매번 다른 해시값이 나옵니다. Rainbow Table 공격을 방어하고, work factor를 조절해 연산 비용을 높일 수 있습니다. Spring Security가 기본 제공하는 PasswordEncoder여서 자연스럽게 선택했습니다.

### Q9. ST_Distance_Sphere를 사용한 이유는요?

> 위도/경도 좌표 간 거리를 계산할 때 평면 거리 공식을 쓰면 지구의 곡률이 고려되지 않아 오차가 생깁니다. MySQL에서 제공하는 `ST_Distance_Sphere()`는 하버사인 공식 기반으로 지구 구면 거리를 정확하게 계산합니다. 반경 검색이라 인덱스 활용이 어렵지만, LIMIT 100으로 결과를 제한해 성능을 보완했습니다.

### Q10. 임시 비밀번호 발급 시 보안 고려사항은요?

> 임시 비밀번호를 평문으로 이메일 발송하므로 이메일 탈취 시 취약할 수 있습니다. 이를 보완하려면 임시 비밀번호 대신 일회용 링크(UUID 토큰 + 만료시간 DB 저장)를 발송하는 방식이 더 안전합니다. 프로젝트 기간 내 간단한 구현으로 임시 비밀번호 방식을 택했지만, 실서비스라면 링크 방식을 고려할 것입니다.
