\[Twice the Bits, Twice the Trouble: Vulnerabilities Induced by Migrating to 64-Bit Platforms]

 ---
# 1. Integer-Related Vuln.

## 1-1. Basic consept

![[Pasted image 20230705080220.png]]
- 각 운영체제 별로 자료형에 할당된 크기가 다르다.
- Linux 64bit 에서 long 은 다른 운영체제들과 달리 8bytes 가 할당되어있다.

## 1-2. Vulnerabilites

#### 1-2-1. Integer truncation
![[Pasted image 20230705085609.png|400]]
32비트, 64비트 윈도우 및 리눅스 운영체제에서, int 형과 short 형은 각각 2bytes, 4bytes 가 된다.
int 형인 x 에 0xffffffff 값을 넣어주게 되면 short 형인 y 에는 truncation 으로 인해 rax 에서 ax, 즉 0x0000ffff 값이 들어가게 된다. 

### 1-2-2. integer overflow
![[Pasted image 20230705090703.png|300]]
![[Pasted image 20230705090906.png]]
32비트, 64비트 윈도우 및 리눅스 운영체제에서, int 자료형에 꽉 차게 넣었을 때 buffer 크기를 int 값에 특정 값을 더한 값으로 전달하게 되면 integer overflow 가 발생하여 buffer 의 크기가 작아지게 된다.
int 자료형인 x 에 -1 을 입력하면 0xffffffff, 즉 -1 값을 가지게 되고, 상수값을 임의로 작은 수로 설정하였을 때 malloc 시 CONST-1 만큼의 버퍼를 생성하게 된다. 해당 코드는 상수값을 2로 설정하였고, 디버깅 결과 malloc 의 size 가 1 이 되었다.   

### 1-3. Integer Signedness Issues

### 1-3-1. Sign Extension
![[Pasted image 20230705092001.png|500]]
malloc 함수의 인자는 movzx, 즉 부호가 없이 바이트형을 변환해서 전달한다(unsigned). 따라서 short 로 -1 을 전달받은 x 는 malloc 함수의 인자 속에서 unsigned short 로 변환되면서 부호가 없는 양수값 0xffff 가 0x0000ffff 로 바뀌게 된다.
하지만 memcpy 의 인자 중 size_t 는 movsx, 즉 부호가 있는 바이트형을 변환해서 전달한다(signed). 따라서 short 로 -1 을 전달받은 x 는 size_t 에서 부호가 있는 형으로 그대로 인식이 되어 0xffffffff 가 된다.

### 1-3-2. Sign comparison
![[Pasted image 20230705101852.png|500]]

int 자료형과 unsigned int 자료형이 비교될 때, unsigned 자료형은 C언어의 Implicit Type Conversion 규칙에 따라 모두 int 자료형으로 바뀌게 된다. 따라서 -1 은 그대로 -1이 되고, unsigned 자료형은 그대로 10이 되어 코드가 종료되지 않는다. 그 상황에서 -1 은 memcpy 에서 movsx 에 의해 0xffffffff 가 되고, 버퍼오버플로우가 발생한다. 
jl 은 부호가 있는 비트 간의 비교를 지원하고, 왼쪽 값이 작으면 SF 와 OF 가 다르게 설정되며 점프가 실행된다. 

---
# So...

위와 같은 취약점들은 순전히 개발자들에 의해 만들어진 타입오류로 인해 생기는 취약점이다.
하지만 후에 기술할 오류들은 32비트 운영체제에서 문제 없이 돌아가는 프로그램들이 64비트 운영체제에서 문제를 일으키는 경우로, 
이는 간접적으로 발견되며 개발자들이 예상하지 못한 부분일 가능성이 크다

이러한 취약점의 원인을 크게 2가지로 나누어 살펴볼 것인데,
1) 32비트 운영체제에서 64비트 운영체제 정수형 크기의 변화
2) 32비트 운영체제에서 64비트의 주소 크기가 커짐
으로 나누어보고자 한다.
---

# 2. 32-> 64 Vuln

## 2-1. Basic concept

![[Pasted image 20230705104211.png]]
- 32bit win,linux / 64bit win / 64bit linux 각 운영체제 별로 자료형 크기가 달라 각각 Integer-Related Vulnerability 다르게 존재
- 어떤 운영체제에서는 괜찮지만 다른 운영체제에서는 Vulnerability 발생 (ex. 주황색 Highlight)

## 2.2) Integer Width Change

32 비트 운영체제와 64비트 운영체제에서 크기가 다를 수 있다. 
-> 운영체제를 바꿔서 실행할 때 오류가 발생할 수 있다. 
심지어 signedness of Comparison 을 뒤집어 버퍼오버플로우 방지를 무력화할 수 있다.

