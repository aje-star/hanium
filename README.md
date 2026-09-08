## 시스템 동작 흐름

```text
[ReSpeaker 6채널 마이크]
        ↓
[음성 인식 + 구조 요청 키워드 탐지]
        ↓
[GCC-PHAT 기반 음원 방향 추정]
        ↓
[방향각에 따라 로봇 회전]
        ↓
[4족보행 알고리즘으로 전진]
        ↓
[2D LiDAR + RF2O + SLAM Toolbox]
        ↓
[장애물 회피 / 지도 생성 / 구조대상자 위치 전송]
```

## 핵심 소스코드

### 1. 구조 요청 음성 감지 및 방향 추정

ReSpeaker XVF3800의 다중 마이크 입력에서 구조 요청 키워드를 감지합니다.  
키워드가 감지되면 GCC-PHAT 기반 TDOA(Time Difference of Arrival) 방식으로 채널 간 도달 시간 차이를 계산하고, 음성의 도착 방향을 추정합니다.

```python
import numpy as np

TARGET_KEYWORDS = ["살려", "도와", "도움", "구해", "여기", "사람"]

def is_rescue_request(text: str) -> bool:
    """음성 인식 문장에서 구조 요청 키워드가 있는지 확인"""
    text = text.replace(" ", "")
    return any(keyword in text for keyword in TARGET_KEYWORDS)

def calculate_direction_vector(mic1, mic2, mic3, mic4):
    """
    4개 마이크 채널의 시간차 정보를 이용하여
    음원 방향 벡터와 방위각을 반환한다.
    """
    x = (mic1 - mic2) + (mic4 - mic3)
    y = (mic1 - mic4) + (mic2 - mic3)

    angle = np.degrees(np.arctan2(y, x))
    if angle < 0:
        angle += 360

    direction = np.array([
        np.cos(np.radians(angle)),
        np.sin(np.radians(angle))
    ])

    return direction, angle

def process_voice_result(text, mic1, mic2, mic3, mic4):
    """구조 요청 확인 후 방향을 계산하는 메인 함수"""
    if is_rescue_request(text):
        direction, angle = calculate_direction_vector(
            mic1, mic2, mic3, mic4
        )

        print(f"[구조 요청 감지] 인식 문장: {text}")
        print(f"[음원 방향] {angle:.1f}°")
        print(f"[방향 벡터] x={direction:.2f}, y={direction:.2f}")[2]

        return angle

    return None
```

### 2. 음원 방향에 따른 로봇 행동 결정

추정된 음성 방향각을 기준으로 로봇이 좌회전, 우회전 또는 전진하도록 결정합니다.  
정면 기준 오차 범위를 두어 센서 노이즈로 인한 불필요한 미세 회전을 줄입니다.

```python
def decide_motion_from_angle(angle: float) -> str:
    """
    정면을 0°로 가정한다.
    - 0° 부근: 전진
    - 0~180°: 우회전
    - 180~360°: 좌회전
    """
    FRONT_TOLERANCE = 20

    if angle is None:
        return "STOP"

    if angle <= FRONT_TOLERANCE or angle >= 360 - FRONT_TOLERANCE:
        return "FORWARD"

    if 0 < angle < 180:
        return "TURN_RIGHT"

    return "TURN_LEFT"
```

```python
angle = process_voice_result(
    text="살려주세요 여기 사람이 있어요",
    mic1=0.0021,
    mic2=0.0015,
    mic3=0.0007,
    mic4=0.0012
)

motion = decide_motion_from_angle(angle)
print(f"[이동 명령] {motion}")
```

### 3. LX-16A 기반 4족보행 모터 제어

각 다리는 수직 관절과 수평 관절, 총 2개의 LX-16A 서보모터로 구성합니다.  
총 8개의 서보모터를 제어하여 서기, 전진, 좌회전, 우회전 동작을 수행합니다.

