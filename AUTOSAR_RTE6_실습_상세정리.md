# AUTOSAR RTE6 실습 (SeatSwitch / SeatHeatingControl) 단계별 상세 정리

이 문서는 경북대 ARC 연구실 Exercise-RTE6 실습 절차 전체를, 각 단계가 "왜" 필요하고 "무엇을" 하는 것인지 개념부터 풀어서 정리한 문서다. 실습 대상은 시트 스위치(SeatSwitch)와 가변저항(Pot)으로 승객 감지 여부와 원하는 열선 강도를 읽어들여, 그 값을 SeatHeatingControl에게 전달하고, SeatHeatingControl이 LED(디지털 출력)와 PWM(열선 강도 출력)을 제어하는 시스템이다.

---

## 검토 의견 (붙여넣어주신 절차에 대해)

전체적으로 절차의 논리와 순서는 AUTOSAR 방법론에 정확히 맞게 구성되어 있다. 다만 아래 6가지는 실제 진행 중에 한 번 더 확인해보는 게 좋다.

1. **`IoHwAb_If_AnalnDir`라는 표기**: 다이어그램에서는 `IoHwAb_If_AnaInDir`(Analog Input Direct)로 표기되어 있다. 실습 노트의 `AnalnDir`는 오탈자로 보이며, 툴에서 실제 인터페이스 이름을 선택할 때 정확한 이름인지 확인이 필요하다.

2. **R_SeatSwitch의 Com Spec 표기 불일치**: `PassengerDetected`는 "Enable **Provided** Com Specs"로, `HeatStrength`는 "Enable **Required** Com Specs"로 적혀 있다. R_SeatSwitch는 Receiver(R-Port)이므로 두 Data Element 모두 원칙적으로는 **Required** Com Spec을 활성화하는 것이 맞다. (다만 툴에 따라 체크박스 명칭이 역할과 무관하게 통일되어 있을 수도 있으니, 실제 화면에서 어떤 명칭으로 뜨는지 한 번 확인해보는 게 안전하다.)

3. **ECU Extract 단계에서 "Performs Flattening" 체크 여부**: 이전 실습(RTE3)에서는 이 단계에 `Performs Flattening` 체크가 명시되어 있었는데, 이번 절차에는 언급이 없다. 실제 화면에 해당 옵션이 보인다면 체크하고 진행하는 것이 안전하다.

4. **`IoHwAbAnalogInputDirectLogical_Test2`라는 이름**: 실제 프로젝트 화면에서는 `Test1`이 존재했다(`Test2`가 아니라). 실습 문서와 실제 프로젝트의 기존 항목 이름이 다를 수 있으니, 이름을 바꾸기 전에 실제로 존재하는 항목명을 눈으로 확인하고 그 항목을 바꾸는 것이 안전하다.

5. **`SWC_SeatHeatingControl` 생성 문구가 2번 적혀 있음**: 실제 오류는 아니고 메모가 중복 작성된 것으로 보인다(같은 동작을 두 번 설명).

6. **I/O Mapping에서 `P_IoHwAb…_Pot`, `P_IoHwAb…_LED_Blue`가 안 보이는 이슈**: 이 문서 어딘가에서 겪었던 것처럼, IoHwAb Logical 설정(Adc Group Ref, Pwm Hw Channel 체인)이 끝까지 유효하게 연결되어 있는지, 그리고 그 설정 이후에 ECU Configuration을 다시 생성(Generate)했는지를 I/O Mapping 진행 전에 먼저 확인하는 것을 권장한다.

이 6가지를 제외하면 나머지 절차는 논리적으로 일관되고 순서도 올바르다. 아래부터는 각 단계가 정확히 무엇을 하는 단계인지 자세히 설명한다.

---

## 0. Project 생성

```
File → Import → General → Existing Projects into Workspace → Next
Browse.. → Base Project 선택 → Copy projects into workspace 체크 → Finish
```

Base Project(다른 사람 또는 학교에서 제공한 원본 프로젝트)를 내 워크스페이스로 가져오는 단계다. `Copy projects into workspace`를 체크하는 이유는, 이 옵션을 켜면 원본 프로젝트 파일은 그대로 보존한 채 워크스페이스 안에 "복사본"을 만들어서 그 복사본 위에서 작업하게 되기 때문이다. 체크하지 않으면 원본 위치의 파일을 직접 수정하게 되어, 실습 중 실수로 원본을 망가뜨릴 위험이 있다.

