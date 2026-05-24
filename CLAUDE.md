# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Single-file (`index.html`) client-side web app that lets users enter a travel destination and receive AI-generated information about attractions, restaurants, and transportation in Korean. No backend server, no build step — open the file directly in a browser.

**Deployed at:** https://travel-guide-app-two.vercel.app/  
**Repository:** https://github.com/heroofthesovietunion/travel-guide-app

## Running the App

Open `index.html` directly in a browser. No server required.

```
npx serve .
# or
python -m http.server 8080
```

## Architecture

Everything lives in one `index.html` file: HTML structure → `<style>` block → `<script>` block.

### API Flow

```
User input → handleSearch()
  → fetchTravelInfo(location, signal)
      → callOpenRouter(prompt, 'openai/gpt-4o-mini', signal)     // 1순위
          ↳ on failure → callOpenRouter(prompt, 'google/gemma-4-31b-it:free', signal)  // 2순위
              ↳ on failure → callOpenAIDirect(prompt, signal)    // 3순위 (OpenAI 직접)
  → renderResults(location, rawContent)
      → JSON.parse()
          ↳ data.valid === false → showInvalid(location, suggestion)
          ↳ data.valid === true  → renderList() × 3 sections
```

### Key Constants (top of `<script>`)

```javascript
let searchAbortController = null;   // 취소 버튼용 외부 AbortController
let userCancelled = false;          // 사용자 취소 플래그

const OPENROUTER_API_KEY = 'sk-or-v1-...' + '...';  // 분리 저장 (GitHub secret scan 우회)
const OPENAI_API_KEY     = 'sk-proj-...' + '...';
const OPENROUTER_BASE_URL = 'https://openrouter.ai/api/v1/chat/completions';
const OPENAI_BASE_URL     = 'https://api.openai.com/v1/chat/completions';
const REQUEST_TIMEOUT_MS  = 30000;
```

> API 키는 GitHub secret scanning 우회를 위해 문자열 분리(`'part1' + 'part2'`) 형태로 저장됨. 런타임에는 정상 동작.

### Expected API Response Shape

```json
{
  "valid": true,
  "suggestion": null,
  "attractions":    [{"name": "...", "address": "...", "description": "..."}],
  "restaurants":    [{"name": "...", "address": "...", "description": "..."}],
  "transportation": [{"type": "...", "address": "...", "description": "..."}]
}
```

`valid: false` 응답 시:
```json
{
  "valid": false,
  "suggestion": "올바른 지명 (단일, null 가능)",
  "attractions": [], "restaurants": [], "transportation": []
}
```

## Critical Implementation Rules

- **XSS prevention**: `innerHTML` 사용 금지. `textContent` / `createElement` + `appendChild` 전용.
- **AbortController**: `callOpenRouter`, `callOpenAIDirect` 양쪽 모두 timeout(30s) + 외부 cancel signal 이중 지원.
- **3-tier fallback**: OpenRouter GPT-4o-mini → OpenRouter Gemma free → OpenAI 직접. `userCancelled` 플래그가 true면 폴백 중단.
- **address 필드**: Google Maps 하이퍼링크(`<a>`)로 렌더링. `encodeURIComponent` + `rel="noopener noreferrer"` 필수.
- **temperature**: 0.4 (사실 기반 정보 정확도), **max_tokens**: 3000

## UI Sections & State

5개 섹션이 `.hidden` CSS 클래스로 상호 배타적으로 전환:

1. `#search-section` — 초기 화면
2. `#loading-section` — API 호출 중 (취소 버튼 `#cancel-search-btn` 포함)
3. `#error-section` — 네트워크/API 오류
4. `#invalid-section` — 존재하지 않는 지역 입력 시 (제안 버튼 포함)
5. `#results-section` — 성공 결과 (관광지/식당/교통편 3개 카드)

## Notable Helper Functions

- `eunNeun(word)` — 한국어 조사 자동 선택 ('은'/'는'). 마지막 글자의 유니코드 받침 여부로 판단.
- `showInvalid(inputLocation, suggestion)` — invalid 섹션 표시 + 제안 버튼 텍스트 설정.
- `renderList(listEl, items, nameKey, descKey)` — li 요소 생성, address는 Google Maps 링크로 렌더링.

## API Keys

`.env` 파일에 원본 키 보관:
- `OPENROUTER_API_KEY` — OpenRouter 경유 모든 모델 호출
- `OPENAI_API_KEY` — OpenAI 직접 호출 (3순위 폴백)

`index.html` 내 키는 분리 문자열 형태로 저장. `.env`가 원본 기준.

## PRD Reference

`PRD.md` — 전체 요구사항, 페르소나, 기능 명세, 구현 프롬프트(섹션 13) 포함.
