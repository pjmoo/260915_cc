---
name: commit-convention
description: Use whenever creating a git commit in this project. Enforces the commit message format "영어카테고리 : 한글 설명 메시지" (English category, colon, Korean description) instead of the default style.
---

# 커밋 메시지 컨벤션

이 프로젝트의 커밋 메시지는 아래 형식을 따른다.

```
<category> : <한글 설명>
```

- `category`는 영어 소문자로 작성한다.
- `category`와 설명 사이는 공백 + 콜론(`:`) + 공백(` : `)으로 구분한다.
- 설명은 한글로, 무엇을 왜 했는지 간결하게 작성한다 (변경 이유 중심, 존댓말/문장부호 없이 서술형 어미로 끝낸다).

## 사용 가능한 category

| category   | 용도 |
|------------|------|
| `feat`     | 새로운 기능 추가 |
| `fix`      | 버그 수정 |
| `refactor` | 동작 변화 없는 코드 구조 개선 |
| `docs`     | 문서(README, CLAUDE.md 등) 변경 |
| `test`     | 테스트 추가/수정 |
| `chore`    | 빌드 설정, 의존성, 스킬/설정 파일 등 잡무성 변경 |
| `style`    | 포매팅 등 코드 동작에 영향 없는 변경 |

## 예시

```
feat : Google OAuth2 로그인 플로우 추가
fix : Redis 연결 타임아웃 시 재시도 로직 누락 수정
chore : 커밋 메시지 컨벤션 스킬 추가
```

## 적용 방법

커밋을 생성할 때 `git commit -m` 메시지를 위 형식으로 작성한다. 첨부되는 attribution 라인(예: 🤖 Generated with Claude Code)은 형식 규칙과 무관하게 그대로 유지한다.
