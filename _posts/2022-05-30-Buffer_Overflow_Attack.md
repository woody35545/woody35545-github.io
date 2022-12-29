--- 
layout: post
title: "Buffer Overflow Exploit"
categories: "Blog"
tags: "Blog"

---
# Buffer Overflow Exploit

## **AT&T와 Intel의 assembly Syntax 차이**

---

GDB (GNU Debugger)

---

컴파일된 실행파일(바이너리 파일)을 Diassemble 해서 Assembly Instruction을 볼 수 있게 해주는 Debugger

### 

### X86의 전통적인 Register

---

- **General Purpose Register(범용 레지스터)** → AX, BX, CX, DX
    - GPR(General Purpose Register)는 일반적인 목적에 따라서 사용할 수 있는 레지스터를 의미
    
    - **GPR Usage**
        
        
        ---
        
        ![Untitled](Buffer%20Overflow%20Exploit%20ce83ce31bcd94d99bfeb231ac74f7137/Untitled.png)
        
        ---
        
        - Legacy structure: 16bits (Low Byte, High Byte)
            - 일반적으로 GPR은 Low Byte(8bits)와 High Byte(8bits) 두 부분으로 구분해서 16bits 크기를 가짐
            - 좌측 이미지에 일반적인 AX, BX, CX, DX가 사용되는 용도가 나와있으나, 범용 레지스터이기 때문에 꼭 저렇게 사용하지 않아도 됨. (용도에 맞게 사용하면 됨)
        
    
- **Pesudo General Purpose Register**
    - SP: Stack Pointer
    - BP: Base Pointer
        - X86 에서는 Base Pointer를 사용하지만 ARM에서는 BP는 사용하지 않음 ( 아키텍처마다 차이가 있음 )
    - SI: Source Index
    - DI: Destination Index
    
- **Special Purpose Register**
    - IP: 현재 실행되고 있는 Instruction의 주소를 담고 있는 Register
    - EFLAGS: 어떤 연산을 수행한 후의 결과(Overflow 등의 상태)를 저장하는 Register

### X86의 Operands

---

- **Register**: 레지스터에서 레지스터로 값을 전달하는 Operation
    
    → MOV EAX, EBX : EBX의 값을 EAX로 Copy. ( Operands는 MOV 지만, 실제 기능은 copy. 즉 MOV로 EAX에 값을 할당한 후에도 EBX 값이 남아있음) 
    
- **Immediate**: Constant 값을 Register에 넣어주는 Operation
- **Memory**: 메모리에서 값을 읽어와서 Register로 넣어주는 Operation, 다양한 Addressing Mode를 제공.
    - Adderssing Mode
        - **Direct**: MOV EAX, [10h]: 주소 10h에 있는 값을 가져와서 EAX에 넣음. 메모리 주소를 직접 지정해서 하는 방식
        - **InDirect**: MOV EAX, [EBX]: EBX 레지스터의 값을 주소로 생각해서 해당 주소로 간 후 값을 가져와 EAX에 넣어줌.
        - **Indexed**: MOV AL, [EBX + ECX * 4 + 10h]: 베이스에서 인덱스를 계산해서 주소를 지정해주는 방식. , 주소연산
        - **Pointer**가 가리키는 값의 크기를 나타내는 방식: MOV AL , byte ptr [BX] → 타입이 나타나면 사람이 어셈블리를 분석할 때 보기 편함.

** MOV Instruction의 자세한 예시는 시스템네트워크보안 강의 슬라이드 참고

- **MOV 이외의 Assembly에서의 Data Movement**
    - XCHG: src와 dst의 value를 swap 해줌
    - PUSH: src의 값을 stack에 넣어줌
    - POP: stack의 top값을 꺼내서 dst에 넣어줌
    - LEA: MOV랑 형식은 같지만 주소값 연산 방식에 차이가 있음.
        - 예를 들어 LEA EAX, [EBX + 10h] 에서 [EBX + 10h] 를 연산할 때, MOV는 EBX + 10h 주소에 있는 값을 EAX에 넣어주지만, LEA는 EBX+10h 연산한 값 자체를 EAX에 넣어줌
    
- **Stack Management Operands**
    - Push, Pop을 이용하여 스택을 관리 (MOV를 사용해도 무방함): word 단위로 실행됨 (x86에서 1 word = 16bits 이며, 워드의 크기는 아키텍처마다 다름)
    - Stack Pointer는 Freely manipulate 가능 (ESP를 임의의 값으로 이동시킬 수 있다)
    - 메모리에서 Stack은 High Address에서 Low Address로 grow (stack grows Downward)
    
- **Arithmetic and Logic 연산을 위한 Operations**
    - ADD, SUB, MUL, OR, XOR ...
    - MUL , DIV 는 Specific Register가 필요 (아키텍처에서 레지스터 사용 룰이 있는데, MUL과 DIV는 이에 따라 정해진 레지스터만 사용한다)
    
- **Conditional Statement에 관련된 Register**
    - Elevators: CMP 같은 condition을 결정하는 역할을 수행해서 boolean flags를 설정해줌.
    - Conditional Jump: Boolean Flag에 따라서 Jump를 수행. (조건에 따라서 IP가 이동시키는 역할을 수행)
    
    Conditional Statement를 수행하는 과정: Expression → Elevator → EFLAGS → Jump 
    
    - CMP와 같은 Expression 의 결과에 따라 Elevator가 Flag값을 설정하고 설정된 Flag 값에 따라서 Jump 수행
    

### Little Endian / Big Endian

---

Endian 은 컴퓨터 메모리에 연속된 대상을 배열하는 Byte Order를 의미한다.

![Untitled](./imgs/2022-05-30-Buffer_Overflow_Attac_imgs/Untitled.png)

- **MSB와 LSB**
    - MSB (Most Significant Bit): 데이터의 최상위 비트를 의미
    - LSB (Least Significant Bit): 데이터의 최하의 비트를 의미

### Little Endian 과 Big Endian의 메모리 할당 방식

e.g) int a = 0x12345678, 변수 a 가 메모리에 할당되는 예시로 설명

![IMG_0EB238410D01-1.jpeg](Buffer%20Overflow%20Exploit%20ce83ce31bcd94d99bfeb231ac74f7137/IMG_0EB238410D01-1.jpeg)

![IMG_0EB238410D01-1.jpeg](Buffer%20Overflow%20Exploit%20ce83ce31bcd94d99bfeb231ac74f7137/IMG_0EB238410D01-1.jpeg)

## Control Flow

---

**Control Flow:** Instruction이 program을 실행하는 Flow를 Control Flow 라고 한다. 즉 Control Flow는 프로그램이 실행되는 Flow

Assembly Code 보는 법:

Compile 중에 Assembly 코드를 확인 → gcc -S 옵션을 주면 .o 파일이 아니라 assembly 파일이 나옴.

컴파일된 프로그램의 Assembly 확인 (Disassemble): IdaPro 또는 objdump 등의 프로그램 사용하여 확인해 볼 수 있음

## Call Stack

---

- **Call Stack: 프로그램에서 현재 실행중인 sub routine에 대한 정보를 저장하는 stack 자료 구조**
- Call Stack 호출 규약 cdecl
    
    #  Caller A에서 Callee B를 호출한다고 가정했을 때 ,
    
    1. B를 실행하기 위한 Parameter(Function Arguments) 들이 stack에 올라간다.
    2. Return Address가 stack에 올라간다. ( 서브루틴이 끝나고 돌아가야할 주소값을 담고 있음 )
    3. A의 EBP를 Stack에 push
    4. .... → 확인하고 다시 적을 것
