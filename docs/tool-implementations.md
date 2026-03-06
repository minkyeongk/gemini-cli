# Gemini CLI 툴 구현 분석

> 소스 기준: `packages/core/src/tools/`

---

## 목차

1. [컨텍스트 전달 방식](#컨텍스트-전달-방식)
2. [툴 전체 목록 및 구현 방식](#툴-전체-목록-및-구현-방식)
3. [Merge Conflict 해결 시 실제 호출 흐름](#merge-conflict-해결-시-실제-호출-흐름)
4. [API 요청 구조 요약](#api-요청-구조-요약)

---

## 컨텍스트 전달 방식

### Turn 1 `contents`에 들어가는 것

```json
{
  "contents": [
    {
      "role": "user",
      "parts": [{ "text": "merge conflict 해결해줘" }]
    }
  ]
}
```

**conflict 내용은 `role: user`에 절대 들어가지 않는다.**

### conflict 내용이 등장하는 시점

| Turn | role    | 내용 |
|------|---------|------|
| 1    | user    | "merge conflict 해결해줘" |
| 1    | model   | tool_call: `run_shell_command("git diff --name-only --diff-filter=U")` |
| 2    | tool    | `"src/config.ts\nsrc/utils.ts"` |
| 2    | model   | tool_call: `read_file("src/config.ts", start_line=10, end_line=40)` |
| 3    | **tool** | `"<<<<<<< HEAD\n...\n=======\n...\n>>>>>>>"` ← **여기서 처음 등장** |
| 3    | model   | tool_call: `replace(old_string="<<<...", new_string="...")` |

- `start_line` / `end_line`으로 범위를 지정해도 결과는 동일하게 **`role: tool`** 로 반환됨
- `role: user`에는 파일 내용이 절대 들어가지 않음

### `replace` 툴에서 문맥 처리 방식

파일 전체를 별도로 보내주지 않는다. 대신 툴 description이 모델에게 강제한다:

> *"This tool requires providing **significant context** around the change to ensure precise targeting."*

모델은 `read_file`로 읽은 내용을 이미 히스토리로 가지고 있으므로, `old_string`에 충분한 주변 코드를 포함해서 위치를 특정한다.

---

## 툴 전체 목록 및 구현 방식

### 1. `read_file`

| 항목 | 내용 |
|------|------|
| 소스 | `tools/read-file.ts` |
| 구현 | **순수 Node.js 내장** |
| 핵심 API | `config.getFileSystemService().readTextFile()` → 내부적으로 `node:fs/promises` |
| 외부 CLI | 없음 |
| 특이사항 | `start_line` / `end_line`으로 부분 읽기 가능. 자동 truncate 적용 (기본 2000줄). 이미지·오디오·PDF도 처리 |

```typescript
// read-file.ts execute() 핵심
const result = await processSingleFileContent(
  this.resolvedPath,
  this.config.getTargetDir(),
  this.config.getFileSystemService(),
  this.params.start_line,
  this.params.end_line,
);
```

---

### 2. `write_file`

| 항목 | 내용 |
|------|------|
| 소스 | `tools/write-file.ts` |
| 구현 | **순수 Node.js 내장** |
| 핵심 API | `node:fs`, `node:fs/promises` |
| 외부 CLI | 없음 |
| 특이사항 | 파일 전체를 덮어씀. 부모 디렉토리 자동 생성. `diff` 라이브러리로 변경 diff 계산 후 llm에게 피드백 반환 |

---

### 3. `replace` (Edit 툴)

| 항목 | 내용 |
|------|------|
| 소스 | `tools/edit.ts` |
| 구현 | **순수 Node.js + npm 라이브러리** |
| 핵심 API | `node:fs/promises`, `diff` npm 패키지, `fast-levenshtein` npm 패키지 |
| 외부 CLI | 없음 |
| 특이사항 | 4단계 매칭 전략을 순서대로 시도. 실패 시 LLM self-correction 호출 |

**4단계 매칭 전략 (순서대로 시도)**

```
1. exact match      - 정확한 문자열 일치 (CRLF 정규화 포함)
2. flexible match   - 각 라인을 trim()해서 비교 (공백/들여쓰기 무시)
3. regex match      - 토큰 기반 정규식 변환 후 매칭
4. fuzzy match      - Levenshtein 거리 기반 (10% 이내 차이 허용)
```

**self-correction 흐름**

```
old_string 매칭 실패
  → FixLLMEditWithInstruction() 호출
    → instruction + old_string + new_string + 에러 + 파일 전체 내용을
      별도 LLM(baseLlmClient)에게 전송해서 수정된 search/replace 쌍 획득
  → 수정된 파라미터로 2차 calculateReplacement() 재시도
```

---

### 4. `grep_search`

| 항목 | 내용 |
|------|------|
| 소스 | `tools/grep.ts` |
| 구현 | **3단계 폴백 전략** |
| 외부 CLI | `git grep` (1순위), `grep` (2순위) |
| 최후 폴백 | `glob` npm + `node:fs/promises` (순수 JS) |

**3단계 폴백 전략 상세**

```
Strategy 1: git grep
  - 조건: git 레포지토리 + git 명령어 사용 가능
  - 명령어: git grep --untracked -n -E --ignore-case <pattern>
  - 장점: git이 관리하는 파일 기준, 빠름

Strategy 2: system grep
  - 조건: grep 명령어 사용 가능
  - 명령어: grep -r -n -H -E -I --exclude-dir=<...> <pattern> .
  - .gitignore의 제외 패턴을 --exclude-dir로 변환

Strategy 3: Pure JavaScript Fallback
  - 조건: 위 두 가지 모두 실패
  - 구현: globStream()으로 파일 목록 순회 + fs.readFile() + RegExp.test()
  - 순수 Node.js로 동작 보장
```

---

### 5. `grep_search` (ripgrep 버전)

| 항목 | 내용 |
|------|------|
| 소스 | `tools/ripGrep.ts` |
| 구현 | **ripgrep 바이너리 사용** |
| 외부 CLI | `rg` (ripgrep) |
| 바이너리 확보 | `@joshua.litt/get-ripgrep` npm 패키지로 자동 다운로드 |
| 저장 경로 | `Storage.getGlobalBinDir()` |
| 특이사항 | 시스템에 `rg`가 없으면 자동으로 다운로드해서 사용 |

```typescript
// ripGrep.ts - rg 바이너리 확보
import { downloadRipGrep } from '@joshua.litt/get-ripgrep';

async function ensureRipgrepAvailable(): Promise<string | null> {
  const existingPath = await resolveExistingRgPath();
  if (existingPath) return existingPath;
  // 없으면 자동 다운로드
  return downloadRipGrep(binDir);
}
```

---

### 6. `glob`

| 항목 | 내용 |
|------|------|
| 소스 | `tools/glob.ts` |
| 구현 | **npm 라이브러리** |
| 핵심 API | `glob` npm 패키지 (`glob()` 함수) |
| 외부 CLI | 없음 |
| 특이사항 | 결과를 수정 시간(mtime) 기준으로 정렬. 최근 24시간 내 파일 우선. `.gitignore` / `.geminiignore` 패턴 적용 |

```typescript
// glob.ts 핵심
import { glob, escape } from 'glob';  // npm 패키지

const entries = await glob(pattern, {
  cwd: searchDir,
  withFileTypes: true,
  stat: true,           // mtime 정보 수집
  nocase: !this.params.case_sensitive,
  ignore: this.config.getFileExclusions().getGlobExcludes(),
});
```

---

### 7. `list_directory`

| 항목 | 내용 |
|------|------|
| 소스 | `tools/ls.ts` |
| 구현 | **순수 Node.js 내장** |
| 핵심 API | `node:fs/promises` (`fs.readdir()`) |
| 외부 CLI | 없음 |
| 특이사항 | 1단계 목록만 반환 (재귀 없음). `ignore` 패턴으로 필터링 |

---

### 8. `run_shell_command`

| 항목 | 내용 |
|------|------|
| 소스 | `tools/shell.ts` |
| 구현 | **Node.js `child_process` + 시스템 쉘** |
| 핵심 API | `ShellExecutionService.execute()` → 내부적으로 `bash -c <command>` |
| 외부 CLI | 사용자가 입력한 임의의 명령어 |
| 특이사항 | `pgrep`으로 백그라운드 프로세스 PID 추적. inactivity timeout 지원. 바이너리 출력 감지 |

**내부 실행 래퍼 구조 (Linux/macOS)**

```bash
# 실제로 실행되는 명령어 (사용자 입력: git status)
{ git status; }; __code=$?; pgrep -g 0 >/tmp/shell_pgrep_xxx.tmp 2>&1; exit $__code;
```

백그라운드 프로세스 PID 수집을 위해 `pgrep`으로 래핑. Windows에서는 래핑 없이 직접 실행.

---

### 9. `google_web_search`

| 항목 | 내용 |
|------|------|
| 소스 | `tools/web-search.ts` |
| 구현 | **Gemini API Grounded Search** |
| 핵심 API | `@google/genai` SDK의 grounded search 기능 |
| 외부 CLI | 없음 |
| 특이사항 | 로컬에서 검색하지 않음. Gemini API 서버에서 Google 검색 수행 후 결과 반환 |

---

### 10. `web_fetch`

| 항목 | 내용 |
|------|------|
| 소스 | `tools/web-fetch.ts` |
| 구현 | **Node.js 내장 fetch + npm 라이브러리** |
| 핵심 API | `fetchWithTimeout()` (내부: `node:fetch`), `html-to-text` npm 패키지 |
| 외부 CLI | 없음 |
| 특이사항 | 최대 20개 URL 처리. 10MB 크기 제한. Private IP 차단. 분당 10회 rate limit. HTML → plain text 변환. LRU cache (15분) |

```typescript
// web-fetch.ts 핵심 흐름
const response = await fetchWithTimeout(url, { timeout: 10000 });
const html = await response.text();
const text = convert(html, { /* html-to-text 옵션 */ });
// 이후 Gemini API로 전달해서 prompt에 맞는 내용 추출
```

---

### 11. `read_many_files`

| 항목 | 내용 |
|------|------|
| 소스 | `tools/read-many-files.ts` |
| 구현 | **npm 라이브러리 + Node.js 내장** |
| 핵심 API | `glob` npm 패키지 + `node:fs/promises` |
| 외부 CLI | 없음 |
| 특이사항 | 여러 파일을 `--- {filePath} ---` 구분자로 이어 붙여 반환. 텍스트 외 이미지·오디오·PDF도 처리 가능 |

---

### 12. `save_memory`

| 항목 | 내용 |
|------|------|
| 소스 | `tools/memoryTool.ts` |
| 구현 | **순수 Node.js 내장** |
| 핵심 API | `node:fs/promises` |
| 외부 CLI | 없음 |
| 특이사항 | 글로벌 메모리 파일(`~/.gemini/GEMINI.md`)에 append. 모든 세션에서 시스템 프롬프트에 포함됨 |

---

### 13. `write_todos`

| 항목 | 내용 |
|------|------|
| 소스 | `tools/write-todos.ts` |
| 구현 | **인메모리 상태 관리** |
| 핵심 API | 내부 이벤트 버스 (MessageBus) |
| 외부 CLI | 없음 |
| 특이사항 | 파일 I/O 없음. UI에 todo 목록을 표시하기 위한 상태 업데이트만 수행 |

---

### 14. `get_internal_docs`

| 항목 | 내용 |
|------|------|
| 소스 | `tools/get-internal-docs.ts` |
| 구현 | **순수 Node.js 내장** |
| 핵심 API | `node:fs/promises` |
| 외부 CLI | 없음 |
| 특이사항 | 번들링된 내부 문서 파일 읽기. 경로 미입력 시 사용 가능한 문서 목록 반환 |

---

### 15. `ask_user`

| 항목 | 내용 |
|------|------|
| 소스 | `tools/ask-user.ts` |
| 구현 | **UI 인터렉션** |
| 핵심 API | MessageBus를 통한 프론트엔드 통신 |
| 외부 CLI | 없음 |
| 특이사항 | 파일 I/O 없음. 최대 4개 질문. choice/text/yesno 타입. multi-select 지원 |

---

### 16. `enter_plan_mode` / `exit_plan_mode`

| 항목 | 내용 |
|------|------|
| 소스 | `tools/enter-plan-mode.ts`, `tools/exit-plan-mode.ts` |
| 구현 | **상태 변경** |
| 핵심 API | 내부 모드 상태 플래그 |
| 외부 CLI | 없음 |
| 특이사항 | Plan Mode에서는 read-only 툴만 사용 가능. `exit_plan_mode` 시 플랜 파일 저장 |

---

### 17. `activate_skill`

| 항목 | 내용 |
|------|------|
| 소스 | `tools/activate-skill.ts` |
| 구현 | **스킬 시스템 연동** |
| 핵심 API | 내부 스킬 레지스트리 |
| 외부 CLI | 없음 |
| 특이사항 | 사용 가능한 스킬 이름 목록은 조건부로 툴 description에 포함됨 |

---

## 구현 방식 분류 요약

| 구현 방식 | 툴 |
|-----------|-----|
| **순수 Node.js 내장 fs** | `read_file`, `write_file`, `list_directory`, `save_memory`, `get_internal_docs` |
| **npm 라이브러리** | `glob` (`glob` 패키지), `replace` (`diff`, `fast-levenshtein`), `read_many_files` (`glob`), `web_fetch` (`html-to-text`) |
| **외부 CLI (폴백 전략)** | `grep_search` (git grep → system grep → JS), `grep_search_ripgrep` (rg 바이너리 자동 다운로드) |
| **시스템 쉘** | `run_shell_command` (`bash -c`) |
| **외부 API** | `google_web_search` (Gemini Grounded Search), `web_fetch` (HTTP fetch) |
| **UI/상태** | `write_todos`, `ask_user`, `enter_plan_mode`, `exit_plan_mode`, `activate_skill` |

---

## Merge Conflict 해결 시 실제 호출 흐름

```
[Turn 1]
  user   → "merge conflict 해결해줘"
  model  → run_shell_command("git diff --name-only --diff-filter=U")

[Turn 2]
  tool   → "src/config.ts\nsrc/utils.ts"
  model  → read_file("src/config.ts")
              └─ node:fs/promises로 파일 읽기
              └─ 결과를 role:tool로 반환

[Turn 3]
  tool   → "function getConfig() {\n<<<<<<< HEAD\n  return {host: 'localhost'};\n=======\n  return {host: 'localhost', timeout: 5000};\n>>>>>>> feature\n}"
  model  → replace(
              file_path="src/config.ts",
              instruction="Resolve merge conflict by keeping feature-branch version",
              old_string="function getConfig() {\n<<<<<<< HEAD\n  return {host: 'localhost'};\n=======\n  return {host: 'localhost', timeout: 5000};\n>>>>>>> feature\n}",
              new_string="function getConfig() {\n  return {host: 'localhost', timeout: 5000};\n}"
            )
              └─ calculateReplacement() → exact match 시도
              └─ 성공 시 node:fs/promises.writeFile()
              └─ diff 생성 후 결과에 포함

[Turn 4]
  tool   → "Successfully modified file: src/config.ts (1 replacements). Here is the updated code: ..."
  model  → (다음 conflict 파일 처리 또는 완료 메시지)
```

**핵심 포인트:**
- `old_string`에 conflict 마커 전체와 주변 함수 시그니처까지 포함 → 위치 특정
- conflict 내용은 `role: tool`로만 전달되며 `role: user`에는 절대 포함되지 않음
- `replace` 실패 시 → 자동으로 별도 LLM 호출해서 self-correction 시도

---

## API 요청 구조 요약

```json
{
  "model": "gemini-2.5-pro",
  "config": {
    "systemInstruction": "(시스템 프롬프트 - conflict 전용 지시 없음, 일반 git 가이드라인만)",
    "tools": [
      {
        "functionDeclarations": [
          { "name": "read_file", ... },
          { "name": "write_file", ... },
          { "name": "grep_search", ... },
          { "name": "glob", ... },
          { "name": "list_directory", ... },
          { "name": "run_shell_command", ... },
          { "name": "replace", ... },
          { "name": "google_web_search", ... },
          { "name": "web_fetch", ... },
          { "name": "read_many_files", ... },
          { "name": "save_memory", ... },
          { "name": "write_todos", ... },
          { "name": "get_internal_docs", ... },
          { "name": "ask_user", ... },
          { "name": "enter_plan_mode", ... }
        ]
      }
    ]
  },
  "contents": [
    { "role": "user", "parts": [{ "text": "merge conflict 해결해줘" }] }
  ]
}
```
