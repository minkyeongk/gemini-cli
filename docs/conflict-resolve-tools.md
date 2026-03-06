# Gemini CLI — Conflict Resolve 관련 툴 구현 분석

> 소스 기준: `packages/core/src/tools/`, `packages/core/src/services/`, `packages/core/src/utils/`
>
> Conflict resolve 흐름에서 실제로 호출되는 툴만 다룬다.

---

## 컨텍스트 전달 방식 (핵심 전제)

```
Turn 1  role: user   → "merge conflict 해결해줘"   ← 유저 입력만
Turn 2  role: tool   → git 명령어 실행 결과
Turn 3  role: tool   → 파일 읽기 결과 (여기서 처음 conflict 내용 등장)
Turn 4  role: tool   → 파일 수정 결과
```

- conflict 파일 내용은 **항상 `role: tool`** 로만 전달됨
- `role: user`에 파일 내용이 사전 주입되는 경우는 없음
- `start_line` / `end_line`으로 부분 읽기를 해도 동일하게 `role: tool`로 반환됨

---

## 1. `run_shell_command`

### 역할 (conflict resolve 흐름에서)
conflict 파일 목록 탐지, git 상태 확인

```bash
git diff --name-only --diff-filter=U   # conflict 파일 목록
git status --porcelain                 # 상태 확인
```

### 소스
- `tools/shell.ts` — 툴 클래스 + 파라미터 처리
- `services/shellExecutionService.ts` — 실제 프로세스 실행

### 구현 방식: Node.js `child_process.spawn` + 시스템 쉘

```
run_shell_command("git status")
  │
  └─ ShellExecutionService.execute()
       │
       ├─ [PTY 사용 가능 시] @lydell/node-pty or node-pty로 pty 생성
       │    └─ pty.spawn(bash, ['-c', <guardedCommand>])
       │
       └─ [PTY 불가 시] child_process.spawn(bash, ['-c', <guardedCommand>])
```

**실제로 실행되는 명령어 래퍼 (Linux/macOS)**

```bash
# 사용자 입력: git status
# 실제 실행:
shopt -u promptvars nullglob extglob nocaseglob dotglob;
{ git status; };
__code=$?;
pgrep -g 0 >/tmp/shell_pgrep_<hex>.tmp 2>&1;
exit $__code;
```

- `shopt -u ...` : bash 옵션 비활성화 (출력 오염 방지)
- `pgrep -g 0` : 백그라운드 프로세스 PID 수집용. `is_background: true` 지원을 위해 항상 래핑
- Windows에서는 래퍼 없이 `powershell.exe -NoProfile -Command <cmd>` 로 직접 실행

**쉘 선택 로직** (`utils/shell-utils.ts:getShellConfiguration()`)

```typescript
// Windows
if (ComSpec ends with powershell.exe / pwsh.exe)
  → { executable: ComSpec, argsPrefix: ['-NoProfile', '-Command'] }
else
  → { executable: ComSpec, argsPrefix: ['/d', '/s', '/c'] }  // cmd.exe

// Linux/macOS
→ { executable: 'bash', argsPrefix: ['-c'], shell: 'bash' }
```

**출력 처리**
- 최대 버퍼: 16MB (`MAX_CHILD_PROCESS_BUFFER_SIZE`)
- 스크롤백: 300,000줄 (`SCROLLBACK_LIMIT`)
- 바이너리 출력 감지 시 스트림 중단
- inactivity timeout 지원 (설정 가능)
- 환경 변수 `GEMINI_CLI=1` 자동 주입

**LLM에게 반환되는 포맷**

```
Output: <stdout+stderr 합친 텍스트>
Exit Code: <0이 아닌 경우에만>
Signal: <시그널로 종료된 경우>
Background PIDs: <백그라운드 프로세스가 있는 경우>
```

---

## 2. `read_file`

### 역할 (conflict resolve 흐름에서)
conflict 마커(`<<<<<<<`, `=======`, `>>>>>>>`)가 포함된 파일 내용 읽기

### 소스
- `tools/read-file.ts` — 툴 클래스
- `utils/fileUtils.ts:processSingleFileContent()` — 실제 파일 처리

### 구현 방식: 순수 Node.js `fs` (외부 CLI 없음)