---

## 1. VFB Level (Virtual Functional Bus Level)

VFB Level은 AUTOSAR 소프트웨어 개발의 최상위, 가장 추상적인 단계다. 이 단계에서는 "어떤 컴포넌트가 있고, 어떤 포트로 무엇을 주고받는가"만 정의하며, 이 컴포넌트가 나중에 어떤 물리적 제어기(ECU)에 배치될지는 전혀 고려하지 않는다.

### ① ARXML(AUTOSAR File) 생성

```
Configuration → System → Swcd_App 우클릭 → New → AUTOSAR File
Package name & File name : App_Rte → Finish
```

ARXML은 "AUTOSAR XML"의 줄임말로, 앞으로 만들 SWC, Interface, Port, Runnable, Composition 같은 모든 모델 정보가 저장될 XML 파일 형식이다. `App_Rte`라는 이름의 새 ARXML 파일을 만드는 건, 이 실습에서 설계할 모든 요소를 담을 "빈 그릇"을 하나 준비하는 것이다. 아직 이 안에는 아무 내용도 없고, 앞으로의 모든 단계가 이 파일(정확히는 이 파일이 대표하는 ARPackage `App_Rte`) 안에 요소를 하나씩 채워나가는 과정이다.

### ② SWC 생성

```
App_Rte[ARPackage] 우클릭 → New → Application Sw Component Type
SWC_SeatSwitch 생성, SWC_SeatHeatingControl 생성
Supports Multiple Instantiation : false (둘 다)
```

`Application Sw Component Type`은 AUTOSAR가 정의하는 여러 SWC 종류 중 하나로, 실제 애플리케이션 로직(하드웨어에 직접 의존하지 않는 로직)을 담는 컴포넌트다. 여기서는 아직 포트도, 인터페이스도, Runnable도 없는 "빈 껍데기" 두 개(SWC_SeatSwitch, SWC_SeatHeatingControl)만 만드는 단계다.

`Supports Multiple Instantiation`을 `false`로 두는 것은, 이 SWC 타입을 여러 개의 인스턴스로 복제해서 쓰지 않겠다는 뜻이다. 예를 들어 차량에 좌석이 4개 있어서 SeatSwitch를 4벌 찍어내야 하는 상황이라면 이 값을 `true`로 둔다. 지금 실습은 좌석 1개짜리 단순 예제이므로 `false`가 맞다.

### ③ Sender-Receiver Interface 생성

```
App_Rte[ARPackage] 우클릭 → New → Sender Receiver Interface
Short Name : If_SeatSwitch, Is Service : false
Data Elements → PassengerDetected (boolean), HeatStrength (uint16)
```

Interface는 "포트가 무엇을 주고받을지"에 대한 규격(계약서)이다. 아직 어떤 컴포넌트에도 속하지 않는, 독립적으로 존재하는 통신 규격을 먼저 정의하는 단계다.

`Sender Receiver Interface`는 한쪽(Sender)은 보내기만, 한쪽(Receiver)은 받기만 하는 단방향 통신 규격이다. `Is Service : false`는 이 규격이 BSW(기본 소프트웨어) 서비스용이 아니라, 순수하게 애플리케이션 컴포넌트끼리 데이터를 주고받기 위한 것임을 뜻한다.

Data Element는 이 인터페이스를 통해 실제로 오가는 "값"의 목록이다. 여기서는 두 개를 정의했다.
- `PassengerDetected` (boolean): 승객이 감지되었는지 여부
- `HeatStrength` (uint16): 원하는 열선 강도 값

즉 `If_SeatSwitch`라는 규격은 "이 통신선을 통해 boolean 값 하나(PassengerDetected)와 16비트 정수 값 하나(HeatStrength)가 함께 오간다"고 정의한 것이다.

### ④ SWC_SeatSwitch 포트 설정

```
Ports + → Sender Receiver Interface → Sender → If_SeatSwitch
  Short Name : P_SeatSwitch
  PassengerDetected → Enable Provided Com Specs 체크
  HeatStrength → Enable Provided Com Specs 체크

Ports + → Client Server Interface → Client → IoHwAb_If_DigDir
  Short Name : R_SW06
  Operations → ReadDirect → Enable Provided Com Specs 체크

Ports + → Client Server Interface → Client → IoHwAb_If_AnaInDir
  Short Name : R_Pot
  Operations → ReadDirect → Enable Provided Com Specs 체크
```

