# 안티 디버깅 기법 (Anti-Debugging)

교안 p128~141. 악성코드가 **분석가의 디버거를 탐지·방해**하는 기법 모음. 실습자료 Part 2 `Anti-Debugging/` 폴더의 번호별 바이너리가 각 기법에 대응한다. 각 exe의 import를 확인해 어떤 방식인지 역으로 식별할 수 있다.

## 실습 바이너리 ↔ 기법 매핑

| 파일 | 기법 | 사용 API / 방식 (import 확인) |
|---|---|---|
| `1.IDP.exe` | IsDebuggerPresent | `kernel32!IsDebuggerPresent` |
| `2.PEB.exe` | PEB.BeingDebugged 직접 읽기 | **관련 API 없음** — PEB 구조 직접 접근 |
| `3.NGF.exe` | NtGlobalFlag | **관련 API 없음** — PEB의 NtGlobalFlag 직접 읽기 |
| `5.FW.exe` | FindWindow | `user32!FindWindowA` |
| `6.RD.exe` | Remote Debugger 체크 | `IsDebuggerPresent` (+ CheckRemoteDebuggerPresent 계열) |
| `7.DC.exe` | Dumb Code (쓰레기 코드) | `IsDebuggerPresent` 외 흐름 교란 |
| `8.FO.exe` | Fake Opcode | `IsDebuggerPresent` 외 가짜 명령어 |
| `9.Exercise.exe` / `9.Answer.exe` | 종합 실습 / 정답 | 여러 기법 결합 |

> `(fixed)` 접미사 파일(`1.IDP(fixed).exe`, `5.FW(fixed).exe`)은 안티디버깅을 우회(패치)한 정답본이다. 원본과 diff하면 어느 바이트를 고쳤는지 바로 보인다.

**핵심 관찰**: `2.PEB.exe`와 `3.NGF.exe`만 관련 API import가 없다. 이는 두 기법이 **API를 안 쓰고 PEB 구조체를 직접 읽기** 때문이다. API 후킹으로는 못 막고, 메모리를 직접 봐야 탐지되는 게 포인트다.

## 기법별 원리와 우회

### 1) IsDebuggerPresent (1.IDP)

가장 기본. `kernel32!IsDebuggerPresent()`는 내부적으로 PEB의 `BeingDebugged` 바이트를 반환한다.

```asm
call IsDebuggerPresent
test eax, eax
jnz  detected          ; 1이면 디버깅 중 → 종료/분기
```

**우회**: `call` 다음 `test eax,eax`의 결과 분기(`jnz`/`je`)를 뒤집거나, EAX를 강제로 0으로. 또는 PEB의 BeingDebugged를 0으로 패치.

### 2) PEB.BeingDebugged 직접 읽기 (2.PEB)

API를 우회하려는 분석가를 다시 우회. `fs:[30h]`으로 PEB 주소를 얻어 오프셋 `+2`(BeingDebugged)를 직접 읽는다.

```asm
; 2.PEB.exe 실제 코드 (검증됨)
0x401038  mov eax, dword ptr fs:[0x30]   ; PEB 주소
0x40103E  mov eax, dword ptr [eax+2]     ; PEB.BeingDebugged
test eax, eax
jnz detected
```

**우회**: 디버거에서 PEB의 `+2` 바이트를 런타임에 0으로 수정. x32dbg 명령창 `dump fs:[30]` 후 오프셋 확인.

### 3) NtGlobalFlag (3.NGF)

디버거로 프로세스를 띄우면 힙 관련 플래그가 세팅된다. PEB 오프셋 `+0x68`(x86)의 `NtGlobalFlag`가 `0x70`이면 디버깅 중.

```asm
; 3.NGF.exe 실제 코드 (검증됨)
0x401038  mov eax, dword ptr fs:[0x30]
0x40103E  mov eax, dword ptr [eax+0x68]  ; NtGlobalFlag
and eax, 0x70
jnz detected
```

**우회**: PEB `+0x68`을 0으로 패치.

### 5) FindWindow (5.FW)

디버거 **창 클래스명**을 찾아 탐지. `FindWindowA("OLLYDBG", 0)` 등이 NULL이 아니면 디버거 실행 중.

```asm
push 0
push "OLLYDBG"         ; 또는 "WinDbgFrameClass" 등
call FindWindowA
test eax, eax
jnz detected
```

**우회**: 분기 패치, 또는 디버거 창 클래스명 변경.

### 6) CheckRemoteDebuggerPresent (6.RD)

다른 프로세스가 자신을 디버깅 중인지 커널에 질의. 핸들을 넘겨 `pbDebuggerPresent`에 결과를 받는다.

### 7) Time Check / Dumb Code (7.DC)

`GetTickCount`/`RDTSC`로 코드 구간 실행 시간을 잰다. 디버거로 싱글스텝하면 시간이 비정상적으로 오래 걸리는 것을 이용. 사이에 의미 없는 **Dumb Code**를 끼워 분석 흐름을 흐트러뜨리기도 한다.

**우회**: 시간 측정 두 지점 사이를 빠르게 F9로 통과하거나, 차이 비교 분기를 패치.

### 8) Fake Opcode (8.FO)

정상 명령어 중간에 **가짜 바이트**를 넣어 디스어셈블러를 오정렬시킨다. 선형 디스어셈블(linear sweep)은 엉뚱하게 해석하지만, 실제 실행 흐름(`jmp`로 건너뜀)은 정상. IDA의 흐름 분석이 깨지도록 유도.

**우회**: 실제 실행 경로를 따라 수동으로 코드 재정렬, 가짜 바이트를 NOP/데이터로 표시.

### 9) Exercise / Answer

위 기법들을 결합한 종합 문제. `9.Exercise.exe`를 분석해 안티디버깅을 모두 우회하고, `9.Answer.exe`로 정답 흐름을 대조한다.

## 공통 우회 워크플로우 (x32dbg)

1. **BP on API**: `IsDebuggerPresent`, `FindWindowA`, `GetTickCount`에 BP(Symbols 탭 또는 Intermodular calls). 호출 직후 반환값 분기를 확인.
2. **분기 뒤집기**: `test eax,eax` 다음 `jnz detected`에서, 플래그(Z)를 수동 토글하거나 명령어를 패치.
3. **PEB 직접 수정**: API를 안 쓰는 2·3번은 PEB의 `+2`(BeingDebugged), `+0x68`(NtGlobalFlag)을 메모리에서 0으로.
4. **플러그인 활용**: ScyllaHide 같은 안티-안티디버그 플러그인이 위 항목을 자동 처리(교안은 수동 실습 권장).
5. **(fixed) 대조**: 정답본과 원본을 바이트 diff해서 패치 위치를 역으로 학습.

## 정리

안티디버깅은 "탐지 → 분석가가 우회 → 더 낮은 레이어에서 재탐지"의 반복이다. `IsDebuggerPresent`(API) → PEB 직접 읽기 → NtGlobalFlag 순으로 **점점 저수준으로 내려가는 흐름**을 이해하는 게 핵심이다. 결국 모든 탐지는 PEB의 몇 개 필드로 수렴하므로, PEB 구조를 알면 대부분 우회된다.