```
read_file("src/config.ts", start_line=10, end_line=40)
  │
  └─ processSingleFileContent()
       │
       ├─ fs.existsSync()          → 파일 존재 확인
       ├─ fs.promises.stat()       → 크기 확인 (기본 제한: 10MB)
       ├─ detectFileType()         → text / binary / svg / image / audio / pdf 판별
       │
       └─ [text 파일인 경우]
            ├─ readFileWithEncoding()   → BOM 감지 + UTF-8/UTF-16/UTF-32 자동 처리
            ├─ content.split('\n')      → 라인 배열로 분리
            ├─ lines.slice(start, end)  → start_line/end_line 적용
            └─ 2000줄 초과 시 자동 truncate
```

**라인 범위 처리 상세**

```typescript
// fileUtils.ts:processSingleFileContent()
const lines = content.split('\n');
const originalLineCount = lines.length;

if (startLine !== undefined || endLine !== undefined) {
  sliceStart = startLine ? startLine - 1 : 0;          // 1-based → 0-based
  sliceEnd = endLine
    ? Math.min(endLine, originalLineCount)
    : Math.min(sliceStart + DEFAULT_MAX_LINES_TEXT_FILE, originalLineCount);
} else {
  sliceEnd = Math.min(DEFAULT_MAX_LINES_TEXT_FILE, originalLineCount); // 기본 2000줄
}

const selectedLines = lines.slice(actualStart, sliceEnd);
```

**truncate 발생 시 LLM에게 전달되는 메시지**

```
IMPORTANT: The file content has been truncated.
Status: Showing lines 10-40 of 350 total lines.
Action: To read more, use start_line: 41 in a subsequent read_file call.

--- FILE CONTENT (truncated) ---
<실제 내용>
```

**제한값 (constants.ts)**

| 항목 | 기본값 |
|------|--------|
| 최대 라인 수 | 2,000줄 |
| 최대 라인 길이 | 2,000자 (이후 truncate) |
| 최대 파일 크기 | 10MB |

---

## 3. `replace` (Edit 툴)

### 역할 (conflict resolve 흐름에서)
`old_string`에 conflict 마커 블록을 포함한 원본을 넣고, `new_string`에 해결된 코드를 넣어 파일 수정

### 소스
- `tools/edit.ts` — 전체 구현
- `utils/llm-edit-fixer.ts` — self-correction LLM 호출

### 구현 방식: 순수 Node.js `fs` + npm 라이브러리 (외부 CLI 없음)

```
replace(file_path, instruction, old_string, new_string)
  │
  ├─ validateToolParamValues()
  │    ├─ omission placeholder 감지 ("...", "rest of methods" 등)
  │    └─ 경로 접근 권한 확인
  │
  ├─ calculateEdit()
  │    ├─ fs.readTextFile()           → 현재 파일 내용 읽기
  │    ├─ detectLineEnding()          → \r\n vs \n 감지
  │    ├─ content.replace(/\r\n/g, '\n')  → CRLF 정규화
  │    │
  │    └─ calculateReplacement()     → 4단계 매칭 (아래 상세)
  │         │
  │         └─ 실패 시 attemptSelfCorrection() → 별도 LLM 호출
  │
  └─ fs.writeTextFile()              → 매칭 성공 시 파일 쓰기
       └─ 원래 라인 엔딩(\r\n / \n) 복원
```

### 4단계 매칭 전략 (`calculateReplacement()`)

매 단계마다 성공하면 즉시 반환, 실패하면 다음 단계로 이동.

**Stage 1: Exact Match**

```typescript
// CRLF를 \n으로 정규화한 뒤 단순 문자열 포함 여부 확인
const occurrences = normalizedCode.split(normalizedSearch).length - 1;
if (occurrences > 0) → safeLiteralReplace() 호출
```

- 가장 빠름
- `$`가 포함된 문자열도 안전하게 처리 (`safeLiteralReplace` — `$` 이스케이프 방지)
- `allow_multiple: false`인데 2개 이상이면 에러

**Stage 2: Flexible Match**

```typescript
// 각 라인을 trim()해서 비교 (공백, 들여쓰기 차이 무시)
const searchLinesStripped = normalizedSearch.split('\n').map(l => l.trim());
const windowStripped = window.map(l => l.trim());
isMatch = windowStripped.every((line, i) => line === searchLinesStripped[i]);

// 매칭 성공 시: 실제 파일의 들여쓰기를 캡처해서 replace 내용에 적용
const indentation = firstLineInMatch.match(/^([ \t]*)/)[1];
applyIndentation(replaceLines, indentation);
```

