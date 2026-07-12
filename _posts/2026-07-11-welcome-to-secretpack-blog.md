---
title: "블로그를 시작하며"
excerpt: "Secretpack 블로그의 첫 글 — 이 블로그에서 다룰 것들과 마크다운/코드 렌더링 확인용 샘플."
date: 2026-07-11
categories:
  - Notes
tags:
  - blog
  - jekyll
---

첫 글입니다. 이 블로그에서는 보안 리서치와 개발 과정의 기록을 남깁니다.
이 글은 홈 · 아카이브 · 카테고리 · 태그 · 검색 · 코드 하이라이팅이 제대로
동작하는지 확인하기 위한 샘플이기도 합니다.

## 다룰 주제

- 리버스 엔지니어링 / 바이너리 분석
- 취약점 연구 / 익스플로잇 개발
- 퍼징 / 프로그램 분석
- 펌웨어 · IoT · 모바일 보안

## 마크다운 렌더링 확인

인라인 코드는 `objdump -d ./target` 처럼 표시되고, 인용은 아래처럼 보입니다.

> "Given enough eyeballs, all bugs are shallow." — Linus's Law

리스트, **굵게**, *기울임*, 그리고 [링크](https://github.com/secretpack) 도 확인합니다.

## 코드 블록 확인

```python
def find_bug(binary):
    for func in binary.functions:
        if func.has_unchecked_memcpy():
            yield func.address
```

```c
// 간단한 스택 오버플로 예시
void vuln(char *input) {
    char buf[64];
    strcpy(buf, input); // no bounds check
}
```

## 표 확인

| 도구        | 용도               |
| ----------- | ------------------ |
| IDA Pro     | 정적 분석          |
| x64dbg      | 동적 분석          |
| AFL++       | 커버리지 퍼징      |

---

앞으로 하나씩 채워 나가겠습니다.