SWC_SeatSwitch는 3개의 포트를 갖는다.

**P_SeatSwitch**는 방금 만든 `If_SeatSwitch` 인터페이스를 Sender 역할로 사용하는 P-Port(Provide Port)다. 즉 SWC_SeatSwitch는 이 포트를 통해 PassengerDetected와 HeatStrength 값을 "제공(보냄)"한다. `Enable Provided Com Specs`를 체크하는 이유는, 이 Data Element 각각에 대해 실제로 통신 관련 세부 속성(어떻게 보낼지)을 활성화해서 RTE가 그에 맞는 `Rte_Write_...` API를 생성할 수 있게 하기 위해서다.

**R_SW06**과 **R_Pot**은 둘 다 Client-Server 인터페이스의 Client 역할을 맡는 R-Port다. 이 포트들은 하드웨어에 직접 접근하지 않고, IoHwAb(I/O Hardware Abstraction, 하드웨어 입출력을 표준화된 방식으로 접근하게 해주는 서비스 레이어)가 제공하는 표준 인터페이스(`IoHwAb_If_DigDir`, `IoHwAb_If_AnaInDir`)를 통해 하드웨어 값을 읽어온다. R_SW06은 디지털 스위치 값을, R_Pot은 아날로그 가변저항 값을 읽기 위한 포트이며, 둘 다 `ReadDirect`라는 오퍼레이션을 사용하도록 설정했다.

이 세 포트를 만드는 시점에 이미 "어떤 인터페이스를 쓸지"까지 함께 지정하는 것이 중요하다. 포트는 인터페이스 없이는 존재할 수 없기 때문에, 포트 생성과 인터페이스 연결은 하나의 통합된 작업이다.

### ⑤ SWC_SeatHeatingControl 포트 설정

```
Ports + → Sender Receiver Interface → Receiver → If_SeatSwitch
  Short Name : R_SeatSwitch
  PassengerDetected → Com Specs 체크 + Init Value(Numerical Value Specification) 설정
  HeatStrength → Com Specs 체크 + Init Value(Numerical Value Specification) 설정

Ports + → Client Server Interface → Client → IoHwAb_If_DigDir
  Short Name : R_LED_Red
  Operations → WriteDirect → Enable Provided Com Specs 체크

Ports + → Client Server Interface → Client → IoHwAb_If_Pwm
  Short Name : R_LED_Blue
  Operations → SetDutyCycle → Enable Provided Com Specs 체크
```

**R_SeatSwitch**는 `If_SeatSwitch`를 Receiver 역할로 쓰는 R-Port다. SWC_SeatSwitch의 P_SeatSwitch와 짝을 이루어, PassengerDetected와 HeatStrength 값을 "받는" 쪽이다.

여기서 두 Data Element 모두에 `Init Value`(초기값)를 설정하는 이유가 중요하다. Sender 쪽(P_SeatSwitch)은 초기값을 굳이 설정하지 않아도 되는데, 보내는 쪽은 처음부터 값을 만들어서 보내기만 하면 되기 때문이다. 반면 Receiver 쪽은 "아직 한 번도 값을 못 받은 상태"가 있을 수 있으므로, 그 공백 상태에 쓸 기본값이 필요하다.

**R_LED_Red**는 디지털 출력용 Client 포트로, `IoHwAb_If_DigDir`의 `WriteDirect` 오퍼레이션을 사용해 LED(또는 열선)를 켜고 끄는 디지털 신호를 하드웨어에 쓴다.

**R_LED_Blue**는 PWM 출력용 Client 포트로, `IoHwAb_If_Pwm`의 `SetDutyCycle` 오퍼레이션을 사용해 듀티 사이클(on/off 비율)을 조절함으로써 밝기 또는 열선 강도를 아날로그적으로 제어한다. WriteDirect(단순 on/off)와 달리 SetDutyCycle은 "얼마나 세게" 켤지까지 조절할 수 있는 방식이다.

### ⑥ CSWC 생성 및 연결

```
App_Rte[ARPackage] 우클릭 → New → Composition Sw Component Type
Short Name : CSWC_SeatHeatingSystem
Components and Ports + → SWC_SeatSwitch, SWC_SeatHeatingControl 추가
Automatic Connection → R_SeatSwitch 선택 → + → P_SeatSwitch 체크 → OK
```