- 들여쓰기가 다른 경우 구제
- 원본 들여쓰기를 보존해서 replace 내용에 자동 적용

**Stage 3: Regex Match**

```typescript
// old_string을 토큰으로 분리해서 유연한 정규식 생성
const delimiters = ['(', ')', ':', '[', ']', '{', '}', '>', '<', '='];
// 구분자 주변에 공백 추가 → 토큰으로 분리 → escapeRegex → \s* 로 연결

const pattern = `^([ \t]*)${escapedTokens.join('\\s*')}`;
const regex = new RegExp(pattern, 'gm');
```

- 토큰 사이 공백이 다른 경우 구제
- 들여쓰기 자동 보존

**Stage 4: Fuzzy Match** (`fast-levenshtein` npm 패키지)

```typescript
// 슬라이딩 윈도우로 Levenshtein 거리 계산
const d_raw  = levenshtein.get(windowText, searchBlock);
const d_norm = levenshtein.get(stripWhitespace(windowText), stripWhitespace(searchBlock));

// 공백 차이는 10% 가중치만 부여
const weightedDist = d_norm + (d_raw - d_norm) * 0.1;
const score = weightedDist / searchBlock.length;

if (score <= 0.1) // 10% 이내 차이면 매칭
```

- 성능 제한: `sourceLines.length * old_string.length² > 4×10⁸` 이면 스킵
- `old_string` 길이 10자 미만이면 false positive 방지로 스킵

### Self-Correction LLM 호출 (`attemptSelfCorrection()`)

4단계 모두 실패 시 별도 LLM을 호출해서 `old_string`을 자동 수정.

**호출 구조**

```
attemptSelfCorrection()
  │
  ├─ 파일이 디스크에서 변경됐는지 SHA256 해시로 확인
  │    → 변경됐으면 최신 내용으로 교체
  │
  └─ FixLLMEditWithInstruction() 호출
       │
       └─ baseLlmClient.generateJson()
            ├─ system: EDIT_SYS_PROMPT (전문 코드 편집 어시스턴트 역할)
            └─ user:   instruction + 실패한 old_string + new_string
                       + 에러 메시지 + 파일 전체 내용
```

**self-correction LLM 시스템 프롬프트 핵심 규칙**

```
1. search 문자열만 최소 수정 (replace는 건드리지 않음)
2. 실패 원인 명시 (공백, 들여쓰기, 라인 엔딩 등)
3. 파일에 이미 변경사항이 있으면 noChangesRequired: true 반환
4. 완전히 새로운 edit 생성 금지 — 기존 파라미터 수정만
```

**반환 스키마**

```json
{
  "search": "수정된 old_string",
  "replace": "원본 그대로",
  "noChangesRequired": false,
  "explanation": "실패 이유 및 수정 방법 설명"
}
```

타임아웃: 40초. 타임아웃 시 원래 에러를 그대로 반환.

**성공 시 LLM에게 반환되는 포맷**

```
Successfully modified file: src/config.ts (1 replacements).
Here is the updated code:
<변경 전후 diff snippet (context 5줄)>
```

diff snippet을 포함해서 반환하므로, 모델이 verify를 위해 추가 `read_file`을 호출하지 않아도 됨.

---

## 4. `grep_search` (conflict 탐지 보조)

### 역할 (conflict resolve 흐름에서)
conflict 마커(`<<<<<<<`)로 전체 레포지토리를 검색해서 conflict 파일 목록 파악

### 구현 방식: 3단계 폴백

```
grep_search(pattern="<<<<<<<")
  │
  ├─ Strategy 1: git grep (우선순위 최고)
  │    조건: git 레포 + git 명령어 사용 가능
  │    └─ git grep --untracked -n -E --ignore-case "<<<<<<<" [-- <include>]
  │
  ├─ Strategy 2: system grep (폴백 1)
  │    조건: grep 명령어 사용 가능
  │    └─ grep -r -n -H -E -I --exclude-dir=node_modules ... "<<<<<<<" .
  │
  └─ Strategy 3: Pure JavaScript (폴백 2, 항상 동작 보장)
       └─ globStream("**/*") → fs.readFile() → RegExp.test() per line
```

