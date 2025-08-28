# AURIX TC275 임베디드 시스템 프로그래밍 교육자료

## 목차
1. [AURIX TC275 개요](#1-aurix-tc275-개요)
2. [개발 환경 구성](#2-개발-환경-구성)
3. [프로젝트 구조](#3-프로젝트-구조)
4. [실습 예제 분석](#4-실습-예제-분석)
5. [GPIO 포트 제어](#5-gpio-포트-제어)
6. [타이머 활용](#6-타이머-활용)
7. [실습 과제](#7-실습-과제)

---

## 1. AURIX TC275 개요

### 1.1 AURIX TC275란?
- **제조사**: Infineon Technologies
- **아키텍처**: TriCore™ 32-bit
- **코어**: 3개의 CPU 코어 (멀티코어 프로세서)
- **주요 용도**: 자동차 전자 제어 시스템, 산업용 제어 시스템
- **특징**: 높은 성능과 안전성을 요구하는 임베디드 애플리케이션에 최적화

### 1.2 주요 특징
- **멀티코어 아키텍처**: CPU0, CPU1, CPU2 독립적 동작
- **풍부한 I/O**: 다양한 GPIO 포트 제공
- **타이머**: 정밀한 타이밍 제어 가능
- **안전성**: Functional Safety 표준 (ISO 26262) 지원
- **개발 도구**: AURIX Development Studio 지원

---

## 2. 개발 환경 구성

### 2.1 필요한 소프트웨어
- **AURIX Development Studio (ADS)**: 통합 개발 환경
- **TASKING 컴파일러**: TriCore 아키텍처용 컴파일러
- **iFx 드라이버 라이브러리**: 하드웨어 추상화 레이어

### 2.2 프로젝트 설정
- **컴파일러**: TASKING TriCore
- **디버거**: TASKING 디버거
- **링커 스크립트**: `.lsl` 파일 사용

---

## 3. 프로젝트 구조

이 교육자료는 다음 3개의 실습 프로젝트를 포함합니다:

### 3.1 프로젝트 목록
```
Aurix-TC275/
├── TC275-LED/                 # LED 블링킹 예제
├── TC275-LED-Button/          # LED + 버튼 제어 예제
├── IO_Port_Test1/            # GPIO 포트 테스트 예제
└── README.md
```

### 3.2 공통 파일 구조
각 프로젝트는 다음과 같은 공통 구조를 가집니다:
```
프로젝트폴더/
├── Cpu0_Main.c              # 메인 CPU 코드
├── Cpu1_Main.c              # CPU1 코드 (보조)
├── Cpu2_Main.c              # CPU2 코드 (보조)
├── 기능별_소스.c             # 기능 구현 파일
├── 기능별_헤더.h             # 헤더 파일
├── Configurations/          # 설정 파일들
├── Libraries/              # 라이브러리 파일들
└── .launch 파일             # 디버그 설정
```

---

## 4. 실습 예제 분석

### 4.1 프로젝트 1: TC275-LED (LED 블링킹)

#### 4.1.1 목표
- LED 2개를 번갈아 깜빡이게 하기
- GPIO 출력 제어 기본 원리 이해

#### 4.1.2 주요 코드 분석

**LED 정의 (Blinky_LED.c)**
```c
#define LED         &MODULE_P00,5    // LED1: P0.5 포트
#define LED2        &MODULE_P00,6    // LED2: P0.6 포트
#define WAIT_TIME_H   250            // 대기 시간 (250ms)
```

**LED 초기화 함수**
```c
void initLED(void)
{
    // LED1을 출력 모드로 설정 (Push-Pull)
    IfxPort_setPinModeOutput(LED, IfxPort_OutputMode_pushPull, IfxPort_OutputIdx_general);
    IfxPort_setPinHigh(LED);  // LED1 끄기 (High = OFF)
    
    // LED2를 출력 모드로 설정 (Push-Pull)
    IfxPort_setPinModeOutput(LED2, IfxPort_OutputMode_pushPull, IfxPort_OutputIdx_general);
    IfxPort_setPinLow(LED2);   // LED2 켜기 (Low = ON)
}
```

**LED 제어 함수들**
```c
// 개별 LED 제어
void blinkLED1(void)
{
    boolean pinState = IfxPort_getPinState(LED);
    if(pinState == TRUE)
        IfxPort_setPinLow(LED);
    else
        IfxPort_setPinHigh(LED);
}

// 통합 LED 제어
void blinkLED_new(void)
{
    blinkLED1();
    blinkLED2();
    waitTime(IfxStm_getTicksFromMilliseconds(BSP_DEFAULT_TIMER, WAIT_TIME_H));
}
```

**메인 루프 (Cpu0_Main.c)**
```c
void core0_main(void)
{
    // 인터럽트 활성화
    IfxCpu_enableInterrupts();
    
    // 와치독 타이머 비활성화 (개발 목적)
    IfxScuWdt_disableCpuWatchdog(IfxScuWdt_getCpuWatchdogPassword());
    IfxScuWdt_disableSafetyWatchdog(IfxScuWdt_getSafetyWatchdogPassword());
    
    // CPU 동기화
    IfxCpu_emitEvent(&cpuSyncEvent);
    IfxCpu_waitEvent(&cpuSyncEvent, 1);
    
    initLED();  // LED 초기화

    while(1)
    {
        blinkLED_new();  // LED 블링킹 실행
        cnt++;           // 카운터 증가
    }
}
```

### 4.2 프로젝트 2: TC275-LED-Button (LED + 버튼 제어)

#### 4.2.1 목표
- 버튼 입력에 따라 LED 제어
- GPIO 입력 처리 방법 이해

#### 4.2.2 주요 코드 분석

**버튼 정의**
```c
#define BUTTON      &MODULE_P00,7    // 버튼: P0.7 포트
```

**버튼에 따른 LED 제어**
```c
void control_LED(void)
{
    // 버튼 상태 확인 (0 = 눌림, 1 = 안 눌림)
    if(IfxPort_getPinState(BUTTON) == 0)
    {
        IfxPort_setPinState(LED, IfxPort_State_low);   // LED 켜기
    }
    else
    {
        IfxPort_setPinState(LED, IfxPort_State_high);  // LED 끄기
    }
}
```

### 4.3 프로젝트 3: IO_Port_Test1 (GPIO 포트 테스트)

#### 4.3.1 목표
- 다중 GPIO 포트 제어
- 순차적 포트 출력 패턴 생성

#### 4.3.2 주요 코드 분석

**포트 정의**
```c
// 출력 포트들
#define TEST_I00    &MODULE_P00,1
#define TEST_I01    &MODULE_P00,3
#define TEST_I02    &MODULE_P00,5
#define TEST_I03    &MODULE_P00,7
#define TEST_I04    &MODULE_P00,9
#define TEST_I05    &MODULE_P00,11

// 입력 포트들
#define TEST_IN0    &MODULE_P33,0
#define TEST_IN1    &MODULE_P33,2
#define TEST_IN2    &MODULE_P33,4
#define TEST_IN3    &MODULE_P33,6
```

**포트 초기화**
```c
void initPort(void)
{
    // 출력 포트 설정
    IfxPort_setPinModeOutput(TEST_I00, IfxPort_OutputMode_pushPull, IfxPort_OutputIdx_general);
    // ... (다른 출력 포트들)
    
    // 입력 포트 설정 (풀업 저항 사용)
    IfxPort_setPinMode(TEST_IN0, IfxPort_Mode_inputPullUp);
    // ... (다른 입력 포트들)
    
    // 모든 출력 포트를 LOW로 초기화
    IfxPort_setPinLow(TEST_I00);
    // ... (다른 포트들)
}
```

**순차적 포트 테스트**
```c
void IOPort_test(void)
{
    // 순차적으로 포트를 HIGH로 설정
    IfxPort_setPinHigh(TEST_I00);
    waitTime(IfxStm_getTicksFromMilliseconds(BSP_DEFAULT_TIMER, WAIT_TIME*2));
    
    IfxPort_setPinHigh(TEST_I01);
    waitTime(IfxStm_getTicksFromMilliseconds(BSP_DEFAULT_TIMER, WAIT_TIME*2));
    
    // ... (계속)
    
    // 역순으로 포트를 LOW로 설정
    IfxPort_setPinLow(TEST_I05);
    waitTime(IfxStm_getTicksFromMilliseconds(BSP_DEFAULT_TIMER, WAIT_TIME*2));
    
    // ... (계속)
}
```

---

## 5. GPIO 포트 제어

### 5.1 GPIO 기본 개념
- **GPIO**: General Purpose Input/Output
- **포트**: 여러 핀들의 그룹 (예: P00, P33)
- **핀**: 개별 입출력 단자 (예: P0.5)

### 5.2 주요 GPIO 함수들

#### 5.2.1 출력 설정
```c
// 출력 모드로 설정
IfxPort_setPinModeOutput(포트, 모드, 인덱스);

// 핀 상태 설정
IfxPort_setPinHigh(포트);    // HIGH (1) 출력
IfxPort_setPinLow(포트);     // LOW (0) 출력
IfxPort_togglePin(포트);     // 상태 반전

// 상태 직접 설정
IfxPort_setPinState(포트, IfxPort_State_high);
IfxPort_setPinState(포트, IfxPort_State_low);
```

#### 5.2.2 입력 설정
```c
// 입력 모드 설정
IfxPort_setPinMode(포트, IfxPort_Mode_inputPullUp);    // 풀업 저항
IfxPort_setPinMode(포트, IfxPort_Mode_inputPullDown);  // 풀다운 저항
IfxPort_setPinMode(포트, IfxPort_Mode_inputNoPullDevice); // 저항 없음

// 입력 상태 읽기
boolean state = IfxPort_getPinState(포트);
```

### 5.3 출력 모드 종류
- **Push-Pull**: 강한 구동력, 일반적인 디지털 출력
- **Open-Drain**: 외부 풀업 저항 필요, I2C 통신 등에 사용

---

## 6. 타이머 활용

### 6.1 시간 지연 함수
```c
// 밀리초 단위 지연
waitTime(IfxStm_getTicksFromMilliseconds(BSP_DEFAULT_TIMER, 지연시간ms));

// 예제: 500ms 지연
waitTime(IfxStm_getTicksFromMilliseconds(BSP_DEFAULT_TIMER, 500));
```

### 6.2 타이머 원리
- **STM (System Timer Module)**: 시스템 타이머
- **Tick**: 시스템 클록의 기본 단위
- **변환**: 밀리초를 tick으로 변환하여 정확한 시간 제어

---

## 7. 실습 과제

### 7.1 기초 과제

#### 과제 1: 교대로 깜빡이는 LED
- LED 2개를 0.5초 간격으로 교대로 깜빡이게 하기
- 힌트: `blinkLED1()`과 `blinkLED2()` 함수를 순차적으로 호출

#### 과제 2: 버튼으로 LED 패턴 변경
- 버튼을 누르지 않으면: LED가 느리게 깜빡임 (1초 간격)
- 버튼을 누르면: LED가 빠르게 깜빡임 (0.1초 간격)

#### 과제 3: 러닝 라이트
- 6개의 LED를 순차적으로 켜서 흐르는 효과 만들기
- `IOPort_test()` 함수를 참조하여 구현

### 7.2 응용 과제

#### 과제 4: 버튼 카운터
- 버튼을 누를 때마다 LED 패턴이 변경되는 시스템
- 패턴 1: 모든 LED OFF
- 패턴 2: LED 1개만 ON
- 패턴 3: LED 2개 ON
- 패턴 4: 모든 LED ON
- 5번째 누르면 다시 패턴 1로 돌아가기

#### 과제 5: PWM 시뮬레이션
- LED의 밝기를 조절하는 PWM(Pulse Width Modulation) 효과
- 빠른 ON/OFF 반복으로 밝기 조절 시뮬레이션

### 7.3 고급 과제

#### 과제 6: 멀티태스킹 시뮬레이션
- CPU0: LED 제어
- CPU1: 버튼 입력 처리
- CPU2: 카운터 관리
- 각 CPU 간 데이터 공유 및 동기화

#### 과제 7: 상태 머신 구현
- 버튼 입력에 따라 시스템 상태가 변경되는 상태 머신
- 상태 1: 대기 모드 (LED 모두 OFF)
- 상태 2: 작업 모드 (LED 블링킹)
- 상태 3: 경고 모드 (LED 빠른 점멸)

---

## 8. 디버깅 및 문제 해결

### 8.1 일반적인 문제들

#### 8.1.1 LED가 켜지지 않을 때
- 포트 핀 번호 확인
- 초기화 함수 호출 여부 확인
- 하드웨어 연결 상태 확인

#### 8.1.2 버튼 입력이 인식되지 않을 때
- 풀업/풀다운 저항 설정 확인
- 버튼 하드웨어 상태 확인
- 입력 모드 설정 확인

### 8.2 디버깅 도구 활용
- **브레이크포인트**: 코드 실행 중단점 설정
- **변수 모니터**: 실시간 변수 값 확인
- **레지스터 뷰**: 하드웨어 레지스터 상태 확인

---

## 9. 참고 자료

### 9.1 공식 문서
- AURIX TC275 User Manual
- iFx Driver Documentation
- AURIX Development Studio User Guide

### 9.2 추가 학습 자료
- Infineon 공식 튜토리얼
- TriCore 아키텍처 가이드
- 임베디드 시스템 설계 원리

---

## 10. 마무리

이 교육자료를 통해 AURIX TC275의 기본적인 GPIO 제어와 타이머 사용법을 익혔습니다. 실습 과제를 통해 점진적으로 복잡한 시스템을 구현해보며 임베디드 시스템 개발 역량을 키워보세요.

**핵심 학습 내용**:
- GPIO 포트의 입출력 설정 및 제어
- 타이머를 이용한 정밀한 시간 제어
- 멀티코어 프로세서의 기본 구조 이해
- 실제 하드웨어 제어 프로그래밍 경험

**다음 단계**:
- ADC/DAC 활용
- 통신 인터페이스 (UART, SPI, I2C)
- 인터럽트 처리
- 실시간 운영체제 (RTOS) 적용

---
*이 교육자료는 AURIX TC275 개발 환경을 기반으로 작성되었습니다.*