CSWC(Composition Software Component)는 "다른 컴포넌트를 포함하는 컴포넌트"다. 지금까지 따로 만들어온 SWC_SeatSwitch와 SWC_SeatHeatingControl을 하나의 상자(CSWC_SeatHeatingSystem) 안에 넣고, 그 안에서 서로 연결(Assembly Connector)해주는 단계다. 이 CSWC 자체도 하나의 Component-type이기 때문에, 나중에 더 상위의 Composition(RootComposition)에 통째로 들어갈 수 있다.

`Automatic Connection`에서 `R_SeatSwitch`를 선택하고 `P_SeatSwitch`를 매칭시킨 것은, 두 포트가 같은 인터페이스(`If_SeatSwitch`)를 쓰는 짝임을 확인하고 Assembly Connector로 실제로 이어주는 작업이다. (`R_LED_Red`, `R_LED_Blue`, `R_SW06`, `R_Pot`은 이 CSWC 내부에서 연결될 상대가 없다 — 이들은 나중에 ECU 레벨에서 IoHwAb Service SWC의 포트와 연결된다.)

---

## 2. RTE Level

VFB Level에서 만든 구조(컴포넌트, 포트, 인터페이스, 연결)는 아직 아무 "동작"도 하지 않는다. RTE Level은 각 컴포넌트에 실제로 "무엇을 할지"를 부여하는 단계다.

### Runnable 설정: SWC_SeatSwitch

```
Runnables + → Short Name : RE_SeatSwitch, Symbol : SeatSwitch_func
Can Be Invoked Concurrently : false

RTE Event → Timing Event, Period : 100msec

Operation/Mode/Trigger Access → SSCP → R_Pot.ReadDirect & R_SW06.ReadDirect

Data/Parameter Access → DSP(Data Sent Points) → P_SeatSwitch.HeatStrength & P_SeatSwitch.PassengerDetected
```

Runnable은 컴포넌트가 실제로 실행할 코드 단위(함수)다. `RE_SeatSwitch`라는 이름의 Runnable을 만들고, 이 함수가 나중에 C 코드에서 `SeatSwitch_func`라는 실제 함수 이름으로 구현될 것이라고 미리 선언한 것이다. `Can Be Invoked Concurrently : false`는 이 Runnable이 동시에 여러 번 중복 실행되지 않도록(한 번에 하나씩만 실행되도록) 하는 설정이다.

**RTE Event (Timing Event, 100ms)**: 이 Runnable이 "언제" 실행될지를 정한다. 스위치와 가변저항 값은 누가 대신 알려주는 게 아니라 직접 주기적으로 확인해야 하므로, 100ms마다 스스로 실행되는 방식(Timing Event)을 쓴다.

**Operation/Mode/Trigger Access (SSCP)**: 이 Runnable 안에서 호출할 Client-Server 오퍼레이션을 미리 등록한다. SSCP(Synchronous Server Call Point)는 "호출하면 그 자리에서 즉시 결과를 받는" 블로킹 방식의 함수 호출이다. `R_Pot.ReadDirect`와 `R_SW06.ReadDirect` 둘 다 등록했으므로, 이 함수는 나중에 `Rte_Call_R_Pot_ReadDirect(...)`와 `Rte_Call_R_SW06_ReadDirect(...)`라는 두 개의 RTE API를 쓸 수 있게 된다.

**Data/Parameter Access (DSP)**: 이 Runnable 안에서 내보낼 데이터를 등록한다. Data Sent Point는 `Rte_Write_P_SeatSwitch_HeatStrength(...)`, `Rte_Write_P_SeatSwitch_PassengerDetected(...)`라는 두 API를 쓸 수 있게 해준다.

정리하면, RE_SeatSwitch가 실행될 때 벌어질 일은: 100ms마다 실행된다 → `Rte_Call_...ReadDirect`로 스위치와 가변저항 값을 동기적으로 읽어온다 → `Rte_Write_...`로 그 두 값을 SWC_SeatHeatingControl 쪽으로 내보낸다.

### Runnable 설정: SWC_SeatHeatingControl

```
Runnables + → Short Name : RE_SeatHeatingControl, Symbol : SeatHeatingControl_func
Can Be Invoked Concurrently : false

RTE Event → Data Received Event → R_SeatSwitch.HeatStrength & R_SeatSwitch.PassengerDetected

Operation/Mode/Trigger Access → SSCP → R_LED_Blue.SetDutyCycle & R_LED_Red.WriteDirect

Data/Parameter Access → DRPBA(Data Received Points By Arguments) → R_SeatSwitch.HeatStrength & R_SeatSwitch.PassengerDetected
```