`run_shell_command("git diff --name-only --diff-filter=U")`로 충분히 conflict 파일을 찾을 수 있기 때문에, 실제 conflict resolve 흐름에서는 잘 사용되지 않지만 모델이 선택적으로 사용할 수 있음.

---

## 실제 호출 흐름 전체 (목업 데이터 포함)

```
[Turn 1]
  role: user
  → "merge conflict 해결해줘"

  role: model (tool_call)
  → run_shell_command
      command: "git diff --name-only --diff-filter=U"

[Turn 2]
  role: tool (result)
  → "src/config.ts\nsrc/utils.ts"
    Exit Code: (없음, 정상)

  role: model (tool_call)
  → read_file
      file_path: "src/config.ts"
      (start_line/end_line 생략 → 파일 전체, 최대 2000줄)

[Turn 3]
  role: tool (result)
  → "function getConfig() {
     <<<<<<< HEAD
       return { host: 'localhost' };
     =======
       return { host: 'localhost', timeout: 5000 };
     >>>>>>> feature-branch
     }"

  role: model (tool_call)
  → replace
      file_path: "src/config.ts"
      instruction: "Resolve merge conflict by keeping the feature-branch
                    version that adds timeout support"
      old_string: "function getConfig() {\n<<<<<<< HEAD\n  return { host: 'localhost' };\n=======\n  return { host: 'localhost', timeout: 5000 };\n>>>>>>> feature-branch\n}"
      new_string: "function getConfig() {\n  return { host: 'localhost', timeout: 5000 };\n}"

      내부 처리:
        1. fs.readFile("src/config.ts")
        2. calculateReplacement()
           → Stage 1 (exact match) 성공
        3. fs.writeFile("src/config.ts", resolvedContent)

[Turn 4]
  role: tool (result)
  → "Successfully modified file: src/config.ts (1 replacements).
     Here is the updated code:
     @@ -1,7 +1,3 @@
      function getConfig() {
     -<<<<<<< HEAD
     -  return { host: 'localhost' };
     -=======
     -  return { host: 'localhost', timeout: 5000 };
     ->>>>>>> feature-branch
     +  return { host: 'localhost', timeout: 5000 };
      }"

  role: model (tool_call)
  → read_file ("src/utils.ts")   ← 다음 conflict 파일 처리
  ...
```

---

## 툴 구현 방식 비교

| 툴 | 외부 CLI | npm 패키지 | Node.js 내장 |
|----|----------|------------|--------------|
| `run_shell_command` | `bash` / `powershell` | `@lydell/node-pty`, `strip-ansi` | `child_process.spawn` |
| `read_file` | 없음 | 없음 | `fs.promises` |
| `replace` | 없음 | `diff`, `fast-levenshtein` | `fs.promises` |
| `grep_search` | `git grep` → `grep` → JS | `glob` (폴백) | `fs.promises` (폴백) |

### `replace` 매칭 전략별 사용 케이스

| 전략 | 성공하는 케이스 | 실패 시 다음 단계 |
|------|----------------|------------------|
| exact | conflict 마커가 정확히 일치 | flexible로 |
| flexible | 들여쓰기만 다른 경우 | regex로 |
| regex | 공백/구분자 간격이 다른 경우 | fuzzy로 |
| fuzzy | 소량의 문자 차이 (10% 이내) | self-correction LLM |
| self-correction | 위 모두 실패 | 에러 반환 |

### `replace` self-correction이 트리거되는 conflict resolve 실패 사례

```
# 실패 사례 1: 모델이 old_string의 들여쓰기를 잘못 추정한 경우
old_string: "<<<<<<< HEAD\n  return {...}\n=======\n"  ← 2칸 들여쓰기로 추정
실제 파일:  "<<<<<<< HEAD\n    return {...}\n=======\n" ← 4칸

# exact → 실패, flexible → 성공 (trim 비교이므로)
# → self-correction까지 가지 않음

# 실패 사례 2: old_string에 conflict 마커 주변 컨텍스트가 부족한 경우
old_string: "<<<<<<< HEAD\n  return {...}"  ← 위아래 컨텍스트 없음
실제 파일:  "<<<<<<< HEAD\n  return {...}" ← 동일하지만 같은 패턴이 2개 존재

# exact → 실패(2개 found), flexible → 실패(2개), regex → 실패(2개)
# fuzzy → 실패, self-correction 호출
# self-correction: 파일 전체를 보고 주변 컨텍스트를 포함한 search 재생성
```