```python
import time

# 실제 환경에서는 LX16A 라이브러리 또는 시리얼 제어 클래스로 교체한다.
class RobotServo:
    def move_time_write(self, servo_id: int, position: int, move_time: int):
        """
        LX-16A 서보모터 목표 위치 제어
        position 범위: 0 ~ 1000
        move_time 단위: ms
        """
        position = max(0, min(1000, position))
        print(
            f"[SERVO] id={servo_id}, "
            f"position={position}, "
            f"time={move_time}ms"
        )

servo = RobotServo()

LEGS = {
    "front_left":  {"vertical": 1, "horizontal": 2},
    "front_right": {"vertical": 3, "horizontal": 4},
    "rear_left":   {"vertical": 5, "horizontal": 6},
    "rear_right":  {"vertical": 7, "horizontal": 8},
}

# 로봇 조립 상태에 맞추어 반드시 보정해야 하는 기준 자세값
POSTURE = {
    "stand": {
        "front_left":  {"vertical": 520, "horizontal": 500},
        "front_right": {"vertical": 480, "horizontal": 500},
        "rear_left":   {"vertical": 520, "horizontal": 500},
        "rear_right":  {"vertical": 480, "horizontal": 500},
    },
    "lift": {
        "front_left":  {"vertical": 430, "horizontal": 500},
        "front_right": {"vertical": 570, "horizontal": 500},
        "rear_left":   {"vertical": 430, "horizontal": 500},
        "rear_right":  {"vertical": 570, "horizontal": 500},
    }
}

def set_leg_pose(leg_name: str, vertical: int, horizontal: int, move_time=300):
    """한 다리의 수직·수평 관절을 동시에 제어"""
    leg = LEGS[leg_name]

    servo.move_time_write(
        leg["vertical"],
        vertical,
        move_time
    )

    servo.move_time_write(
        leg["horizontal"],
        horizontal,
        move_time
    )

def set_robot_pose(pose_name: str, move_time=500):
    """stand 등 미리 정의한 전신 자세로 이동"""
    pose = POSTURE[pose_name]

    for leg_name, values in pose.items():
        set_leg_pose(
            leg_name,
            values["vertical"],
            values["horizontal"],
            move_time
        )

def stand():
    """기본 기립 자세"""
    set_robot_pose("stand", move_time=600)
    time.sleep(0.6)

def stop():
    """안정 자세로 정지"""
    stand()
    print("[ROBOT] 정지 및 자세 안정화")
```

### 4. 4족보행 전진 알고리즘

대각선 다리를 번갈아 들어 이동하는 **Trot Gait** 방식의 예시입니다.  
한쪽 대각선 다리가 이동하는 동안 반대쪽 대각선 다리는 지면을 지지하여 균형을 유지합니다.

```python
def move_leg_forward(leg_name: str, direction=80):
    """다리를 들어 올린 뒤 전방으로 이동"""
    base = POSTURE["stand"][leg_name]

    # 1) 다리 들어 올리기
    set_leg_pose(
        leg_name,
        POSTURE["lift"][leg_name]["vertical"],
        base["horizontal"],
        move_time=200
    )

    # 2) 수평 관절을 전방 방향으로 이동
    set_leg_pose(
        leg_name,
        POSTURE["lift"][leg_name]["vertical"],
        base["horizontal"] + direction,
        move_time=250
    )

    # 3) 지면에 내려 지지
    set_leg_pose(
        leg_name,
        base["vertical"],
        base["horizontal"] + direction,
        move_time=200
    )

def move_leg_backward(leg_name: str, direction=80):
    """지지 다리를 뒤로 보내며 몸체를 전진시키는 동작"""
    base = POSTURE["stand"][leg_name]

    set_leg_pose(
        leg_name,
        base["vertical"],
        base["horizontal"] - direction,
        move_time=300
    )

def walk_forward(steps=1):
    """
    대각선 보행 순서
    1. 앞왼쪽 + 뒤오른쪽 이동
    2. 앞오른쪽 + 뒤왼쪽 이동
    """
    for step in range(steps):
        print(f"[WALK] 전진 스텝 {step + 1}/{steps}")

        # 첫 번째 대각선 다리
        move_leg_forward("front_left")
        move_leg_forward("rear_right")

        move_leg_backward("front_right")
        move_leg_backward("rear_left")

        # 두 번째 대각선 다리
        move_leg_forward("front_right")
        move_leg_forward("rear_left")

        move_leg_backward("front_left")
        move_leg_backward("rear_right")

        stand()
```

### 5. 제자리 좌·우 회전 제어

음성 방향이 로봇 정면이 아닐 경우, 4개의 다리 수평 관절을 반대 방향으로 움직여 로봇의 방향을 조절합니다.

