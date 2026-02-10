---
title: "Swagger로 백엔드-모바일 협업하기 - API 스펙 문서화와 커뮤니케이션"
date: 2024-03-15
categories: [backend]
tags: [swagger, api, collaboration, documentation, spring-boot]
---

"이 API 응답 필드명이 뭐였죠?", "Boolean인가요 String인가요?" 프론트엔드 개발자와 매일 주고받던 질문들이다. 배차 서비스를 만들면서 플러터 개발자와 Swagger 기반으로 협업한 경험을 정리한다.

## 문제

고소작업차 배차 서비스를 개발하면서 백엔드(나)와 모바일(플러터 개발자) 2명이 협업했다. 초기에는 카카오톡으로 API 스펙을 공유했다.

```
[나] 배차 목록 조회 API 만들었습니다
GET /api/dispatches?status=PENDING
응답:
{
  "dispatches": [
    {
      "id": 1,
      "siteAddress": "서울시 강남구...",
      "workDate": "2024-03-15",
      ...
    }
  ]
}

[플러터 개발자] workDate가 String인가요 Date인가요?
[나] String입니다. "YYYY-MM-DD" 형식이에요
[플러터 개발자] price 필드는 없나요?
[나] 아 빠뜨렸네요. price도 있습니다 (Integer)
```

문제점:
- 카톡으로 스펙 공유 → 나중에 찾기 어렵다
- 필드 타입 질문 반복 (String? Integer? Boolean?)
- 응답 예시가 실제 API와 다를 때가 있다
- API 수정 시 변경사항 전달 누락

## 해결: Swagger 도입

Swagger(OpenAPI)로 API 스펙을 자동 문서화하고, 이 문서를 기준으로 협업하기로 했다.

### 1. Spring Boot Swagger 설정

```java
// build.gradle
dependencies {
    implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:2.2.0'
}
```

```java
// SwaggerConfig.java
@Configuration
public class SwaggerConfig {

    @Bean
    public OpenAPI openAPI() {
        return new OpenAPI()
            .info(new Info()
                .title("배차 서비스 API")
                .version("v1.0")
                .description("고소작업차 실시간 배차 서비스 API 문서"));
    }
}
```

서버 실행 후 `http://localhost:8080/swagger-ui.html` 접속하면 자동 생성된 API 문서를 볼 수 있다.

### 2. DTO에 Swagger 어노테이션 추가

```java
@Schema(description = "배차 응답")
public class DispatchResponse {

    @Schema(description = "배차 ID", example = "1")
    private Long id;

    @Schema(description = "현장 주소", example = "서울시 강남구 테헤란로 123")
    private String siteAddress;

    @Schema(description = "작업 일자 (YYYY-MM-DD)", example = "2024-03-15")
    private String workDate;

    @Schema(description = "작업 시작 시간 (HH:mm)", example = "09:00")
    private String workStartTime;

    @Schema(description = "작업 높이 (미터)", example = "15")
    private Integer workHeight;

    @Schema(description = "단가 (원)", example = "150000")
    private Integer price;

    @Schema(description = "배차 상태", example = "PENDING",
            allowableValues = {"PENDING", "ACCEPTED", "COMPLETED", "CANCELLED"})
    private String status;

    @Schema(description = "수락한 기사 ID (미수락 시 null)", example = "123", nullable = true)
    private Long acceptedDriverId;
}
```

### 3. Controller에 API 설명 추가

