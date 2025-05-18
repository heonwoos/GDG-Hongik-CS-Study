# WEEKLY I LEARNED | Third Week | Linking

## 0. 링킹 이전 과정

- 소스 파일 `main.c`와 `sum.c`로부터 실행 파일 `prog`를 만드는 예제

```shell
gcc -o prog main.c sum.c
```

#### gcc 옵션 일부 설명

<table>
    <tr>
        <th>옵션</th>
        <th>설명</th>
    </tr>
    <tr>
        <td><code>-E</code></td>
        <td>전처리 단계만 수행</td>
    </tr>
    <tr>
        <td><code>-S</code></td>
        <td>컴파일 단계까지만 수행</td>
    </tr>
    <tr>
        <td><code>-c</code></td>
        <td>어셈블리 단계까지만 수행</td>
    </tr>
    <tr>
        <td><code>-o <em>file</em></code></td>
        <td>출력 파일의 이름을 <code><em>file</em></code>로 설정</td>
    </tr>
</table>

#### 빌드 과정

`.c` -> `.i` -> `.s` -> `.o` -> 실행파일(Windows에서 `.exe`)

1. `.c` -> `.i` (전처리; preprocessing): `#include` `#define`등을 처리, 전처리된 파일(`.i`)을 생성
2. `.i` -> `.s` (컴파일; compliation): 고급 언어를 어셈블리어로 번역, 어셈블리 파일(`.s`)을 생성
3. `.s` -> `.o` (어셈블리; assembly): 어셈블리어를 기계어로 번역한 relocatable object file(`.o`)을 생성
4. `.o` -> 실행파일 (링킹; linking): relocatable object file들을 하나로 묶어 실행파일(executable object file)을 생성

## 1. 링커(Linker)가 하는 일

### Symbol Resolution

- symbol의 정의와 참조를 매핑 (각 참조는 하나의 정의에 매핑되어야 함)

### Relocation

- 여러 object file의 주소를 하나의 기준 주소로 맞춤

### Execution File 생성

- Symbol Resolution과 Relocation을 거쳐 실행 파일 생성

## 2. Object File에 대하여

### obejct file의 종류

1. **relocatable object file**: 다른 object file과 결합
   - **shared object file**: load, run-time 때 메모리에 저장될 수 있음(? 뭔지 모르겠어요), 동적 링킹에 사용 가능(? 뭔지 모르겠어요)
2. **executable object file**: 메모리로 바로 복사되어 실행될 수 있음(실행 파일; Windows에서 `.exe`)

### relocatable object file의 구조

> Linux/Unix에서 object file의 구조는 Executable and Linkable Format(ELF)을 따른며, 그 구조는 다음과 같다.

<table>
    <tr>
        <th>영역</th>
        <th>설명</th>
    </tr>
    <tr>
        <td><strong>ELF header</strong></td>
        <td>메타 데이터</td>
    </tr>
    <tr>
        <td><strong>Sections</strong></td>
        <td>데이터 및 코드</td>
    </tr>
    <tr>
        <td><strong>Sections Header Table</strong></td>
        <td>각 section의 위치와 크기, section의 entry</td>
    </tr>
</table>

- Sections의 `.symtab`이라는 **symbol table**이 존재하는데, 링커가 이것을 이용해 **symbol resolution**을 한다.

## 3. Symbol Resolution에 대하여

### symbol의 종류

어떤 relocatable object 하나를 `m`이라고 할 때,

1. **global symbol**
   - `m`에 정의되어 있음, 다른 모듈에서 참조 가능
   - Non-static 함수, non-static 전역 변수
2. **externals**
   - 다른 모듈에 정의되어 있음, `m`에서 참조
3. **local symbols**
   - `m`에 정의되어 있음, `m`에서만 참조 가능
   - static 함수와 static 전역 변수

- (non-static인) **지역변수(local program variable)는 local symbol과 다른 것**으로, 런타임에 정해지므로 링커의 관심 대상이 아니다.

- **static 지역변수**는 `.bss` 및 `.data`에 저장이 된다.
  - 이때 지역변수이므로 이름이 같은 정적 변수가 있을 수 있는데, 컴파일러가 symbol table에 `var_name.1`, `var_name.2`처럼 변경해서 어셈블리 파일을 만든다.

### symbol table 구조

```cpp
typedef struct {
    int name;           //byte offset into str table that points to symbol name
    int value;          //symbol's addr: section offset, or VM addr
    int size;           //obj size in bytes
    char type:4,        //data, func, sec, or src file name (4 bits)
        binding:4;      //local or global (4 bits)
    char reserved;      //unused
    char section;       //section header index, ABS, UNDEF or COMMON
} Elf_symbol;
```

### symbol resolution 과정

- 링커는 symbol의 참조를 relocatable object file의 symbol table에 있는 symbol의 정의에 대응시킴

#### local symbol resolution

- 참조가 있는 모듈에 정의가 있으므로 쉽게 처리

#### global symbol resolution

- 컴파일러: 참조가 있는 모듈에 정의가 없음 -> 다른 모듈에 있다고 가정
- **다른 모듈에 같은 이름의 symbol 정의가 존재할 위험**
- 다른 모듈에서도 정의를 찾지 못하면 에러 출력 후 종료

### Linux 컴파일 규칙

#### strong/weak symbols

- **strong symbols**: 함수, 초기화된 전역변수
- **weak symbols**: 초기화되지 않은 전역변수

#### RULE

1. strong symbol은 오직 하나만 허용
2. strong symbol 하나와 weak symbol 여러 개가 있다면 strong symbol을 고른다.
3. weak symbol만 여러 개 있다면 아무거나 고른다.

### static library

- object file들을 하나의 static library로 묶을 수 있는데 다음과 같은 gcc 명령어로 링커에게 library를 사용하도록 할 수 있다.

```shell
gcc main.c /lib/libm.a /lib/libc.a
```

- 이때 링커는 `main.c`에서 **참조하는 object file만 복사하게 되어서** executable object file의 용량을 줄이고 실행시 메모리를 절약할 수 있다.
- 링커가 명령어에 입력된 파일들을 차례로 읽어가며 static library에서 필요한 objective files, unresolved symbols를 차례로 파악해서 선별하므로 참조하는 파일은 정의하는 파일보다 먼저 입력되어야 한다.
- `.a` 확장자는 archive를 뜻하고, Linux/Unix 계열에서 사용한다.
- Windows에서는 동일하진 않지만 비슷한 확장자로 `.dll` `.lib`이 있다.

## 더 읽어보면 좋을 자료

https://people.cs.pitt.edu/~xianeizhang/notes/Linking.html
