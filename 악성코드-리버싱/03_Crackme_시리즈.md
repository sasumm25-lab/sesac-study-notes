# Crackme 시리즈 분석

교안 p58, p89, p117에서 다루는 크랙미 실습. 난이도 순으로 진행하며, 공통 전략은 **성공/실패 문자열을 찾아 그 분기(조건 점프)를 역추적**하는 것이다. `Crackme2`는 이미 별도 문서([abex_2nd_crackme_분석](abex_2nd_crackme_분석.md))에서 abex 수준으로 상세 분석했다.

## 공통 전략 — 크랙미 분석의 삼각형

1. **성공 블록** (예: "Congrats!") 과 **실패 블록** (예: "Beggar off!") 문자열을 찾는다.
2. 두 블록을 가르는 **조건 점프**를 역추적한다.
3. 여기서 두 갈래:
   - **키젠형**: 점프 앞 연산을 읽어 시리얼 생성 알고리즘을 복원 → 정답 산출
   - **패치형**: 조건 점프를 뒤집거나(`je`↔`jne`) NOP 처리 → 항상 성공

x32dbg: 우클릭 → **Search for → All referenced strings** → 실패 문자열 검색 → 더블클릭으로 참조 지점 점프 → 위로 스크롤하며 분기 확인.

## Crackme1.exe (교안 p58) — Delphi

**식별**: import에 `SOFTWARE\Borland\Delphi\RTL`, `EInvalidPointer` 등 → **Borland Delphi**로 컴파일된 크랙미. Delphi는 VCL 이벤트 핸들러(버튼 OnClick) 안에 검증 로직이 있다.

**핵심 문자열** (strings로 추출):

| 문자열 | 역할 |
|---|---|
| `Enter a Name!` | 이름 미입력 경고 |
| `Enter a Serial!` / `No Serial entered` | 시리얼 미입력 경고 |
| `Congrats! You cracked this CrackMe!` | **성공(good boy)** |
| `Beggar off!` | **실패(bad boy)** |
| `Registered User` | 등록 상태 표시 |

**분석 절차**

1. String references에서 `Beggar off!` 검색 → 더블클릭 → 실패 MessageBox를 호출하는 코드로 점프.
2. 그 위로 스크롤하면 성공(`Congrats!`)과 실패로 갈라지는 조건 점프(`jz`/`jnz` 또는 `test eax,eax` 뒤 분기)가 보인다. Delphi 시리얼 비교는 보통 `Name`을 가공한 문자열과 입력 `Serial`을 문자열 비교(`lstrcmp`류)하거나 정수 비교한다.
3. **패치형 풀이**: 실패로 보내는 점프에 BP → 실행 → 조건을 반대로 뒤집으면(예: `jnz`→`jz`, 또는 `nop`) 아무 시리얼이나 통과. 가장 빠른 크랙.
4. **키젠형 풀이**: 분기 직전 EAX/문자열을 만들어내는 연산을 따라가 알고리즘 복원. Crackme2(abex)에서 연습한 `Hex(Asc(c)+상수)` 패턴과 같은 방식으로 접근.

> Tip: Delphi 바이너리는 IDA/x32dbg의 문자열 참조가 잘 잡히지만, 문자열이 길이 프리픽스가 붙은 `AnsiString` 형태라 참조가 문자열 시작보다 약간 앞을 가리킬 수 있다. 분기 로직만 정확히 짚으면 된다.

## Crackme2.exe (교안 p89) — VB6 / abex 2nd crackme

Name/Serial 방식. 시리얼 = 이름 각 글자에 `Hex(Asc(글자)+100)`을 이어붙인 값. 상세는 **[abex_2nd_crackme_분석.md](abex_2nd_crackme_분석.md)** 참조. 핵심 비교는 `0x403329 __vbaVarTstEq`, 갈림 점프는 `0x403332 je`.

## Crackme3.zip (교안 p117)

압축 상태로 제공되는 상위 실습. Code Cave(교안에 `0x01004b00` 언급)를 활용한 후킹/패치 계열로 이어진다. 앞선 크랙미에서 익힌 문자열 추적 + 조건 점프 패치를 응용.

## Crackme4/5/6.exe — 추가 연습

같은 `Crack/` 폴더의 추가 크랙미. 파일 특성:

| 파일 | 크기 | 타입 | 접근 |
|---|---|---|---|
| Crackme4.exe | 8KB | PE32 GUI | 소형 — 어셈블리 직독 가능, 문자열/분기 바로 추적 |
| Crackme5.exe | 26KB | PE32 GUI | 문자열 참조 → 분기 패치 |
| Crackme6.exe | 26KB | PE32 GUI | 위와 동일 전략 |

작은 크랙미(특히 Crackme4)는 String references + 조건 점프만으로 빠르게 풀린다. 동일한 삼각형 전략을 반복 적용해 감을 익히는 용도다.

## 정리

크랙미는 결국 **문자열 → 분기 → (키젠 or 패치)** 세 단계의 반복이다. 패치는 빠르지만 알고리즘을 배우려면 키젠형으로 분기 앞 연산을 읽는 습관이 중요하다. abex(Crackme2)에서 그 연산 복원을 제대로 연습했으므로, 나머지 크랙미는 같은 틀을 적용하면 된다.