```java
@RestController
@RequestMapping("/api/dispatches")
@Tag(name = "배차", description = "배차 관리 API")
@RequiredArgsConstructor
public class DispatchController {

    private final DispatchService dispatchService;

    @Operation(summary = "배차 목록 조회", description = "상태별 배차 목록을 조회합니다")
    @ApiResponses({
        @ApiResponse(responseCode = "200", description = "조회 성공"),
        @ApiResponse(responseCode = "401", description = "인증 실패")
    })
    @GetMapping
    public ResponseEntity<List<DispatchResponse>> getDispatches(
            @Parameter(description = "배차 상태 (미입력 시 전체)", required = false)
            @RequestParam(required = false) DispatchStatus status) {

        List<DispatchResponse> dispatches = dispatchService.getDispatches(status);
        return ResponseEntity.ok(dispatches);
    }

    @Operation(summary = "배차 수락", description = "기사가 배차를 수락합니다")
    @PostMapping("/{dispatchId}/accept")
    public ResponseEntity<DispatchAcceptResponse> acceptDispatch(
            @Parameter(description = "배차 ID") @PathVariable Long dispatchId,
            @AuthDriver Driver driver) {

        DispatchAcceptResponse response = dispatchService.acceptDispatch(dispatchId, driver.getId());
        return ResponseEntity.ok(response);
    }
}
```

### 4. Swagger UI에서 실제 API 테스트

Swagger UI에는 "Try it out" 버튼이 있어서 브라우저에서 직접 API를 호출할 수 있다.

```
GET /api/dispatches?status=PENDING
[Try it out] → [Execute] 버튼 클릭

Response:
200 OK
{
  "dispatches": [
    {
      "id": 1,
      "siteAddress": "서울시 강남구 테헤란로 123",
      "workDate": "2024-03-15",
      "workStartTime": "09:00",
      "workHeight": 15,
      "price": 150000,
      "status": "PENDING",
      "acceptedDriverId": null
    }
  ]
}
```

플러터 개발자가 직접 테스트해볼 수 있어서 "이 API 어떻게 호출하나요?"라는 질문이 줄었다.

## 협업 프로세스 변경

### Before: 카톡 기반 협업

```
1. 백엔드 API 개발
2. 카톡으로 스펙 전달 (복사 붙여넣기)
3. 플러터 개발자가 질문 (타입, 필드명 등)
4. 답변
5. 플러터 개발자가 API 호출 구현
6. 실제 호출 시 에러 발생
7. 다시 카톡으로 확인
```

### After: Swagger 기반 협업

```
1. 백엔드 API 개발 (Swagger 어노테이션 포함)
2. Swagger URL 공유 (1회만)
3. 플러터 개발자가 Swagger UI에서 직접 확인
4. "Try it out"으로 실제 API 테스트
5. 타입, 필드명 확인 후 플러터 모델 작성
6. 통합 테스트
```

## Swagger 작성 규칙

팀 내에서 합의한 Swagger 작성 규칙:

1. **모든 DTO 필드에 @Schema 필수**
   - description: 필드 설명
   - example: 예시 값
   - nullable: null 가능 여부 명시

2. **Enum 타입은 allowableValues 명시**
   ```java
   @Schema(description = "배차 상태",
           allowableValues = {"PENDING", "ACCEPTED", "COMPLETED", "CANCELLED"})
   private String status;
   ```

3. **날짜/시간 포맷 명시**
   ```java
   @Schema(description = "작업 일자 (YYYY-MM-DD)", example = "2024-03-15")
   private String workDate;
   ```

4. **응답 코드별 설명 추가**
   ```java
   @ApiResponses({
       @ApiResponse(responseCode = "200", description = "조회 성공"),
       @ApiResponse(responseCode = "400", description = "잘못된 요청"),
       @ApiResponse(responseCode = "401", description = "인증 실패"),
       @ApiResponse(responseCode = "404", description = "배차 없음")
   })
   ```

## 실제 협업 개선 사례

### 1. 필드 타입 질문 감소

**Before:**
- "workHeight가 String인가요 Integer인가요?" (하루 평균 5~6회)

**After:**
- Swagger에 타입이 명시돼있어서 질문 거의 없음

### 2. API 변경사항 전달 자동화

기존에는 API 수정 시 카톡으로 "price 필드 추가했습니다" 같은 메시지를 보냈다. Swagger는 코드 변경 시 자동 반영되므로 별도 전달 불필요.

