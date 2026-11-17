# API 명세서

> 기능 ID는 [기능 명세서](./feature-spec.md)를 따른다.
> 와이어프레임에 없지만 직접 정한 내용은 **(결정)** 으로 표시했다.

## 0. 공통 규약

| 항목 | 규칙 |
| --- | --- |
| 기본 경로 | 모든 API는 `/api`로 시작한다. |
| 데이터 형식 | 요청·응답 본문은 JSON(UTF-8). 필드 이름은 `camelCase`. 파일이 들어가는 요청만 `multipart/form-data`. |
| 인증 | **세션 방식** (결정). 로그인에 성공하면 서버가 세션 쿠키(`JSESSIONID`)를 내려 주고, 이후 요청에 쿠키가 함께 간다. 표의 인증 열: `공개` = 로그인 불필요, `로그인` = 로그인 필요. |
| 시각 | ISO-8601 UTC 문자열. 예: `2026-11-18T02:30:00Z` |
| ID | 숫자. URL의 ID로 남의 과제·댓글을 수정·삭제하려 하면 서버가 작성자를 확인해 거절한다. |
| 글자 수 | 앞뒤 공백을 제거한 뒤 글자 단위로 센다. |
| 작성자 표시 | 탈퇴한 회원이 작성자이면 `author.nickname`은 `"탈퇴한 회원"`, `author.withdrawn`은 `true`. |
| 성공 상태 코드 | 조회·수정 `200 OK`, 생성 `201 Created`, 본문 없음 `204 No Content` |
| 페이지네이션 | 요청 `page`(0부터, 기본 0), `size`(기본 10, 최대 50). 응답은 아래 형식. 화면에 보이는 페이지 번호는 `page + 1`이다. |
| 오류 응답 | 모든 API에서 아래 형식으로 통일한다. 프론트는 `code`로 분기한다. |

**페이지 응답 형식**
```json
{
  "items": [],
  "page": 0,
  "size": 10,
  "totalElements": 13,
  "totalPages": 2
}
```

**오류 응답 형식**
```json
{
  "code": "INVALID_INPUT",
  "message": "입력값을 확인해주세요.",
  "errors": [
    { "field": "title", "reason": "제목은 앞뒤 공백을 제외하고 2~100자로 입력해주세요." }
  ]
}
```
`errors`는 입력값 오류(`INVALID_INPUT`)일 때만 채우고, 그 밖에는 빈 배열이다.

---

## 1. API 목록

| 기능 ID | Method | URL | 설명 | 인증 |
| --- | --- | --- | --- | --- |
| F-19 | `GET` | `/api/summary` | 사이드바 요약 | 공개 |
| F-01 | `POST` | `/api/members` | 스터디 가입 | 공개 |
| F-02 | `POST` | `/api/auth/login` | 로그인 | 공개 |
| F-03 | `POST` | `/api/auth/logout` | 로그아웃 | 로그인 |
| F-04 | `GET` | `/api/members/me` | 내 정보 조회 | 로그인 |
| F-05 | `DELETE` | `/api/members/me` | 스터디 탈퇴 | 로그인 |
| F-06 | `GET` | `/api/members` | 회원 목록 조회·검색 | 로그인 |
| F-07, F-08 | `GET` | `/api/posts` | 과제 목록 조회·검색 | 로그인 |
| F-09 | `GET` | `/api/members/me/posts` | 내가 제출한 과제 조회·검색 | 로그인 |
| F-10 | `GET` | `/api/posts/{postId}` | 과제 상세 조회 | 로그인 |
| F-11 | `POST` | `/api/posts` | 과제 제출 | 로그인 |
| F-12 | `PUT` | `/api/posts/{postId}` | 과제 수정 | 로그인 (작성자) |
| F-13 | `DELETE` | `/api/posts/{postId}` | 과제 삭제 | 로그인 (작성자) |
| F-14 | `GET` | `/api/attachments/{attachmentId}` | 첨부파일 다운로드 | 로그인 |
| F-10 | `GET` | `/api/posts/{postId}/comments` | 댓글·답글 목록 | 로그인 |
| F-15, F-16 | `POST` | `/api/posts/{postId}/comments` | 댓글·답글 작성 | 로그인 |
| F-17 | `PATCH` | `/api/comments/{commentId}` | 댓글·답글 수정 | 로그인 (작성자) |
| F-18 | `DELETE` | `/api/comments/{commentId}` | 댓글·답글 삭제 | 로그인 (작성자) |

