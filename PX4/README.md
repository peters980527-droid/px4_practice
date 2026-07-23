[FULL FLIGHT_CODE](https://github.com/peters980527-droid/px4_practice/blob/main/PX4/FLIGHT_CODE.md)



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
```
`Cd`는 Drag Coefficient, 즉 항력계수.

1.1의 값은 우선 대중적인 값을 쓴거고 측정하는 방법은 ...

`A_f`값도 실측한게 아닌 `drag_coeff = 0.5 * rho * Cd * A_f / m` 공식에서 역산해서 구해낸것 (Cd가 1.1 이라고 가정하고).

`tau`는 time constant임. 응답속도를 나타낸 숫자. 단위는 초(s). 이것도 0.3으로 가정한거고 정확한 측정은 직접 roll pitch 명령을 주고 반응을 재야함. 나중에.

`dt` 와 `N`은 미래를 나누어 보는 시간 가격과 스텝임. 예측 구간은 `dt * N`으로 결정됌. 즉 2초. 

0.1을 더 낮춰서 더 촘촘하게 볼 수 있지만 예측시간을 유지하려면 `N`을 늘려야하는데 그러면 후술할 문제점들이 발생함.

반대로 너무 올리면 계산은 가벼워지지만 너무 거칠게 보상함.

에측 시간은 1초, 2초, 3초 정도로해서 나중에 데이터를 비교하면 더 좋을듯. 

*** 실제 센서를 이용한 preview와 지금 스케줄 preview의 가장 큰 차이는 실제센서는 바람속도와 드론과 센서의 거리에 따라 예측 성능이 달라진다는 것. 예를 들어서 거리가 5m 이고 바람이 5m/s 로 불고있다면 센서에서 바람을 잰 순간부터 1초만에 드론에 바람이 도달한다는것. 그렇게 되면 2초의 예측구간에서 20스텝에서 10스텝부터 갑자기 바람이 등장함. 즉, 성능이 저하될 수 있음.***

***본 연구에서는 LiDAR와 같은 실시간 원격 풍속계의 직접적인 센서 모델을 구현하지 않고, 팬의 작동 명령과 기체 위치에서 측정된 바람 도달 지연을 이용하여 미래 외란 정보를 생성하였다. 따라서 본 실험은 이상적인 wind-preview 정보가 주어졌을 때 NMPC의 외란 선제 대응 성능을 검증하는 것을 목적으로 한다.***

`dt`와 `N`의 값에 대해서는 참고할 만한 소스는 [MATLAB 공식 예재](https://www.mathworks.com/help/mpc/ug/control-of-quadrotor-using-nonlinear-model-predictive-control.html) , [논문 예재](https://cdn.syscop.de/publications/Zanelli2018.pdf) 등 찾아볼 수 있음. 현재 값은 즉, 크게 벗어난 값이 아님.

나중에 `N`을 바꿔가면서 데이터 비교하는게 좋을듯.

!!! `dt`는 계산 주기가 아님. 시간간격이고 계산 주기는 후술할 `CTRL_HZ` 와 `CTRL_DT` 가 담당함.

```python
CTRL_HZ = 50
CTRL_DT = 1.0 / CTRL_HZ
```
`CTRL_HZ`는 50hz, 1초에 50번, 즉 0.02초 마다 NMPC가 계산함. 0.02초마다 20스텝을 계산하는것. 바람 변화는 각 최적화 내부에서는 0.1초 단위로 표현되지만, 0.02초마다 예측을 새로 만들기 때문에 바람 시작까지 남은 시간은 계속 0.02초 단위로 갱신되는 구조

주기를 50으로 한 이유는 우선 가정해서 빠르게 한것. 나중에 실험으로 callback 해서 계산 속도를 재야할듯. 50hz 면 전체 계산 과정이 20ms 안에 안정적으로 끝나야함.

```python
THRUST_MIN_N = (THRUST_MIN / HOVER_THRUST) * m * g
THRUST_MAX_N = (THRUST_MAX / HOVER_THRUST) * m * g
```
`THRUST_MIN_N`은 PX4에서 쓰는 정규화된 추력(최소)을 NMPC가 쓰는 Newton으로 변환해주는 것. PX4에 이미 min으로 0.35, max로 0.92를 쓰니까 NMPC도 동일하게 이걸 알고 있어야함.

min은 필요없다고 생각할 수 있는데 그러면 만약 추력이 0이 되는 계산이 나오면 드론은 자유낙하를 함. 최소한 랜딩할 수 있게 0.35로 제한을 두는것.

[The thrust is considered to be non-negative and limited according to 0 ≤ T (t) ≤ Tmax, for all t ∈ R≥0, (8) where Tmax > g is the maximal thrust.](https://heemels.tue.nl/content/papers/AndLef_TAC24a.pdf)

`THRUST_MAX_N`도 마찬가지로 max 로 설정한 0.92를 NMPC가 알 수 있게 변환해주는 식.

NMPC가 1 이상의 추력을 계산해버리면 예측이 완전히 틀려버리기 때문에 (모터 최대 추력은 1) 제한을 둬야함. [Furthermore, rotor saturation should be properly addressed in a controller to avoid performance degradation or even instability due to a gap between the commanded input and the actual input during saturation.](https://arxiv.org/abs/2404.11320)

또한 드론이 기울었을때 모터마다 내는 출력이 다르게 되는데 이미 총추력을 쓰는중이라면 기울여서 대응을 못함. 배터리, 모터 과부화 등등 안전 문제도 있어서 0.85 에서 0.92 가 적당한듯.

[If mixing becomes saturated towards the upper bound the commanded thrust is reduced to ensure that no motor is commanded to deliver more than 100% thrust.](https://docs.px4.io/main/en/config_mc/pid_tuning_guide_multicopter)

```python
px    = ca.MX.sym('px');    py    = ca.MX.sym('py');    pz    = ca.MX.sym('pz')
vx    = ca.MX.sym('vx');    vy    = ca.MX.sym('vy');    vz    = ca.MX.sym('vz')
phi   = ca.MX.sym('phi');   theta = ca.MX.sym('theta'); psi   = ca.MX.sym('psi')
ix    = ca.MX.sym('ix');    iy    = ca.MX.sym('iy')      # 적분 상태 (오차 누적)
x_sym = ca.vertcat(px, py, pz, vx, vy, vz, phi, theta, psi, ix, iy)
```
현재 모델 (11상태) 를 각각 CasADi 에 변수로 지정해주는거임.

`px    = ca.MX.sym('px')` 만 일단 보면 x의 위치를 ca (CasADi 라이브러리), MX(matrix 행렬 1x1), sym (symbolic variable, 기호 변수 생성), ('px') CasADi 내부에서 표시할 변수 이름 지정. 해주는거임

[CasADi 문서](https://web.casadi.org/docs/#the-mx-symbolics)

CasADi 는 스칼라, 벡터, 행렬을 전부 같은 방식으로 처리하는게 좋아서 px같은 스칼라도 1x1 행렬 표현으로 다룸.

그리고 어차피 나중에 11x1 상태 벡터로 표현해야해서 MX (matrix expression)으로 표현하는것.

CasADi에는 `SX`라는 비슷한 역할이 있는데 우리는 벡터, 행렬에 더 계산이 들어가서 MX가 더 좋음. CasADi에서도 그걸 설명함 “MX can be more economical when working with operations that are naturally vector or matrix valued.”