**RTE Event (Data Received Event)**: SWC_SeatSwitch와 반대로, 이 Runnable은 스스로 확인할 게 없고 데이터가 도착했을 때만 반응하면 된다. 그래서 Timing Event가 아니라 "값이 도착한 순간" 정확히 트리거되는 Data Received Event를 쓴다. 이렇게 하면 불필요한 반복 실행 없이, 값이 바뀌는 즉시 반응할 수 있다.

**Operation/Mode/Trigger Access (SSCP)**: `R_LED_Blue.SetDutyCycle`과 `R_LED_Red.WriteDirect`를 등록해서, 받은 값을 바탕으로 하드웨어에 출력을 쓸 수 있게 한다.

**Data/Parameter Access (DRPBA)**: `Data Received Points By Arguments`는 이 Runnable이 이미 Data Received Event로 트리거되었기 때문에, RTE가 그 방금 도착한 값을 함수 인자(argument)로 곧바로 넘겨주는 방식이다. 별도로 값을 다시 조회하는 호출을 할 필요가 없다.

정리하면, RE_SeatHeatingControl은 새 값이 도착하는 순간 실행되어, 그 값을 인자로 받고 → `Rte_Call_...WriteDirect`, `Rte_Call_...SetDutyCycle`로 LED/열선을 실제로 제어한다.

---

## 3. C Coding

```
Static Code → Reference Code → src 우클릭 → New → File
SWC_SeatSwitch.c, SWC_SeatHeatingControl.c 생성 후 코드 작성
```

지금까지는 "무엇을 할지"에 대한 껍데기(Runnable, Event, Access Point)만 만들었을 뿐, 실제 알고리즘은 아직 없다. 이 단계에서 각 Runnable의 Symbol 이름(`SeatSwitch_func`, `SeatHeatingControl_func`)과 정확히 일치하는 함수를 C 파일에 작성한다. 함수 안에서는 앞서 등록해둔 Access Point들에 대응하는 RTE API(`Rte_Call_...`, `Rte_Write_...` 등)를 호출하는 코드를 작성한다. RTE가 생성해주는 `Rte_<Swc>.h` 헤더를 include해야 이 API들을 쓸 수 있다.

---

## 4. ECU Mapping

```
Configuration → System → Composition → RootComposition.arxml → CSWC_RootComposition 더블클릭
Components and Ports + → CSWC_SeatHeatingSystem 선택
```

VFB Level에서 만든 컴포넌트 구조는 "제어기를 고려하지 않은" 설계다. 이 단계에서 처음으로 "이 컴포넌트 묶음을 실제로 어느 제어기에 넣을지"를 결정한다. 개별 SWC를 하나씩 넣지 않고 CSWC(Composition) 단위로 RootComposition에 넣는 이유는, CSWC가 이미 두 SWC와 그 연결 관계를 캡슐화하고 있으므로, 그 캡슐 전체를 한 번에 배치하는 것이 더 명확하고 안전하기 때문이다.

---

## 5. ECU Extract

```
Auto-Wiz → System Configuration & ECU Extract → ECU Software Components Mapping
OK → SWC/Connector 적용 확인 → Apply
```

시스템 전체 구성 정보(System Configuration) 중에서, 지금 대상으로 하는 이 특정 제어기에 필요한 정보만 추출(Extract)하는 단계다. 여러 ECU가 있는 큰 시스템이라면 각 ECU마다 필요한 부분만 뽑아내야 하는데, 이 실습은 ECU가 하나뿐이지만 절차상 반드시 거쳐야 하는 단계다. 이 과정을 거쳐야 이후 ECU Configuration 단계에서 이 컴포넌트들에 대한 세부 설정(Task 매핑, I/O 매핑 등)을 진행할 수 있는 상태가 된다.

---

## 6. ECU Configuration

### ① OS Configuration

```
Auto-Wiz → ECU Configuration & Code Generation OS 1.3.0 클릭.
Task + → OsTask_SWC_SeatSwitch_100ms (Activation 1, Priority 117)
Task + → OsTask_SWC_SeatHeatingControl (Activation 1, Priority 118)

Alarm + → OsAlarm_SWC_SeatSwitch_100ms
  Counter Ref → OsCounter_Main
  Action → Activate Task → OsTask_SWC_SeatSwitch_100ms

Application → OsApplication0
  App Alarm Ref에 만든 Alarm 추가
  App Task Ref에 만든 Task 2개 추가
```