```python
def turn_left():
    """로봇을 좌측으로 조금 회전"""
    print("[ROBOT] 좌회전")

    set_leg_pose("front_left",  520, 430, 300)
    set_leg_pose("rear_left",   520, 430, 300)
    set_leg_pose("front_right", 480, 570, 300)
    set_leg_pose("rear_right",  480, 570, 300)

    time.sleep(0.3)
    stand()

def turn_right():
    """로봇을 우측으로 조금 회전"""
    print("[ROBOT] 우회전")

    set_leg_pose("front_left",  520, 570, 300)
    set_leg_pose("rear_left",   520, 570, 300)
    set_leg_pose("front_right", 480, 430, 300)
    set_leg_pose("rear_right",  480, 430, 300)

    time.sleep(0.3)
    stand()
```

### 6. 음성 방향과 보행 제어 통합

음성 인식 결과를 받아 방향을 보정한 뒤, 목표 방향으로 전진합니다.

```python
def move_toward_rescue_voice(angle):
    """구조 요청 음성 방향을 향해 로봇을 이동"""
    command = decide_motion_from_angle(angle)

    if command == "TURN_LEFT":
        turn_left()

    elif command == "TURN_RIGHT":
        turn_right()

    elif command == "FORWARD":
        walk_forward(steps=1)

    else:
        stop()

# 예시: 음원이 오른쪽 60° 방향에서 감지된 경우
move_toward_rescue_voice(angle=60.0)
```

### 7. LiDAR 기반 장애물 감지

2D LiDAR의 정면 거리 데이터를 이용해 로봇 전방의 장애물을 확인합니다.  
정면에 일정 거리 이내의 장애물이 존재하면 보행을 멈추고 회전 또는 경로 재계획을 수행합니다.

```python
SAFE_DISTANCE_M = 0.45

def is_obstacle_ahead(scan_ranges):
    """
    scan_ranges: LiDAR 거리 배열
    정면 범위의 최소 거리값을 기준으로 장애물을 판단한다.
    """
    front_ranges = scan_ranges[350:360] + scan_ranges[0:11]

    valid_ranges = [
        distance for distance in front_ranges
        if distance > 0.05 and distance < 10.0
    ]

    if not valid_ranges:
        return False

    nearest_distance = min(valid_ranges)

    print(f"[LiDAR] 정면 최소 거리: {nearest_distance:.2f} m")

    return nearest_distance < SAFE_DISTANCE_M

def avoid_obstacle(scan_ranges):
    """장애물 존재 시 안전 정지 후 회피 방향 결정"""
    if is_obstacle_ahead(scan_ranges):
        stop()
        print("[LiDAR] 장애물 감지 → 우회전 후 재탐색")
        turn_right()
        return True

    return False
```

### 8. ROS 2 LiDAR·Odometry·SLAM 실행

```bash
# 1. LiDAR 드라이버 실행
ros2 launch sllidar_ros2 sllidar_a1_launch.py \
  serial_port:=/dev/ttyUSB1 \
  serial_baudrate:=115200

# 2. LaserScan 기반 RF2O Odometry 실행
ros2 run rf2o_laser_odometry rf2o_laser_odometry_node \
  --ros-args \
  -p laser_scan_topic:=/scan \
  -p odom_topic:=/odom \
  -p base_frame_id:=base_link \
  -p odom_frame_id:=odom

# 3. SLAM Toolbox로 2D 지도 생성
ros2 launch slam_toolbox online_async_launch.py

# 4. RViz2에서 Map, LaserScan, TF, Odometry 확인
rviz2
```

### 9. 재난 접근 통합 제어 시나리오

```python
def rescue_mission(text, mic1, mic2, mic3, mic4, lidar_ranges):
    """
    1. 구조 요청 음성 탐지
    2. 음성 방향 추정
    3. LiDAR 전방 장애물 확인
    4. 장애물이 없으면 구조 요청 방향으로 이동
    """
    angle = process_voice_result(text, mic1, mic2, mic3, mic4)

    if angle is None:
        print("[MISSION] 구조 요청 음성을 탐지하지 못했습니다.")
        stop()
        return

    if avoid_obstacle(lidar_ranges):
        print("[MISSION] 장애물 회피 후 음성 방향을 재탐색합니다.")
        return

    move_toward_rescue_voice(angle)
```