32비트에선 문제가 되지 않는 size_t -> unsigned int 변환이 64비트 win, linux 에서는 모두 truncation 이 발생하게 된다.
32비트, 64비트 win 에서는 문제가 되지 않는 long -> int 변환이 64비트 linux 에서는 truncation 이 발생하게 된다. 
또한, size_t 와 같은 크기를 가지고 있는 pointer 을 다룰 때 두 가지 취약점 패턴이 발생하게 된다. 

### 2.1.1) Incorrect pointer differences

메모리의 크기는 시작 포인터와 끝 포인터 간의 차이를 통해 구해지는데, 그 크기를 담는 자료형은 ptrdiff_t 로 pointer 과 같은 크기인 8bytes 를 가진다. 하지만 이 값은 보통 int 자료형으로 담기게 되는데, 32비트 운영체제에서는 pointer 자료형과 int 자료형의 크기가 같기 떄문에 문제가 되지 않지만, 64비트 운영체제는 int 자료형이 4bytes, pointer 자료형이 8bytes 로 차이가 나고 이는 pointer 자료형에 담긴 값이 int 자료형으로 옮겨질 때 truncation 이 발생하면서 자료의 유실을 발생하게 한다. 

str 값에 0xffff ffff 보다 큰 값(0x1 0000 00ff) 를 넣었다고 했을 때, eol 은 결국 str 크기인 0x1 0000 00ff 가 되고, 이를 int 형에 전달하면 맨 끝 한 자리는 잘리게 된다(0x000000ff)
이 값을 버퍼의 크기와 비교하여 버퍼오버플로우를 방지하는 코드를 넣었더라도, 0xff 값이 버퍼의 크기보다 작을 경우 해당 코드를 우회하게 된다. 

이는 기본 라이브러리(fgets, fseek, snprintf) 에서 발생하는 취약점으로, 후에 다시 활용된다.

파일이 설사 4기가보다 작아서 pointer truncation 이 일어나지 않더라도 공격자는 악의적으로 메모리를 늘릴 수 있다. 하지만 ASLR 기법으로 막힌다.

![[Pasted image 20230705150327.png]]

버퍼 크기를 2만큼, 지정한 후 A를 int 최댓값 수 + 1 만큼 쓰면 string 길이는 0x1 0000 0001 이 되고, 이는 버퍼 크기보다 작기 때문에 버퍼오버플로우 방지 코드를 우회해서 strcpy 를 실행하게 된다. 

### 2.1.2) New Signedness Issues

#### 2.1.2.1) Basic Concept

![[Pasted image 20230705150639.png]]
- signed extension 은 signed 자료형으로 음수값을 표현하고 있는 데이터가 unsigned comparison 으로 바뀌면 큰 양수가 되는 것을 말한다
- 다만, unsigned 의 범위를 signed 값이 커버할 수 있다면 64비트 안에서는 *명시적으로 타입을 signed 로 지정하였을 경우* signed 로 바뀐다. 

#### 2.1.2.2) Signed Extension
1. signed 자료형이 unsigned 자료형으로 변환될 때 64비트는 크기가 확장되기 때문에 sign extension 일어남
	signed 자료형은 더 넓은 자료형으로 바뀔 때 signed extension 형태로 값 전달함(-1 을 -1 그대로 전달) -> 이걸 unsigned 가 받으면? 엄청 큰 양수가 됨 -> 잠재적 취약점

### 2.1.2.3) Signedness of comparison
2. signedness of comparison 바뀌어 버퍼 오버플로우 방어 무력화시킴
	보통 정수는 비교 전 unsigned, 즉 양수로 바뀌어야 함(32비트 운영체제에서는 타입을 명시적으로 변경해도 이를 무시함)
	64비트 운영체제에서는 명시적으로 규정한 타입을 인정함, 즉 명시적으로 signed 로 규정되었을 경우 unsigned 로 규정된 최댓값을 signed는 담을 수가 있게 되면 unsigned 자료형은 signed 자료형으로 바뀌게 된다.(64비트 리눅스 한정) -> 비교할 때 signed 자료형 으로 비교되므로 0xffff ffff 는 -1 로 인식, 버퍼오버플로우 방어기법을 우회하게 된다.  
	
	![[Pasted image 20230705153832.png|500]]
	len 에 -1 을 주게 되면 signed long 값이라 처음은 -1 이 들어오게 된다. 다만 BUF_SIZE 와 비교하는 과정에서 signed 형으로 비교되고, 버퍼오버플로우 방지 코드를 우회하게 된다. 그 후 memcpy 에서 받는 len 값은 unsigned int 로 casting 되어 unsigned int 최대값이 된다.  

## 2.3) Larger Address Space

