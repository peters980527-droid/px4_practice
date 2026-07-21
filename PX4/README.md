```python
from std_msgs.msg import Empty
```
ROS2의 Package `std_msgs.msg` 에서 `Empty`를 불러옴.

나중에 `/wind_start` 로 팬 시작 신호를 보내기 위해 사용함.


```python
import os
```
Python에서 파일 경로와 환경변수 등 기능을 불러오기 위해서 사용.

```python
import time
```
Python의 시간 관련 기능을 불러옴.

NMPC 시작 후 얼마나 시간이 지났는지 계산하기 위해 사용.

```python
os.environ['ACADOS_SOURCE_DIR'] = os.path.expanduser('~/acados')
```
`expanduser()`는 경로를 만들고, `os.environ`은 그 경로를 공용 저장소에 넣으며, `AcadosOcpSolver`의 내부 라이브러리가 나중에 그 값을 읽는다.

```python
import rclpy
```
ROS 2의 Python 기능 모음인 rclpy를 불러오는 줄

이 코드 아래에서 `rclpy.init()`, `rclpy.spin(node)`, `rclpy.shutdown()`을 사용하기 위해 필요

ROS 2 공식 Python API 임.

["The first lines of code after the comments import rclpy so its Node class can be used."](https://docs.ros.org/en/jazzy/Tutorials/Beginner-Client-Libraries/Writing-A-Simple-Py-Publisher-And-Subscriber.html?utm_source=chatgpt.com)
```python
from rclpy.node import Node
```
`rclpy.node` 모듈에서 `Node` 클래스만 가져오는 코드

아래에서 NMPCController를 ROS 2 노드로 만들기 위해 사용 (ex. class NMPCController(Node):)

["The first lines of code after the comments import rclpy so its Node class can be used."](https://docs.ros.org/en/jazzy/Tutorials/Beginner-Client-Libraries/Writing-A-Simple-Py-Publisher-And-Subscriber.html?utm_source=chatgpt.com)

```python
from rclpy.qos import QoSProfile, ReliabilityPolicy, HistoryPolicy
```
ROS 2 통신 방식을 설정하는 세 가지 기능

- `QoSProfile`: 통신 설정들을 하나로 묶음

- `ReliabilityPolicy`: 메시지 전달 신뢰성 설정

- `HistoryPolicy`: 이전 메시지를 얼마나 보관할지 설정

qos는 PX4가 보내는 위치·자세 데이터를 구독할 때 사용

PX4의 발행 설정과 ROS 2 구독자의 기본 설정이 서로 맞지 않으므로, ROS 2 구독 코드에서 호환되는 QoS를 지정해야 한다.

[PX4 QoS settings for publishers are incompatible with the default QoS settings for ROS 2 subscribers.](https://docs.px4.io/main/en/middleware/uxrce_dds#px4-ros-2-qos-settings)

```python
import numpy as np
```
`NumPy`는 숫자를 여러 개 묶어서 벡터·행렬로 계산할 때 사용하는 도구

```python
import casadi as ca
```
최적화·수식 계산 라이브러리인 `CasADi`를 불러오고, 코드에서는 ca라는 짧은 이름으로 사용하는 줄

[CasADi is an open-source tool for nonlinear optimization and algorithmic differentiation.](https://web.casadi.org/?utm_source=chatgpt.com)

사람이 드론 동역학의 모든 편미분식을 직접 작성하지 않아도 되게끔 수학식을 작성하고 번역하는 설계자

그걸 이제 푸는게 ACADOS

[acados is written in C, but control problems can be conveniently formulated using the CasADi symbolic framework via the high-level acados interfaces to the programming languages Python, MATLAB, and Octave.](https://docs.acados.org/?utm_source=chatgpt.com)

[CasADi 공식 문서에서 symbolic framework와 자동 미분 확인하기](https://web.casadi.org/docs/?utm_source=chatgpt.com)

```python
from acados_template import AcadosOcp, AcadosOcpSolver, AcadosModel
```
`acados_template`에서 NMPC 문제를 만들고 실행하는 데 필요한 세 가지 클래스를 가져오는 줄

`acados_template`은 os에 저장된 ~/acados 안에 있음. 거기서 세 가지 클래스를 빼오는 것.

- `AcadosModel`: 드론의 상태·입력·동역학 수식을 담는 그릇

- `AcadosOcp`: 비용함수·제약조건·예측구간 등을 포함한 최적제어 문제 전체

- `AcadosOcpSolver`: 완성된 문제를 바탕으로 solver를 생성하고 실행하는 기능

[acados class에 대한 설명은 여기 참](https://docs.acados.org/python_interface/index.html?utm_source=chatgpt.com)

```python
from px4_msgs.msg import (
    OffboardControlMode,
    TrajectorySetpoint,
    VehicleAttitudeSetpoint,
    VehicleCommand,
    VehicleLocalPosition,
    VehicleAttitude
)
```
px4_msgs 패키지에서 PX4와 ROS 2가 주고받는 메시지 형식 6개를 가져오는 코드

[px4_msg 리스트1](https://docs.px4.io/main/en/middleware/dds_topics?utm_source=chatgpt.com)

[px4_msg 리스트2](https://github.com/PX4/px4_msgs)

- `OffboardControlMode`	위치제어인지 자세제어인지 PX4에 알림

- `TrajectorySetpoint`	이륙 중 목표 위치·고도 전달

- `VehicleAttitudeSetpoint`	NMPC의 자세와 추력 명령 전달

- `VehicleCommand` Offboard 전환, 시동, 착륙 명령

- `VehicleLocalPosition` PX4에서 위치와 속도를 받음 (px4 to nmpc)

- `VehicleAttitude`	PX4에서 현재 자세 quaternion을 받음 (px4 to nmpc)

NMPC는 현재 위치, 속도, 자세가 필요해서 `VehicleLocalPosition`, `VehicleAttitude`를 필요로 함. 이륙할때는 px4로 목표 위치를 보내므로 `TrajectorySetpoint`가 필요한거고, NMPC 전환 후 PX4로 자세와 추력을 보내므로 `VehicleAttitudeSetpoint` 필요. PX4 에 현재 사용하는 Offboard 제어방식을 알려줘야 하므로 `OffboardControlMode`, 그리고 Offboard로 전환, Arm, Land 명령을 보내니까 `VehicleCommand`.

```python
FAN_PERCENT      = 80.0   # 팬 파워 %  <- laptop/fan_node.py 의 FAN_POWER 와 반드시 동일!
FAN_DELAY_BEFORE = 3.0    # /wind_start 후 fan_node가 팬을 켜기까지 [s]
WIND_DURATION    = 20.0   # fan_node의 팬 유지 시간 [s]
USE_WIND_PREVIEW = False  # preview ON/OFF
USE_INTEGRAL     = True   # 적분항 ON/OFF
Q_INT            = 10.0   # 적분 가중치 (VICON 노이즈로 ix 떨리면 5 로)
INT_LIM          = 2.0    # anti-windup: 적분 누적 한계 [m*s]
ANGLE_LIM        = 45.0   # roll/pitch 제한 (deg). 90%+ 바람이면 45 권장
THRUST_MIN       = 0.35   # 정규화 추력 하한
THRUST_MAX       = 0.92   # 정규화 추력 상한. ANGLE_LIM=45 면 0.92 권장
Q_POS            = 60.0   # x,y 위치 가중치 (검증값 60)
```
실험 설정 값

`INT_LIM`은 적분값에 제한을 두는거임. 예를 들어서 드론이 목표 위치에서 4초동안 0.5m 벗어나 있다면 1초에 한번 누적되니까 `0.5m * 4s = 2 m*s`. 이게 문제가 되는 부분은 적분 누적이 너무 커지면 반대 방향에 대한 보상이 더 강해지는데 그러면 바람이 갑자기 사라지면 오버슈트가 나올 수 있음. 반대로 제한을 너무 작게 두면 적분 효과가 부족하게 나옴.

값을 바꿔가면서 튜닝을 해야함.

```python
WIND_MAX      = 16.0 * (FAN_PERCENT / 100.0)   # 100% = 16 m/s 선형 가정
ANGLE_LIM_RAD = np.radians(ANGLE_LIM)          # solver 제약 + clip 에 함께 사용
Q_INT_EFF     = Q_INT if USE_INTEGRAL else 0.0
HOVER_THRUST  = 0.59
TAKEOFF_ALT   = 1.0
```
`WIND_MAX`에 선형 가정이라는건 바람의 가속도 증가가 선형인게 아니라 특정 퍼센트에서의 바람속도가 정확하게 일치한다고 가정해서 선형이라고 한것. 예를들어서 100%에 16m/s이면 50%엔 8m/s. 하지만 실제로는 모터 회전수, 팬의 공기역학, 팬과 드론의 거리차이 등 변수가 적용해서 비선형임.

`ANGLE_LIM_RAD = np.radians(ANGLE_LIM)`는 각도제한 45도를 라디안 값으로 바꾼것. 약 0.785 rad. CasADi·acados의 삼각함수와 자세 계산에서는 각도를 보통 라디안으로 사용하기 때문에 변환.


```python
m   = 1.9
g   = 9.81
Cd  = 1.1
A_f = 0.0876
rho = 1.225
tau = 0.30
dt  = 0.1
N   = 20
CTRL_HZ = 50
CTRL_DT = 1.0 / CTRL_HZ
```
`Cd`는 Drag Coefficient, 즉 항력계수. 




