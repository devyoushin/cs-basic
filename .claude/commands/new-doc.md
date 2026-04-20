---
description: CS 신규 문서를 스캐폴딩합니다. 사용법: /new-doc <카테고리> <주제>
---

아래 지침에 따라 CS 지식 베이스 문서를 새로 작성해 주세요.

## 입력 정보
- 사용자 입력: $ARGUMENTS
- 첫 번째 인자 = 카테고리 (01_os, 02_network, 03_database, 04_distributed-systems, 05_system-design, 06_security, 07_observability, 08_jvm, 09_container, deepdive)
- 두 번째 인자 이후 = 주제 (하이픈 구분)

## 파일 생성 규칙
1. 파일명: `{주제}.md` (소문자, 하이픈 구분)
2. 저장 위치: `{카테고리}/` 디렉토리
3. `rules/doc-writing.md`의 문서 작성 규칙 준수
4. `rules/cs-conventions.md`의 코드 작성 규칙 준수

## 문서 구조 (반드시 아래 섹션 모두 포함)

```markdown
# {주제명}

## 1. 개요
(핵심 개념 정의 + 왜 중요한지)

## 2. 핵심 설명
### 2.1 동작 원리
### 2.2 구체적 예시 및 코드
### 2.3 면접 포인트

## 3. 실무 포인트

## 4. 심화 내용 (Senior 레벨)

## 5. TIP
```

문서 작성 완료 후 `CLAUDE.md`의 카테고리별 파일 목록에 항목을 추가해 주세요.
