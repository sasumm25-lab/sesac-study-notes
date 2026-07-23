# Nop 코드 케이브 패치 실습 (Nop.exe 시리즈)

교안 p38~40. `Nop(empty).exe`, `Nop(quiz1 - template).exe`, `Nop(quiz2 - template).exe`는 **`.text` 시작(0x401000)부터 수백 바이트가 전부 `NOP`(0x90)으로 채워진** 빈 코드 공간(code cave)이다. 학생이 x32dbg에서 이 자리에 직접 어셈블리를 써 넣으며 명령어를 익히는 실습용 템플릿이다.

## 1. 템플릿 구조

`Nop(empty).exe`를 열면 EP(0x4012A7)의 CRT 스텁과 별개로, `0x401000`부터 이렇게 되어 있다:

```asm
0x401000  nop
0x401001  nop
0x401002  nop
...        (계속 nop)
```

여기에 코드를 써 넣고 `Ctrl+*`(Set New Origin Here)로 EIP를 `0x401000`에 맞춘 뒤 `F7`로 실행한다.

`Nop(quiz2 - template).exe`는 MessageBox 뼈대가 미리 들어있는 버전이다:

```asm
0x401000  nop
0x401001  push 0                    ; uType = MB_OK
0x401003  push 0x409278             ; lpCaption
0x401008  push 0x4092A0             ; lpText
0x40100D  push 0                    ; hWnd = NULL
0x40100F  call dword ptr [0x4080E4] ; USER32!MessageBoxW
0x401015  nop ...
0x40101A  mov eax, 1
0x40101F  mov ebx, 1
0x401024  nop ...                   ; ← 이 아래 빈 공간에 로직 작성
```

`call [0x4080E4]`가 `MessageBoxW`의 IAT 슬롯이다. 즉 "조건을 만족하면 MessageBox를 띄우는" 크랙미형 실습의 스캐폴드다.

## 2. 실습 A — 메모리 복사 (memcpy)

교안 명령어 파트(MOV/INC/DEC/JNZ)를 코드 케이브에서 직접 구현. `0x40C000`의 16바이트를 `0x40C070`으로 복사:

```asm
mov esi, 40C000          ; 원본(source)
mov edi, 40C070          ; 목적지(destination)
mov ecx, 10              ; 0x10 = 16바이트 (x32dbg에 10 입력 시 10진수 처리 주의 → 0x10 명시)

; --- 루프 시작 (이 줄 주소 기억) ---
mov al, byte ptr ds:[esi]   ; 1바이트 읽기
mov byte ptr ds:[edi], al   ; 1바이트 쓰기
inc esi                     ; 포인터 둘 다 전진
inc edi
dec ecx                     ; 카운터 감소 → ZF 세팅
jne 시작주소                 ; ECX≠0 이면 반복
```

**흔한 실수 3가지** (실제 실습 중 나온 버그):

1. `mov [edi], [esi]` — 메모리→메모리 직접 이동 불가. 반드시 `al` 등 레지스터 경유.
2. 목적지에 `[esi]`로 잘못 씀 — 원본에 되쓰게 되어 복사가 안 됨. `[edi]`로.
3. `mov ecx, 10`을 16진수로 착각 — x32dbg는 10진수 10(=0xA)로 받는다. `0x10` 명시.
4. 점프 목적지를 루프 시작(`mov al,...`)이 아니라 그 다음 줄로 잡으면, 값 읽기 줄을 건너뛰어 AL이 고정됨.

`REP MOVSB` 버전(문자열 명령어)도 있지만, 교안 목적은 "차례대로" 도는 걸 눈으로 보는 것이므로 반복문 버전이 학습에 좋다.

## 3. 실습 B — 문자열 비교 후 분기 (strcmp + MessageBox)

`0x40A150`(문자열1)과 `0x40A200`(문자열2)을 비교해 같으면 MessageBox, 다르면 종료:

```asm
mov esi, 40A150
mov edi, 40A200

; --- 비교 루프 ---
mov al, byte ptr ds:[esi]
mov bl, byte ptr ds:[edi]
cmp al, bl
jne NOT_EQUAL            ; 한 글자라도 다르면 실패
cmp al, 0               ; 널 종료면 (여기까지 다 같았음)
je  EQUAL               ; 성공
inc esi
inc edi
jmp 비교루프시작

EQUAL:
push 0
push 40A150             ; 캡션
push 40A150             ; 텍스트
push 0
call MessageBoxA

NOT_EQUAL:
push 0
call ExitProcess
```

**핵심 포인트**: `cmp al, bl` 로 먼저 같은지 본 뒤에만 `cmp al, 0`으로 종료를 검사한다. 두 글자가 같다고 확인된 상태에서 AL=0이면 BL도 0 — 즉 **양쪽이 같은 지점에서 동시에 끝났다**는 뜻이라, 널 검사는 한쪽만 해도 충분하다. `abc` vs `abcd`처럼 길이가 달라도 `cmp al,bl`에서 자동으로 걸러진다.

`cmp [esi],[edi]`도 메모리→메모리라 불가능해서 AL/BL을 거친다.

## 4. x32dbg 작업 팁

- 코드 입력: 명령어 줄에서 **스페이스** → 어셈블 창. 연속 입력됨. **"Fill with NOP's"** 체크로 남는 바이트 자동 패딩.
- 숫자는 기본 16진수 인식(`40C000`). 10진수 혼동 피하려면 `0x` 명시.
- 점프 목적지엔 라벨이 안 먹으므로, 대상 줄의 실제 주소(맨 왼쪽)를 확인해 `jne 40101C`처럼 직접 입력.
- `40C000` 영역이 쓰기 금지면 `Alt+M` → 섹션 우클릭 → Set Page Memory Rights로 권한 부여.
- 결과 확인: 덤프 창 `Ctrl+G`로 원본/목적지 주소를 각각 띄워 실행 전후 값 비교.
