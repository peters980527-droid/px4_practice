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

`x_sym = ca.vertcat(px, py, pz, vx, vy, vz, phi, theta, psi, ix, iy)` 는 현재 모델의 11 상태를 세로로 하나의 상태벡터로 만드는 것. 

즉, x_sym[0] = px, ..., x_sym[10] = iy

[Vertical and horizontal concatenation is performed using the functions vertcat and horzcat](https://web.casadi.org/docs/)

일반적인 12상태 모델과 현재 11상태 모델의 차이점:

일반적인 12상태 모델은 위치 3개, 선속도 3개, 자세각 3개, 각속도 pqr 3개 해서 총 12개임. 이 모델의 입력은 보통 추력, 그리고 roll pitch yaw 토크임. 즉, 흐름은 토크가 입력되면, 각속도 pqr이 변화하고 그러면 자세각이 변하고, 드론이 이동하는거임.

현재 코드는 11상태 모델임. 위치 3개, 선속도 3개, 자세각 3개, 적분상태 2개. pqr이 없음. 대신 NMPC가 토크를 직접 계산하지 않고 목표 자세각을 계산해서 (T, roll, pitch, yaw) px4에 넘겨주는 구조임. 그러면 px4에서 자세 각속도를 제어하고 그러면 모터가 제어되는 식임. 즉, pqr이랑 모터 토크를 빠르게 제어하는 일은 px4내부 제어기가 담당함. 그래서 NMPC에서는 PX4내부를 모델링 할 필요없음.

Roll 동작으로 비교해보면: 일반 12상태 모델: NMPC가 roll 토크 계산하면 roll 각속도 증가하고 roll 각도가 변화하고 그럼. 현재 모델은 NMPC가 목표 roll 각도만 계산하면 px4가 그걸 받고 모터를 제어.

적분상태 ix랑 iy는 실제 물리 상태가 아님. 위치오차를 계속 누적해서 steady state error를 제거하려고 추가한 제어기용 상태임. 그래서 만약 일반적인 12상태 모델에도 적분제어기를 추가한다면 14상태 모델이 되는것.

제어기가 상태에 추가되는 이유는 상태는 꼭 물리량만 의미하지는 않기 때문. 상태는 쉽게 말해서 현재 이후의 움직임을 계산하기 위해 기억해야 하는 값임.

교과서에도 써져있음 : [The basic approach in integral feedback is to create a state within the controller.](https://introcontrol.mit.edu/_static/fall21/extras/Feedback%20Systems%20Murray.pdf)

[“Introduce new state (accumulation of distance error).”](https://introcontrol.mit.edu/_static/fall25/lectures/Matrix_PID.pdf)

적분항은 과거부터 지금까지의 오차를 누적함

<img width="212" height="78" alt="image" src="https://github.com/user-attachments/assets/158989c8-1329-4fbd-9c26-f96c90e0e0a5" />

즉 제어 명령을 계산하려면 ix를 기억해야 하므로 제어기 내부 상태가 되는것.

이렇게 물리 상태에 제어기 상태를 붙인 것을 확장 상태(augumented state)라고 함. 



```python
T         = ca.MX.sym('T')
phi_cmd   = ca.MX.sym('phi_cmd')
theta_cmd = ca.MX.sym('theta_cmd')
psi_cmd_delta = ca.MX.sym('psi_cmd_delta')
u_sym = ca.vertcat(T, phi_cmd, theta_cmd, psi_cmd_delta)
```
이 부분은 상태가 아니라 NMPC가 결정할 제어 입력을 만드는 부분

`phi_cmd_delta`는 기준 yaw에서 얼마나 더 돌릴지 나타내는 변화량. 현재는 이 값이 0으로 고정되어 있어서 드론은 시작할 때 바라보던 yaw를 그대로 유지함. (후에 코드 나옴)

`u_sym`이 바로 px4에 들어가는게 아니라 NMPC에서 이 벡터의 최적값 u_opt를 계산한 뒤 그 중 추력과 자세 명령을 px4 메세지로 변환해서 보냄

```python
wx = ca.MX.sym('wx'); wy = ca.MX.sym('wy'); wz = ca.MX.sym('wz')
psi_ref = ca.MX.sym('psi_ref')
p_sym = ca.vertcat(wx, wy, wz, psi_ref)
```
`wx`는 x 방향 바람임. 예를 들어 wx가 2라면 x방향으로 2m/s 의 바람이 있다는 뜻.

`psi_ref`는 유지하고 싶은 기준 yaw. yaw는 px4에서 알려주기 때문에 외부 설정값임. 그래서 바람과 함께 외부 입력값으로 들어감.

```python
vrel_x = vx - wx;  vrel_y = vy - wy;  vrel_z = vz - wz
drag_coeff = 0.5 * rho * Cd * A_f / m
```
`vrel`은 상대속도. 예를들어 x축에서 드론속도 vx = 2, 바람속도 -3 ==> vrel 은 5m/s. 즉, 드론은 공기를 5m/s 로 맞는 것처럼 느낌.

`drag_coeff`는 항력계수가 아닌 (항력계수는 Cd임!) 공기밀도, 면적, 질량까지 포함한 계산용 계수인데 상대속도를 항력 가속도로 바꿔주는 비례계수임. 상대속도는 매번 바뀌니까 그거만 빼고 식을 좀더 간단하고 짧게 만들려고 따로 저장해놓는거임 매번 계산안해도 되게끔. 원래 항력 가속도 형태가 <img width="284" height="93" alt="image" src="https://github.com/user-attachments/assets/a1e0d6d3-cb8f-44a9-a5b6-c3ca881779b6" /> 이 형태임

앞부분 <img width="121" height="70" alt="image" src="https://github.com/user-attachments/assets/30f2dd68-aa7e-4121-b282-9f56ef0426b8" /> 을 따로 drag coeff로 저장한것.

여기서 Cd는 우선 임의의 값임. 측정하면 더 좋긴한데...

```python
ax = -(T/m)*(ca.cos(phi)*ca.sin(theta)*ca.cos(psi) + ca.sin(phi)*ca.sin(psi)) - drag_coeff*vrel_x*ca.sqrt(vrel_x**2 + 0.01)
ay = -(T/m)*(ca.cos(phi)*ca.sin(theta)*ca.sin(psi) - ca.sin(phi)*ca.cos(psi)) - drag_coeff*vrel_y*ca.sqrt(vrel_y**2 + 0.01)
az =  (T/m)*ca.cos(phi)*ca.cos(theta) - g - drag_coeff*vrel_z*ca.sqrt(vrel_z**2 + 0.01)
```
우선 여기서 구하는 가속도는  nmpc에서 미래를 예측할때 필요한 가속도를 구하려는 것. 그리고 실제 가속도를 구하는 공식이기도 함.

흐름은 우선 추력 자세 바람 후보들로 ax를 계산하고 그 ax로 미래 속도 vx, 위치 px 를 계산하는거임

즉, 실제 가속도를 나타내는 물리 기반 식이지만, Cd, 면적, 추력 등이 정확하지 않으면 센서가 측정한 실제 가속도와 완벽히 똑같을 수는 없음.

PX4에도 가속도를 측정하는 센서가 있지만 이건 현재 상태의 가속도만 알려줄뿐, 미래를 예측하는 용도로는 사용될수 없음.

[Quan Quan Introduction to Multicopter Design and Control](https://www.researchgate.net/profile/Quan-Quan-2/publication/350758388_Correction_to_Multicopter_Design_and_Control_Practice/links/652f92cdb5c77c79f9c415b4/Correction-to-Multicopter-Design-and-Control-Practice.pdf?__cf_chl_tk=vbyK_8sqOiUumbGgMPN30Xxyq5dRXm0i40C7bAwr4N8-1784863972-1.0.1.1-UYe_3FTHNDoK4KdR.jk2aTRzDXjcEvUHVYTzPvCpoKI) 를 보면 <img width="323" height="80" alt="image" src="https://github.com/user-attachments/assets/f3144c88-6780-4c72-9a22-00d0c57b3747" /> 의 식이 나옴. 즉, 가속도 = 중력 + 자세에따라 회전된 추력임. Re3는 회전행렬 식, 즉, 
<img width="305" height="91" alt="image" src="https://github.com/user-attachments/assets/179491cc-6093-4cfa-9a83-a57ff5d506ce" />

나중에 후술할거지만 전체적인 흐름을 보면:

- 미래 바람 wx 입력
- vrel_x = vx - wx 계산
- 바람 항력 가속도 계산
- ax가 바람 방향으로 변함
- 미래 vx와 px가 밀릴 것으로 예측
- nmpc가 반대 방향의 roll/pitch를 계산 (추력, pitch, roll 을 이것저것 넣어보면서 베스트 후보들을 만듦. 그중에 cost가 가장 적은거를 뽑는거임)
- 바람 상쇄

ax drag 는 드론이 바람이 없어도 움직일때 생기는 항력이랑 바람을 줘서 생기는 항력이랑 똑같음.

항력의 원래 공식은 <img width="210" height="65" alt="image" src="https://github.com/user-attachments/assets/93d1b330-9399-47c0-8a53-3ce6a4268d54" />, 그리고 방향까지 포함한 1차원에서는 <img width="312" height="76" alt="image" src="https://github.com/user-attachments/assets/dfe2b498-6a68-4035-b32b-57aafce2c706" /> 로 표현이 가능함. (Multicopter Design and Control Practice의 6.63 식을 참고)

현재 코드에서 쓴 `ca.sqrt(vrel_x**2 + 0.01)` 는 v = 0 부근에서도 함수가 부드럽게 미분되도록 만든 NMPC용 smooth approximation 임. 

[Most Computationally Efficient Smooth Approximation to ∣x∣](https://www.cs.utep.edu/vladik/2013/tr13-44.pdf)

안해도 성능이 저하되거나 그런건 아니지만 간혹 solver가 fail하거나 그럴수 있음.


##################################################################################################################################################################################################
전체적인 구조
##################################################################################################################################################################################################
1. PX4로 부터 11개의 상태 중 9개의 현재상태를 받음. 적분상태는 코드 내부에서 위치 오차를 누적해서 만든 값이기 때문에 외부값이 아님.
   - Vicon, IMU 에서 얻은 위치, 속도, 자세(주로 IMU) 값을 PX4 내부에 있는 EKF로 융합해서 각각의 상태들을 추정(더 정확).
   - 가속도는 따로 안받음. 위에서 설명했듯이 가속도는 NMPC 모델안에서 추력, 자세, 속도, 바람을 이용해서 계산함 (미래 예측용 가속도).
   - 그래도 여전히 EKF에서는 위치 속도 자세를 추정할때 센서 가속도 값을 사용하긴 하는듯. 하지만 NMPC는 안받음!

2. PX4로부터 현재 상태가 0.02초마다 들어오는지는 모르겠지만 NMPC는 어쨌든 0.02초에 한번씩 현재 상태를 받음.
   - 0.02초마다 일어나는 일임!!
   - 현재 상태를 시작점으로 고정함.
   - acados가 임시 명령 후보를 넣어봄.
   - 그 명령을 쓰면 앞으로 어떻게 움직일지 예측함.
   - 예측 결과가 목표에 얼마나 가까운지 비용을 계산함.
   - 비용이 줄어들도록 명령 후보를 계속 수정함 (최적화).
   - 이걸 0.02초마다 한번에 미래 20스텝의 명령을 한번에 최적화 까지 다 계산함.
   - 즉, acados가 u_0 부터 u_19까지 계산하고 각 명령에 따른 미래 상태 x_1 부터 x_20 까지 예측하고 목표에 가장 잘 도달하는 명령만 뽑아서 배열 선택.
   - 실제 드론에는 20개의 명령을 전부 보내지는 않고 첫번째 명령만 보냄. 그리고 0.02초 뒤에 또 실제 상태를 다시 받아서 20스텝을 새로 계산하는식임. receding horizon

3. 2번에서 "그 명령을 쓰면 앞으로 어떻게 움직이지 예측" 하는 부분이 동역학의 핵심임
   - 우선 처음에 명령 20개를 전부 만들어 놓음
   - 그 명령 중 첫번째를 첫번째 스텝에 현재 상태랑 계산을 해서 미래 상태를 계산함.
   - 첫번째 상태에서 계산한 미래 상태가 두번째 스텝의 현재 상태로 쓰이는 구조임. 그러면 그거랑 미리 정해둔 두번째 명령이랑 계산해서 또 세번째 스텝에서 쓸 미래 상태를 정하고.
   - 비용도 각 스텝을 지날때마다 계산됌. 그 비용들은 마지막에 다 더해져서 총 비용을 줌.
   - 그럼 0.02초에 그 비용을 찾는 계산을 몇번해서 명령을 찾아내느냐가 궁금할 수 있는데 답은 현재 코드에서 쓰는 SQP RTI (Sequential Quadratic Programming - Real Time Iteration) 에 달렸음.
   - SQP RTI는 0.02초 안에 빠르게 계산해야 하므로, 한 제어 주기 안에서 끝까지 반복해서 완벽한 정답을 찾기보다는 비용이 줄어드는 방향을 계산해서 명령 묶음을 한번만 개선하고 바로 종료하는 식임.
   - RTI가 힌 제어 주기당 SQP 갱신을 한번만 수행하는 방식임. SQP는 명령 후보 주변, 즉 현재 명령에서 앞뒤 명령중 어느거를 선택해야 비용이 주는지 계산을해서 비용이 더 감소하는 방향으로 한번 더 갱신하는 식임.
  
4. 어떻게 다음 미래 상태를 계산해서 예측하는지
   - 우선 한 스텝 계산에 필요한 값들을 준비해야함
   - 첫 번째 예측 스탭에서는 현재 드론의 상태, 첫번째 명령 후보, 첫번째 스텝에서 사용할 바람 (현재 또는 예측된 바람)
   - 1. 드론이 느끼는 상대 바람 계산 => vrel_x = vx - wx
     2. 방금 구한 상대 속도로 바람이 드론을 얼마나 미는지 계산해야함. 즉, 항력 가속도를 계산 => adrag_x.
     3. 추력과 현재 자세로 가속도를 계산함. 명령 후보 u_0에 있는 추력과 현재 상태에 들어있는 자세를 이용해서 추력 가속을 계산함. 코드의 ax 안의 `-(T/m)*(ca.cos(phi)*ca.sin(theta)*ca.cos(psi) + ca.sin(phi)*ca.sin(psi))`가 추력 가속도를 계산하는 부분임.
     4. 이제 구한 항력 가속도, 추력 가속도, 그리고 z방향은 중력까지 더해서 총 가속도를 구함. 이게 각 방향별로 ax, ay, az가 되는거임.
     5. 이렇게 구한 ax ay az는 f_expr 즉, 변화율 벡터에 들어감. ax ay az는 현재 상태의 속도와 바람이 들어가서 만들어진 애들이라 얘를 적분해서 vx vy vz를 찾으면 그게 또 f_expr안에 속도 변화율이 됌.
     6. 자세 변화율도 자세 명령에 자세 목표를 뺀거에 tau를 나눠서 구함.
     7. 즉, `f_expr = [
    vx,
    vy,
    vz,
    ax(x, u, w),
    ay(x, u, w),
    az(x, u, w),
    (phi_cmd - phi) / tau,
    (theta_cmd - theta) / tau,
    (psi_cmd - psi) / tau,
    px,
    py
]` 사실 이런 모양인거임.

    8. px, py도 마찬가지로 vx vy vz를 erk 적분기가 구해줌.
    9. 그렇게 전체 f_expr이 구해지면 이제 이걸 또 ERK 적분기로 전체를 적분함. 그게 이제 바로 다음스텝에서 쓸 미래 상태가 되는거임!!!
