---

## 2. 회원·인증

### 사이드바 요약 (F-19)

`GET /api/summary` · 공개

- 성공 `200 OK`
  ```json
  { "activeMemberCount": 4, "postCount": 13 }
  ```

### 스터디 가입 (F-01)

`POST /api/members` · 공개

- 요청 Body
  ```json
  {
    "name": "이수현",
    "nickname": "수현",
    "email": "suhyun@example.com",
    "password": "study1234!",
    "interest": "API 설계",
    "bio": "스프링을 공부하고 있어요."
  }
  ```

  | 필드 | 필수 | 제약 |
  | --- | --- | --- |
  | `name` | O | 2~30자 |
  | `nickname` | O | 2~20자, 활동 회원끼리 중복 불가 (대소문자 무시) |
  | `email` | O | 이메일 형식, 최대 100자, 활동 회원끼리 중복 불가. 소문자로 저장 |
  | `password` | O | 8~64자 |
  | `interest` | X | 최대 50자 |
  | `bio` | X | 최대 200자 |

  비밀번호 확인과 공개 정보 확인 체크는 화면에서만 검사하고 보내지 않는다. (결정)

- 성공 `201 Created` · 가입과 동시에 로그인 세션이 만들어진다.
  ```json
  {
    "memberId": 5,
    "nickname": "수현"
  }
  ```

- 실패

  | 상태 | 코드 | 상황 |
  | --- | --- | --- |
  | 400 | `INVALID_INPUT` | 길이·형식 위반 (`errors`에 필드별 사유) |
  | 409 | `DUPLICATE_EMAIL` | 이미 사용 중인 이메일 |
  | 409 | `DUPLICATE_NICKNAME` | 이미 사용 중인 닉네임 |

### 로그인 (F-02)

`POST /api/auth/login` · 공개

- 요청 Body
  ```json
  { "email": "doyun@example.com", "password": "study1234!" }
  ```
- 성공 `200 OK` · 응답 헤더 `Set-Cookie: JSESSIONID=...`
  ```json
  { "memberId": 1, "nickname": "도윤" }
  ```
- 실패

  | 상태 | 코드 | 상황 |
  | --- | --- | --- |
  | 400 | `INVALID_INPUT` | 이메일·비밀번호가 비어 있음 |
  | 401 | `LOGIN_FAILED` | 없는 이메일, 틀린 비밀번호, 탈퇴한 회원 (구분하지 않음) |

### 로그아웃 (F-03)

`POST /api/auth/logout` · 로그인

- 성공 `204 No Content` · 세션을 끝낸다.
- 실패: 이미 로그아웃된 상태여도 `204`를 준다. (결정)

### 내 정보 조회 (F-04)

`GET /api/members/me` · 로그인

- 성공 `200 OK`
  ```json
  {
    "memberId": 1,
    "name": "김도윤",
    "nickname": "도윤",
    "email": "doyun@example.com",
    "interest": "API 설계 · Spring",
    "bio": "작은 기능도 끝까지 구현하고 기록하려고 합니다.",
    "joinedAt": "2026-10-19T00:00:00Z"
  }
  ```
  `interest`, `bio`는 없으면 `null`.
- 실패

  | 상태 | 코드 | 상황 |
  | --- | --- | --- |
  | 401 | `UNAUTHORIZED` | 로그인하지 않음 |

### 스터디 탈퇴 (F-05)

`DELETE /api/members/me` · 로그인

- 요청 Body: 없음. 체크와 확인 문구 `탈퇴`는 화면에서 검사한다. (결정)
- 성공 `204 No Content`
  - 탈퇴 시각을 기록하고 이름·닉네임·이메일·관심 분야·자기소개·비밀번호를 비운다.
  - 세션을 끝낸다. 과제·첨부·댓글은 그대로 둔다.
- 실패

  | 상태 | 코드 | 상황 |
  | --- | --- | --- |
  | 401 | `UNAUTHORIZED` | 로그인하지 않음 |

### 회원 목록 조회·검색 (F-06)

`GET /api/members?query={검색어}` · 로그인

- Query

  | 이름 | 필수 | 설명 |
  | --- | --- | --- |
  | `query` | X | 닉네임 또는 관심 분야에 부분 일치, 대소문자 무시. 없으면 전체 |

