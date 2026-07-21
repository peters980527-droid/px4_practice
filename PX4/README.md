```python
from std_msgs.msg import Empty
```
ROS2의 Package std_msgs.msg 에서 Empty를 불러옴.

나중에 /wind_start 로 팬 시작 신호를 보내기 위해 사용함.


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
expanduser()는 경로를 만들고, os.environ은 그 경로를 공용 저장소에 넣으며, AcadosOcpSolver의 내부 라이브러리가 나중에 그 값을 읽는다.

```python
import rclpy
```
ROS 2의 Python 기능 모음인 rclpy를 불러오는 줄

이 코드 아래에서 rclpy.init(), rclpy.spin(node), rclpy.shutdown()을 사용하기 위해 필요

ROS 2 공식 Python API 임.

["The first lines of code after the comments import rclpy so its Node class can be used."](https://docs.ros.org/en/jazzy/Tutorials/Beginner-Client-Libraries/Writing-A-Simple-Py-Publisher-And-Subscriber.html?utm_source=chatgpt.com)
```python
from rclpy.node import Node
```
rclpy.node 모듈에서 Node 클래스만 가져오는 코드

아래에서 NMPCController를 ROS 2 노드로 만들기 위해 사용 (ex. class NMPCController(Node):)

["The first lines of code after the comments import rclpy so its Node class can be used."](https://docs.ros.org/en/jazzy/Tutorials/Beginner-Client-Libraries/Writing-A-Simple-Py-Publisher-And-Subscriber.html?utm_source=chatgpt.com)

```python
from rclpy.qos import QoSProfile, ReliabilityPolicy, HistoryPolicy
```
ROS 2 통신 방식을 설정하는 세 가지 기능

- QoSProfile: 통신 설정들을 하나로 묶음

- ReliabilityPolicy: 메시지 전달 신뢰성 설정

- HistoryPolicy: 이전 메시지를 얼마나 보관할지 설정

qos는 PX4가 보내는 위치·자세 데이터를 구독할 때 사용

PX4의 발행 설정과 ROS 2 구독자의 기본 설정이 서로 맞지 않으므로, ROS 2 구독 코드에서 호환되는 QoS를 지정해야 한다.

[PX4 QoS settings for publishers are incompatible with the default QoS settings for ROS 2 subscribers.](https://docs.px4.io/main/en/middleware/uxrce_dds#px4-ros-2-qos-settings)

```python
import numpy as np
```
NumPy는 숫자를 여러 개 묶어서 벡터·행렬로 계산할 때 사용하는 도구

```python
import casadi as ca
```
최적화·수식 계산 라이브러리인 CasADi를 불러오고, 코드에서는 ca라는 짧은 이름으로 사용하는 줄

[CasADi is an open-source tool for nonlinear optimization and algorithmic differentiation.](https://web.casadi.org/?utm_source=chatgpt.com)






















