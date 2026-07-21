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

"The first lines of code after the comments import rclpy so its Node class can be used."

```python
from rclpy.node import Node
```
rclpy.node 모듈에서 Node 클래스만 가져오는 코드

아래에서 NMPCController를 ROS 2 노드로 만들기 위해 사용 (ex. class NMPCController(Node):)

["The first lines of code after the comments import rclpy so its Node class can be used."](https://docs.ros.org/en/jazzy/Tutorials/Beginner-Client-Libraries/Writing-A-Simple-Py-Publisher-And-Subscriber.html?utm_source=chatgpt.com)