- 성공 `200 OK` · 활동 회원만, 가입일 최신순. 페이지 나누기 없음.
  ```json
  {
    "activeMemberCount": 4,
    "items": [
      {
        "memberId": 4,
        "nickname": "유진",
        "interest": "프론트 협업 · HTTP",
        "bio": "API를 사용하는 입장에서 함께 의견을 나누고 싶어요.",
        "joinedAt": "2026-10-22T02:00:00Z",
        "me": false
      }
    ]
  }
  ```
  이름과 이메일은 응답에 넣지 않는다.
- 실패

  | 상태 | 코드 | 상황 |
  | --- | --- | --- |
  | 401 | `UNAUTHORIZED` | 로그인하지 않음 |

---

## 3. 과제

### 과제 목록 조회·검색 (F-07, F-08)

`GET /api/posts?page=0&size=10&scope=ALL&query=api` · 로그인

- Query

  | 이름 | 필수 | 설명 |
  | --- | --- | --- |
  | `page` | X | 0부터. 기본 0 |
  | `size` | X | 기본 10, 최대 50 |
  | `scope` | X | `ALL`(제목+내용), `TITLE`, `AUTHOR`. 기본 `ALL` |
  | `query` | X | 부분 일치, 대소문자 무시. 없으면 전체 |

- 정렬: 작성일 최신순, 같으면 `postId` 큰 순
- 성공 `200 OK`
  ```json
  {
    "items": [
      {
        "postId": 101,
        "title": "[2주차] 회원 API 명세 초안과 예외 처리 정리",
        "author": { "memberId": 1, "nickname": "도윤", "withdrawn": false },
        "commentCount": 3,
        "attachmentCount": 2,
        "createdAt": "2026-11-11T04:00:00Z"
      }
    ],
    "page": 0,
    "size": 10,
    "totalElements": 13,
    "totalPages": 2
  }
  ```
  `commentCount`는 삭제된 댓글을 뺀 수. 결과가 없거나 없는 페이지를 요청하면 `items`가 빈 배열이다. (결정)
- 실패

  | 상태 | 코드 | 상황 |
  | --- | --- | --- |
  | 400 | `INVALID_INPUT` | `page`가 음수, `size`가 1~50이 아님, 숫자가 아님 |
  | 400 | `INVALID_SEARCH_SCOPE` | `scope`가 정해진 세 값이 아님 |
  | 401 | `UNAUTHORIZED` | 로그인하지 않음 |

### 내가 제출한 과제 조회·검색 (F-09)

`GET /api/members/me/posts?page=0&size=10&scope=ALL&query=` · 로그인

