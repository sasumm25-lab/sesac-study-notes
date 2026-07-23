# abex' 2nd crackme 분석 (x32dbg)

VB6로 제작된 교육용 크랙미. 이름(Name)을 입력하면 그에 대응하는 시리얼(Serial)을 계산해 비교하는 전형적인 Name/Serial 방식이다. 이 문서는 x32dbg로 정답 시리얼을 찾아내는 과정을 정리한다.

## 대상 정보

| 항목 | 값 |
|---|---|
| 파일 | Crackme2.exe (PE32 GUI, Intel 386) |
| 런타임 | MSVBVM60.DLL (Visual Basic 6) |
| ImageBase | `0x400000` |
| 입력 | Name(Text1), Serial(Text2) |

VB6 네이티브 컴파일 바이너리라 문자열 비교·연산이 `__vbaVarTstEq`, `__vbaVarAdd`, `__vbaVarCat` 같은 VB 런타임 헬퍼로 처리된다. 이 함수 이름들이 분석의 이정표가 된다.

## 1. 접근 전략 — 문자열 따라가기

크랙미 분석의 정석은 **성공/실패 메시지 문자열을 찾아 그 참조 지점에서 갈림 분기(조건 점프)를 역추적**하는 것이다. 실패했을 때 뜨는 메시지를 기준으로 삼으면 실패 블록 → 갈림 점프 → 성공 블록 순으로 코드 구조가 드러난다.

이 크랙미에 박혀 있는 문자열:

| 문자열 | 주소 | 의미 |
|---|---|---|
| `Nope, this serial is wrong!` | `0x402440` | 실패 |
| `Yep, this key is right!` | `0x4023E4` | 성공 |
| `Congratulations` | `0x402418` | 성공 제목 |
| `Please enter at least 4 chars as name` | `0x40237C` | 이름 길이 부족 |

### x32dbg 조작 순서

1. CPU 창 **우클릭 → Search for → Current Module → String references**
2. 목록 검색칸에 `wrong` 입력 → `Nope, this serial is wrong!` 더블클릭 → 실패 메시지를 push하는 코드로 점프
3. 그 지점에서 **위로 스크롤**하며 실패 블록으로 들어오게 만든 조건 점프를 찾는다
4. 문자열 검색이 잘 안 걸리면 대안: 우클릭 → Search for → **Intermodular calls**에서 메시지박스 호출(`rtcMsgBox`)에 BP

## 2. 핵심 비교 지점

역추적하면 다음 비교문에 도달한다.

```asm
0x403321  lea edx, [ebp-0x44]      ; 계산된 정답 시리얼
0x403324  lea eax, [ebp-0x34]      ; 사용자가 입력한 시리얼
0x403327  push edx
0x403328  push eax
0x403329  call __vbaVarTstEq       ; 두 시리얼 비교
0x40332F  test ax, ax
0x403332  je  0x403408             ; ← 다르면 실패 블록으로 점프 (갈림길)
```

`__vbaVarTstEq`는 두 값이 같으면 `AX = 0xFFFF`, 다르면 `AX = 0`을 반환한다. `test ax,ax` 뒤의 `je 0x403408`은 "AX가 0(=다름)이면 실패로 가라"는 뜻이다. 같으면 점프하지 않고 아래 성공 블록(`Yep, this key is right!`)으로 흐른다.

- `[ebp-0x44]` = 크랙미가 계산한 **정답 시리얼**
- `[ebp-0x34]` = 내가 입력한 시리얼

`0x403332`의 `je`에 **F2로 BP**를 걸고 멈춘 뒤, push된 두 변수를 따라가면 정답 시리얼을 눈으로 확인할 수 있다. 우측 플래그의 **Z**로도 판정 결과를 읽을 수 있다(맞으면 Z=0으로 점프 안 함, 틀리면 Z=1로 실패 점프).

## 3. 시리얼 생성 알고리즘

`Check` 버튼 핸들러 안, 이름 글자를 순회하는 `For` 루프에서 시리얼이 만들어진다.

```asm
0x4031F7  call ord516          ; Asc(글자)   → 아스키 코드
0x403233  mov  [ebp-0xD4], 0x64 ; 상수 100
0x403243  call __vbaVarAdd     ; Asc + 100
0x40325B  call ord573          ; Hex$(합계) → 16진수 문자열
0x40327B  call __vbaVarCat     ; 결과 문자열 이어붙이기
0x40329A  call __vbaVarForNext ; 다음 글자 반복
```

정리하면 시리얼 생성 규칙은 다음과 같다.

> 이름의 **각 글자마다 `Hex(Asc(글자) + 100)`을 구해 전부 이어붙인 것**이 정답 시리얼이다.

루프 진입 전 `__vbaVarTstLt`로 이름 길이가 4자 미만이면 `Please enter at least 4 chars as name`을 띄우고 중단하는 길이 검사도 있다.

## 4. 키젠 로직 (검증용)

```python
def keygen(name: str) -> str:
    return "".join(format(ord(c) + 100, "X") for c in name)

# 예시
keygen("AAAB")  # -> 'A5A5A5A6'
```

| 글자 | Asc | +100 | Hex |
|---|---|---|---|
| A | 65 | 165 | A5 |
| A | 65 | 165 | A5 |
| A | 65 | 165 | A5 |
| B | 66 | 166 | A6 |

→ Name `AAAB` 의 시리얼 = **`A5A5A5A6`**

## 5. 정리 — 크랙미 분석의 삼각형

문자열 검색으로 **성공 블록 · 실패 블록 · 둘을 가르는 조건 점프** 세 지점을 찾는 것이 핵심이다. 이 크랙미에서는:

- 갈림 점프: `0x403332  je 0x403408`
- 성공 블록: `Yep, this key is right!` (fall-through)
- 실패 블록: `Nope, this serial is wrong!` (`0x403408~`)

알고리즘까지 파악하면 특정 이름에 의존하지 않는 키젠을 만들 수 있고, 알고리즘을 몰라도 비교 지점 BP만으로 정답 시리얼을 덤프해 확인할 수 있다.

---

*교육용 리버싱 실습 정리. 대상은 공개 배포된 학습용 크랙미다.*