RTE가 생성한 Runnable 호출 코드는 결국 OS(운영체제)의 Task 안에서 실행되어야 한다. 이 단계는 그 실행 그릇(Task)과, 그 그릇을 정해진 시간마다 깨우는 알람(Alarm)을 준비하는 단계다.

`OsTask_SWC_SeatSwitch_100ms`는 SWC_SeatSwitch의 주기적인 Runnable(RE_SeatSwitch)이 실행될 Task이고, `OsTask_SWC_SeatHeatingControl`은 데이터 수신 시 반응하는 Runnable(RE_SeatHeatingControl)이 실행될 Task다. Priority(117, 118)는 여러 Task가 동시에 실행 대기 중일 때 어느 것을 먼저 처리할지 결정하는 우선순위다.

`OsAlarm_SWC_SeatSwitch_100ms`는 `OsCounter_Main`(시간을 세는 카운터)을 기준으로, 시간이 다 되면 `OsTask_SWC_SeatSwitch_100ms`를 깨우는(Activate Task) 역할을 한다. 이게 바로 Timing Event(100ms)가 실제 하드웨어/OS 수준에서 구현되는 방식이다. SWC_SeatHeatingControl 쪽 Task에는 별도 Alarm이 필요 없는데, 이 Task는 시간이 아니라 "데이터 도착"에 반응하기 때문이다(RTE가 데이터 도착 시점에 직접 이 Task를 활성화한다).

`OsApplication0`에 Alarm과 두 Task를 등록하는 것은, 이 자원들을 하나의 OS Application(그룹) 소속으로 명시해서 관리하기 위한 작업이다.

### ② RTE Configuration

```
Configure ECU and Generate Code → Generate ECU Configuration
Next → Rte 선택 → Next → Rte: Generate SwInstance configuration 체크 → Finish
```

여기서 실제로 각 SWC(SWC_SeatSwitch, SWC_SeatHeatingControl)에 대응하는 `RteSwComponentInstance`가 생성된다. 이는 "제어기에 배치된 이 컴포넌트에 대해 제어기 관련 설정을 하겠다"는 명시화 단계다.

```
RTE event to Task Mapping
  SWC_SeatSwitch: unMapped → OsTask_SWC_SeatSwitch_100ms, TE_RE_SeatSwitch 선택 → Add
  SWC_SeatHeatingControl: unMapped → OsTask_SWC_SeatHeatingControl, DRE_RE_* 선택 → Add
```

`RteEventToTaskMapping`은 "이 Runnable을 깨우는 이 Event를, 실제로 어느 OS Task 위에서 실행시킬지"를 연결하는 작업이다. `TE_RE_SeatSwitch`(Timing Event)는 `OsTask_SWC_SeatSwitch_100ms`에, `DRE_RE_*`(Data Received Event)는 `OsTask_SWC_SeatHeatingControl`에 매핑된다. 이 매핑이 있어야 RTE가 생성하는 코드가 "이 이벤트가 발생하면 이 Task를 활성화해서 그 안에서 Runnable을 호출하라"는 실제 동작을 만들어낼 수 있다.

### ③ I/O Configuration

이 단계는 순수 소프트웨어(SWC, Runnable) 설정을 실제 물리적인 MCU 핀/타이머/ADC 자원과 연결하는 준비 작업이다.

**Port(핀) 설정**: `PTA31`(EMIOS_1_CH_14 출력 모드로 설정, PWM 신호가 나갈 물리 핀)과 `PTA11`(아날로그 입력 모드로 설정, 가변저항 값이 들어올 물리 핀)의 동작 모드를 지정한다.

**Pwm/Emios/Mcl 체인**: PWM 출력(SetDutyCycle)이 실제로 동작하려면 여러 계층의 설정이 순서대로 연결되어야 한다.
- `PwmChannel_PTA31` (Pwm 모듈의 논리 채널, 주기 8191, 초기 듀티 0)이 실제로 어떤 하드웨어 채널을 쓸지 지정해야 하므로
- `PwmEmiosChannels_CH14` (Emios 모듈의 물리 채널, OPWMB 모드)를 만들고
- 이 Emios 채널이 사용할 타이밍 기준인 `EmiosMclMasterBus_1_CH8` (Mcl 모듈의 마스터 버스, 카운터 방식/분주비 설정)을 만든 뒤
- `PwmChannel_PTA31 → PwmEmiosChannels_CH14 → EmiosMclMasterBus_1_CH8`로 참조를 끝까지 연결한다.