- Query, 정렬, 응답, 실패: [과제 목록 조회](#과제-목록-조회검색-f-07-f-08)와 같다. 로그인한 회원이 쓴 과제만 돌려준다.

### 과제 상세 조회 (F-10)

`GET /api/posts/{postId}` · 로그인

- 성공 `200 OK`
  ```json
  {
    "postId": 101,
    "title": "[2주차] 회원 API 명세 초안과 예외 처리 정리",
    "content": "이번 과제에서는 가입부터 탈퇴까지…",
    "author": { "memberId": 1, "nickname": "도윤", "withdrawn": false },
    "mine": true,
    "createdAt": "2026-11-11T04:00:00Z",
    "updatedAt": null,
    "commentCount": 3,
    "attachments": [
      {
        "attachmentId": 7,
        "originalName": "회원-API-명세서.md",
        "contentType": "text/markdown",
        "sizeBytes": 412
      }
    ]
  }
  ```
  `mine`은 로그인한 회원이 작성자인지 여부다. 화면의 [수정]·[삭제] 버튼 표시에 쓴다. `updatedAt`이 `null`이 아니면 "수정됨"을 표시한다.
- 실패

  | 상태 | 코드 | 상황 |
  | --- | --- | --- |
  | 401 | `UNAUTHORIZED` | 로그인하지 않음 |
  | 404 | `POST_NOT_FOUND` | 없거나 삭제된 과제 |

### 과제 제출 (F-11)

`POST /api/posts` · 로그인 · `Content-Type: multipart/form-data`

- 요청 (form-data)

  | 파트 | 타입 | 필수 | 제약 |
  | --- | --- | --- | --- |
  | `title` | 텍스트 | O | 앞뒤 공백 제외 2~100자 |
  | `content` | 텍스트 | O | 앞뒤 공백 제외 1~10,000자 |
  | `files` | 파일 (여러 개) | X | 최대 5개, 하나당 10MB, 합계 30MB, 허용 확장자만 |

  허용 확장자: `pdf`, `zip`, `md`, `txt`, `png`, `jpg`, `jpeg`, `webp`, `docx`, `pptx`, `json`, `sql`

  JSON이 아닌 multipart를 쓰는 이유: 파일은 JSON 문자열로 담을 수 없어서, 텍스트와 파일을 한 요청에 함께 보내려면 multipart가 필요하다.

- 성공 `201 Created` · 응답 헤더 `Location: /api/posts/114`
  ```json
  { "postId": 114 }
  ```
- 실패 · 첨부 조건을 하나라도 어기면 과제도 저장하지 않는다. (결정)

  | 상태 | 코드 | 상황 |
  | --- | --- | --- |
  | 400 | `INVALID_INPUT` | 제목·내용 길이 위반 |
  | 400 | `TOO_MANY_FILES` | 파일 5개 초과 |
  | 400 | `UNSUPPORTED_FILE_TYPE` | 허용하지 않는 확장자 |
  | 400 | `FILE_TOO_LARGE` | 파일 하나가 10MB 초과 |
  | 400 | `TOTAL_FILE_SIZE_EXCEEDED` | 합계 30MB 초과 |
  | 401 | `UNAUTHORIZED` | 로그인하지 않음 |

### 과제 수정 (F-12)

`PUT /api/posts/{postId}` · 로그인 (작성자) · `Content-Type: multipart/form-data`

- 요청 (form-data)

  | 파트 | 타입 | 필수 | 설명 |
  | --- | --- | --- | --- |
  | `title` | 텍스트 | O | F-11과 같음 |
  | `content` | 텍스트 | O | F-11과 같음 |
  | `keepAttachmentIds` | 숫자 (여러 개) | X | 남길 기존 첨부 ID. 보내지 않은 기존 첨부는 삭제한다 |
  | `files` | 파일 (여러 개) | X | 새로 추가할 파일 |

  개수·용량 제한은 `keepAttachmentIds`의 파일과 `files`를 **합쳐서** 검사한다.

  `PATCH`가 아닌 `PUT`을 쓰는 이유: 수정 화면은 바뀐 칸만이 아니라 제목, 내용, 남길 파일 목록을 **전부** 보내고, 서버는 그 상태로 통째로 바꾸기 때문이다.

- 성공 `200 OK`
  ```json
  { "postId": 101, "updatedAt": "2026-11-18T03:00:00Z" }
  ```
- 실패

  | 상태 | 코드 | 상황 |
  | --- | --- | --- |
  | 400 | `INVALID_INPUT` | 제목·내용 길이 위반 |
  | 400 | `TOO_MANY_FILES` 외 첨부 오류 | F-11과 같음 |
  | 400 | `INVALID_ATTACHMENT` | `keepAttachmentIds`에 이 과제의 첨부가 아닌 ID가 있음 |
  | 401 | `UNAUTHORIZED` | 로그인하지 않음 |
  | 403 | `NOT_POST_AUTHOR` | 작성자가 아님 (탈퇴 후 다시 가입한 경우 포함) |
  | 404 | `POST_NOT_FOUND` | 없거나 삭제된 과제 |

### 과제 삭제 (F-13)

`DELETE /api/posts/{postId}` · 로그인 (작성자)

- 성공 `204 No Content` · 댓글 → 첨부(저장된 파일 포함) → 과제 순으로 완전히 삭제한다.
- 실패

  | 상태 | 코드 | 상황 |
  | --- | --- | --- |
  | 401 | `UNAUTHORIZED` | 로그인하지 않음 |
  | 403 | `NOT_POST_AUTHOR` | 작성자가 아님 |
  | 404 | `POST_NOT_FOUND` | 없거나 이미 삭제된 과제 |

### 첨부파일 다운로드 (F-14)

`GET /api/attachments/{attachmentId}` · 로그인

- 성공 `200 OK` · 본문은 JSON이 아니라 **파일 그 자체**다.
  ```http
  Content-Type: text/markdown
  Content-Length: 412
  Content-Disposition: attachment; filename*=UTF-8''%ED%9A%8C%EC%9B%90-API-%EB%AA%85%EC%84%B8%EC%84%9C.md
  ```
  `Content-Disposition`은 브라우저가 화면에 띄우지 않고 내려받게 하고, 한글 원래 파일 이름을 유지하게 한다.
- 실패 (JSON 오류 응답)

  | 상태 | 코드 | 상황 |
  | --- | --- | --- |
  | 401 | `UNAUTHORIZED` | 로그인하지 않음 |
  | 404 | `ATTACHMENT_NOT_FOUND` | 없거나 삭제된 파일 |

---

## 4. 댓글

### 댓글·답글 목록 (F-10)

`GET /api/posts/{postId}/comments` · 로그인

- 성공 `200 OK` · 최상위 댓글과 답글 모두 작성일 오래된 순(같으면 ID 작은 순). 답글은 부모의 `replies`에 넣는다.
  ```json
  {
    "commentCount": 3,
    "items": [
      {
        "commentId": 201,
        "content": "탈퇴한 회원의 닉네임 대신 공통 이름을 표시하면 좋겠어요.",
        "author": { "memberId": 2, "nickname": "서연", "withdrawn": false },
        "mine": false,
        "deleted": false,
        "createdAt": "2026-11-11T04:10:00Z",
        "updatedAt": null,
        "replies": [
          {
            "commentId": 202,
            "content": "좋아요. 기존 글과 댓글은 유지하고 작성자 표시는 통일하는 것으로 반영해볼게요.",
            "author": { "memberId": 1, "nickname": "도윤", "withdrawn": false },
            "mine": true,
            "deleted": false,
            "createdAt": "2026-11-11T04:20:00Z",
            "updatedAt": null
          }
        ]
      }
    ]
  }
  ```
  - 삭제됐지만 살아 있는 답글이 있는 최상위 댓글: `deleted: true`, `content`와 `author`는 `null`. 화면은 "삭제된 댓글입니다."를 표시한다.
  - 답글이 없는 삭제된 댓글과 삭제된 답글은 응답에서 뺀다.
  - `commentCount`는 삭제된 댓글을 뺀 수다.
- 실패

  | 상태 | 코드 | 상황 |
  | --- | --- | --- |
  | 401 | `UNAUTHORIZED` | 로그인하지 않음 |
  | 404 | `POST_NOT_FOUND` | 없거나 삭제된 과제 |

  댓글을 상세 응답에 넣지 않고 따로 둔 이유: 댓글을 쓰거나 지운 뒤에 과제 본문과 첨부까지 다시 받지 않고 댓글만 새로 받으면 된다. (결정)

### 댓글·답글 작성 (F-15, F-16)

`POST /api/posts/{postId}/comments` · 로그인

- 요청 Body
  ```json
  { "content": "프론트에서는 오류 사유가 구분되면 좋겠습니다.", "parentCommentId": null }
  ```

  | 필드 | 필수 | 제약 |
  | --- | --- | --- |
  | `content` | O | 앞뒤 공백 제외 1~1,000자 |
  | `parentCommentId` | X | 답글이면 부모 댓글 ID. 최상위 댓글이면 `null` 또는 생략 |

- 성공 `201 Created`
  ```json
  { "commentId": 205 }
  ```
- 실패

  | 상태 | 코드 | 상황 |
  | --- | --- | --- |
  | 400 | `INVALID_INPUT` | 내용 길이 위반 |
  | 400 | `INVALID_PARENT_COMMENT` | 부모가 답글임(2단계 금지), 부모가 삭제됨, 부모가 다른 과제의 댓글임 |
  | 401 | `UNAUTHORIZED` | 로그인하지 않음 |
  | 404 | `POST_NOT_FOUND` | 없거나 삭제된 과제 |
  | 404 | `COMMENT_NOT_FOUND` | `parentCommentId`의 댓글이 없음 |

### 댓글·답글 수정 (F-17)

`PATCH /api/comments/{commentId}` · 로그인 (작성자)

- 요청 Body
  ```json
  { "content": "수정한 댓글 내용" }
  ```
  `PUT`이 아닌 `PATCH`를 쓰는 이유: 댓글에서 바꿀 수 있는 것은 내용 하나뿐이고, 과제나 부모 댓글은 바꾸지 않는다.
- 성공 `200 OK`
  ```json
  { "commentId": 201, "updatedAt": "2026-11-18T03:10:00Z" }
  ```
- 실패

  | 상태 | 코드 | 상황 |
  | --- | --- | --- |
  | 400 | `INVALID_INPUT` | 내용 길이 위반 |
  | 401 | `UNAUTHORIZED` | 로그인하지 않음 |
  | 403 | `NOT_COMMENT_AUTHOR` | 작성자가 아님 |
  | 404 | `COMMENT_NOT_FOUND` | 없거나 삭제된 댓글 |

### 댓글·답글 삭제 (F-18)

`DELETE /api/comments/{commentId}` · 로그인 (작성자)

- 성공 `204 No Content` · 내용을 비우고 삭제 시각을 기록한다(소프트 삭제). 답글은 그대로 둔다.
- 실패

  | 상태 | 코드 | 상황 |
  | --- | --- | --- |
  | 401 | `UNAUTHORIZED` | 로그인하지 않음 |
  | 403 | `NOT_COMMENT_AUTHOR` | 작성자가 아님 |
  | 404 | `COMMENT_NOT_FOUND` | 없거나 이미 삭제된 댓글 |

---

## 5. 오류 코드 모음

| 상태 | 코드 | 상황 | 안내 문구 (`message`) |
| --- | --- | --- | --- |
| 400 | `INVALID_INPUT` | 입력값 길이·형식 위반 | `errors`의 필드별 사유를 따른다 |
| 400 | `INVALID_SEARCH_SCOPE` | 검색 범위 값이 잘못됨 | 검색 범위를 다시 선택해주세요. |
| 400 | `TOO_MANY_FILES` | 첨부 5개 초과 | 파일은 기존 파일을 포함해 최대 5개까지 첨부할 수 있습니다. |
| 400 | `UNSUPPORTED_FILE_TYPE` | 허용하지 않는 확장자 | 허용하지 않는 파일 형식이 포함되어 있습니다. |
| 400 | `FILE_TOO_LARGE` | 파일 하나 10MB 초과 | 파일 하나의 크기는 10 MB 이하여야 합니다. |
| 400 | `TOTAL_FILE_SIZE_EXCEEDED` | 첨부 합계 30MB 초과 | 첨부파일 전체 크기는 30 MB 이하여야 합니다. |
| 400 | `INVALID_ATTACHMENT` | 남길 첨부 ID가 이 과제의 것이 아님 | 첨부파일 정보를 다시 확인해주세요. |
| 400 | `INVALID_PARENT_COMMENT` | 답글을 달 수 없는 댓글 | 답글을 달 수 없는 댓글입니다. |
| 401 | `UNAUTHORIZED` | 로그인하지 않음 | 로그인이 필요합니다. |
| 401 | `LOGIN_FAILED` | 로그인 실패 | 이메일 또는 비밀번호를 확인해주세요. |
| 403 | `NOT_POST_AUTHOR` | 과제 작성자가 아님 | 본인이 작성한 과제만 수정·삭제할 수 있습니다. |
| 403 | `NOT_COMMENT_AUTHOR` | 댓글 작성자가 아님 | 본인이 작성한 댓글만 수정·삭제할 수 있습니다. |
| 404 | `POST_NOT_FOUND` | 과제 없음 | 과제를 찾을 수 없습니다. |
| 404 | `COMMENT_NOT_FOUND` | 댓글 없음 | 댓글을 찾을 수 없습니다. |
| 404 | `ATTACHMENT_NOT_FOUND` | 첨부파일 없음 | 첨부파일을 찾을 수 없습니다. |
| 409 | `DUPLICATE_EMAIL` | 이메일 중복 | 이미 사용 중인 이메일입니다. |
| 409 | `DUPLICATE_NICKNAME` | 닉네임 중복 | 이미 사용 중인 닉네임입니다. |
| 500 | `INTERNAL_ERROR` | 서버 내부 오류 | 잠시 후 다시 시도해주세요. |

**상태 코드를 고른 기준**

| 상태 | 언제 |
| --- | --- |
| 400 | 요청 자체가 규칙에 맞지 않음. 고쳐서 다시 보내면 성공할 수 있음 |
| 401 | **누구인지 모름** (로그인 안 함, 로그인 실패) |
| 403 | 누구인지는 알지만 **권한이 없음** (남의 글) |
| 404 | 대상이 없음 |
| 409 | 요청은 올바르지만 **이미 있는 데이터와 충돌** (중복 이메일·닉네임) |