### 2.3.1) 잠재적 Integer Overflow
32비트 운영체제와 달리 64비트 win, linux 운영체제에서, size_t 의 크기는 8, unsigned int 의 크기는 4 bytes 로, 보통 반복문에 쓰이는 변수는 int 형이기 때문에 size_t 값에 int 형의 최댓값보다 더 큰 수를 넣고 반복문의 조건을 0부터 size_t 보다 작거나 같을 때까지 i를 더한다고 하면 i 는 결국 integer overflow 가 발생하고 반복문은 무한 반복된다. 

### 2.3.2) 잠재적 signedness Issues
32비트에서는 memory 용량 문제로 int 양수 최대값을 초과하는 값을 넣으면 오류를 발생시키지만, 64비트에서는 음수로 변환됨(signed 최대치보다 크고 unsigned 최대치보다 작은 값을 넣으면 int 는 음수값으로 변환, )

### 2.3.3) 라이브러리 함수의 예외적 작동 

#### 2.3.3.1) fprintf, snprintf and vsnprintf
snprintf 는 str 값을 받을 때 int 형 최댓값보다 더 큰 길이의 문자열을 받으면 -1 을 리턴한다.
리턴한 값을 그대로 사용하게 될 경우 stack memory corruption 유발 가능

#### 2.2.3.2) file processing(fseek, ftell, fgetpos)
해당 함수들은 64비트 운영체제 상에서 4GB 넘는 파일들을 다루기로 설계되어있지 않다. 
ftello, ftello64 or \_\_ftelli64 함수들은 이러한 취약점을 막기 위한 대응책이지만 ftell 함수는 여전히 많이 쓰이고 있다. 
보통은 함수 실행 중 오류가 발생했을 때 리턴값을 -1 로 주고있지만, Long int 자료형의 최대값을 넘어가는 리턴값을 가지게 되면 0을 리턴한다. 
이 점을 이용해 파일의 사이즈를 0으로 리턴하여 버퍼를 아주 작은 크기로 설정하고, memory corruption 을 일으킨다. 


# 3) Imperial Study - Debian packages & Github

## 3-1) 얼마나 잘못된 타입 conversion 이 많이 쓰이고 있는지 
- 모두 직접적으로 취약점으로 이어지는 것은 아님
	- Debian stable(“Jessie”, release 8.2) packages - total 45000 warnings 

## 3-2) 잘못 쓰인 type conversion 들이 64비트 운영체제로 옮겨졌을 때 취약점이 발생하는 특징을 자동으로 검사하기
-  64비트 운영체제에서만 일어나는 New truncation 은 atol 이라는 함수를 통해서 발생함 (21% of atol call)
- .signedness issue 는 memcpy 에서 발생(10% of call)
- 잠재적 integer overflow 는 loop variable 을 size_t 로, loop body 에 (unsigned) int 로(9.5% of call)
- 잠재적 signedness issue 는 strlen 에서 발생 (15% of call)
- 라이브러리 함수의 예외적 작동은 snprintf 와 ftell(30% and 70% of call)

# 4) Case Study

## 4-1) New truncation - from 2007 in PHP(CVE-2007-1884) and linux kernal
- PHP - long -> int (php_sprintf_getnumber 함수가 리턴값을 long 으로 내는데 int 로 받음)
	- linux kernal - snprintf 로 메모리 인자 보낼 때 ptrdiff_t 값은 int 형보다 큼
- New signed issues - zlib and libarchive
	- zlib - sign extension
		- gzprintf : unsigned int 로 초기화된 값 int 형으로 바꿈 -> vsnprintf 의 인자 중 size_t 형으로 들어감 -> sign extension 발생 
	- libarchive - sign comparison
		- 64비트 운영체제로 넘어오면서 size_t 가 int64_t 로 변환될 때 두 형은 같은 크기를 가지므로 최대값 넘기면 둘다 sign 으로 바뀌면서 음수로 바뀜
- 잠재적 Integer overflow - GNU C Library. and C++ shared pointers
	- GNU C Library 는 문자열을 읽는데 카운터의 자료형을 int 로 구현, 문자열을 int 의 최댓값 넘어선 길이로 주면 계속 증가함, 버퍼오버플로우 발생
	- C++ shared pointers. pointer 의 reference 가 int 형으로 선언되어있어 internal counter 을 overflow 시킬 수 있고, int 의 최댓값을 가지는 포인터는 빈 공간을 가리키게 됨, use-after-free 공격으로 이어질 수 있음
- 잠재적 signedness Issues - libarchive 
	- Joilet identifiers 의 최대 허용 길이를 체크할 때 이름의 길이를 size_t 로 받아 int 로 explicitly cast 함 -> signedness 바꿀 정도의 길이 정도이면 되나 utf 16 및 multi bytes 사용하고 있으므로 최소한 원래 메모리 크기의 3배 정도가 필요할 것이다.