이 체인이 하나라도 끊기면 SetDutyCycle을 호출해도 실제 PWM 신호가 나오지 않는다.

**IoHwAb Logical 설정**: `IoHwAbAnalogInputDirectLogical_Pot`(아날로그 입력 논리 채널, 실제 ADC 그룹을 참조)과 `IoHwAbPwmLogical_LED_Blue`(PWM 출력 논리 채널, 방금 만든 `PwmChannel_PTA31`을 참조)를 만든다. 이 Logical 설정이 곧 IoHwAb Service SWC가 자동으로 갖게 되는 실제 P-Port(`P_IoHwAb…_Pot`, `P_IoHwAb…_LED_Blue`)의 근거가 된다. 즉 이 설정이 정확히(참조까지 전부) 채워져 있어야 다음 단계에서 연결할 포트가 실제로 나타난다.

### ④ I/O Mapping

```
Configure ECU and Generate Code → Service and I/O → IoHwAb
P_IoHwAb…_Pot → Respect Naming Rule 해제 → R_Pot 선택 → OK
P_IoHwAb…_LED_Red → R_LED_Red
P_IoHwAb…_SW06 → R_SW06
P_IoHwAb…_LED_Blue → R_LED_Blue
```

여기서 드디어 SWC 쪽의 Client 포트(R_Pot, R_LED_Red, R_SW06, R_LED_Blue)와, IoHwAb Service SWC 쪽의 실제 Provide 포트(P_IoHwAb…)를 서로 연결한다. `Respect Naming Rule`을 해제하는 이유는, 두 포트의 이름이 자동 매칭 규칙(이름이 비슷해야 자동으로 이어줌)에 딱 맞지 않아서, 이름 규칙과 무관하게 수동으로 정확한 짝을 골라 연결해주기 위해서다. 이 연결이 완료되어야 SWC가 요청하는 하드웨어 접근이 실제 IoHwAb 구현으로 이어진다.

---

## 7. Generate & Build

```
Build → Scons.arxml → SCons 더블클릭
All Contents → RTSW → Generation → Module → Rte 폴더
Input Files List → Add → 'App_Rte' 입력 → Add → OK
좌측 상단 망치 아이콘의 화살표 클릭 → Build
```

지금까지 arxml 모델 안에 정의한 모든 내용(SWC, 포트, Runnable, ECU 설정 등)은 아직 실제 컴파일 가능한 C 코드로 바뀌지 않은 상태다. 이 단계는 두 부분으로 나뉜다.

먼저, Rte 모듈의 코드 생성기가 `App_Rte`라는 우리가 만든 arxml 파일을 입력으로 사용하도록 빌드 설정(Input Files List)에 등록한다. 이걸 등록하지 않으면 빌드 시스템이 우리가 만든 파일의 존재를 모르고 지나쳐버린다.

등록이 끝나면 실제 빌드(망치 아이콘)를 실행한다. 이 과정에서 각 BSW/RTE 코드 생성기가 arxml 모델을 읽어 `Rte.c`, `Rte_SWC_SeatSwitch.h` 같은 실제 C 소스/헤더 파일을 만들어내고, 우리가 3단계에서 작성한 `SWC_SeatSwitch.c`, `SWC_SeatHeatingControl.c`와 함께 컴파일·링크되어 최종 실행 파일이 완성된다.

---

## 전체 흐름 한 줄 요약

VFB Level(구조 설계: 컴포넌트·포트·인터페이스·연결) → RTE Level(행동 설계: Runnable·Event·Access Point) → C Coding(실제 알고리즘) → ECU Mapping/Extract(제어기 배치) → ECU Configuration(OS Task/Alarm, Event-Task 매핑, 실제 하드웨어 핀/타이머/ADC 연결, SWC-IoHwAb 포트 연결) → Generate & Build(전체를 실행 파일로 변환). 이 순서 하나하나가 "추상적인 설계"에서 "제어기에서 실제로 도는 코드"로 점점 구체화되어가는 과정이다.