```java
// 필드 추가
@Schema(description = "배차 마감 시간 (HH:mm)", example = "18:00")
private String deadline;  // 추가

// 서버 재시작 → Swagger UI 자동 업데이트
```

### 3. 프론트엔드 모델 자동 생성

플러터 개발자가 Swagger JSON을 다운로드해서 모델 클래스를 자동 생성했다.

```bash
# Swagger JSON 다운로드
curl http://localhost:8080/v3/api-docs > swagger.json

# Dart 모델 자동 생성 (openapi-generator 사용)
openapi-generator generate -i swagger.json -g dart -o lib/models
```

이렇게 생성된 Dart 모델 클래스는 백엔드 DTO와 100% 일치한다.

### 4. 커뮤니케이션 비용 감소

카톡 메시지 분석 결과:
- Swagger 도입 전: API 관련 카톡 메시지 하루 평균 20~30건
- Swagger 도입 후: API 관련 카톡 메시지 하루 평균 5건

85% 감소. 대부분 "이 필드가 뭐죠?", "타입이 뭐죠?" 같은 단순 질문이 사라졌다.

## Swagger 유지보수 팁

1. **코드 우선, 문서는 자동 생성**
   - 수동으로 Swagger YAML 작성하지 않는다
   - 코드에 어노테이션만 추가하면 자동 생성

2. **예시 값은 실제 데이터와 비슷하게**
   ```java
   // 나쁜 예
   @Schema(example = "1")
   private String siteAddress;

   // 좋은 예
   @Schema(example = "서울시 강남구 테헤란로 123")
   private String siteAddress;
   ```

3. **deprecated API는 명시**
   ```java
   @Deprecated
   @Operation(summary = "배차 목록 조회 (구버전)", deprecated = true)
   @GetMapping("/v1/dispatches")
   public ResponseEntity<?> getDispatchesV1() { ... }
   ```

4. **인증이 필요한 API는 Security 스키마 추가**
   ```java
   @SecurityRequirement(name = "bearer-token")
   @GetMapping("/me")
   public ResponseEntity<DriverResponse> getMyInfo(@AuthDriver Driver driver) { ... }
   ```

## 한계

1. **Swagger 작성 비용이 있다.** 어노테이션을 추가하는 시간이 필요하다. 하지만 카톡으로 스펙 설명하는 시간보다 짧다.

2. **복잡한 비즈니스 로직은 설명 부족.** "이 API를 언제 호출하나요?", "선후 관계가 어떻게 되나요?" 같은 건 Swagger만으로 설명 어렵다. 별도 문서나 대화가 필요하다.

3. **런타임 검증은 안 된다.** Swagger는 문서일 뿐, 실제 응답이 문서와 다를 수 있다. 통합 테스트가 여전히 필요하다.

## 배운 점

1. **문서화는 협업의 시작이다.** 같은 내용을 여러 번 설명하는 시간을 줄이고, 그 시간에 기능 개발에 집중할 수 있다.

2. **코드에서 자동 생성되는 문서가 제일 정확하다.** 수동으로 작성한 문서는 코드와 달라질 수 있다. 코드를 문서의 단일 진실 공급원(Single Source of Truth)으로 만든다.

3. **Try it out 기능이 강력하다.** 플러터 개발자가 백엔드 서버를 띄우지 않고도 브라우저에서 바로 테스트할 수 있다. Postman 없이도 충분하다.

4. **협업 도구는 팀 전체가 사용해야 효과가 있다.** 나만 Swagger를 작성하고 상대방이 안 보면 의미 없다. 팀 내 합의가 중요하다.

## 마무리

Swagger는 "API 문서 도구"가 아니라 "협업 도구"다. 백엔드와 프론트엔드가 같은 문서를 보고, 같은 스펙을 이해하고, 같은 타입으로 통신한다. 카톡으로 "이 필드가 뭐죠?"라고 물어보는 시간을 줄이고, 기능 개발에 집중할 수 있게 해준다. 도입 비용은 작지만 협업 효율은 크게 올라간다.
