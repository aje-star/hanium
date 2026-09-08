# 도마뱀 생체모방 기반 고난도 재난 접근 로봇

> 음성 방향 인식, LiDAR 기반 SLAM, 4족보행 제어를 결합하여  
> 재난 현장에서 구조대상자의 음성 방향을 추정하고 접근하는 생체모방형 수색·구조 로봇

## 프로젝트 개요

본 프로젝트는 게코 도마뱀의 이동 구조에서 영감을 받아 개발하는 4족보행 기반 재난 접근 로봇입니다.

다중 마이크 배열을 통해 구조 요청 음성을 감지하고 음원 방향을 추정하며, 2D LiDAR와 ROS 2 SLAM 기능을 통해 주변 환경을 지도화합니다. 8개의 LX-16A 스마트 서보모터를 이용한 다관절 보행 구조로 재난 현장의 잔해와 협소 공간에 접근하는 것을 목표로 합니다.

## 주요 기능

- 다중 마이크 배열 기반 구조 요청 음성 인식
- GCC-PHAT 기반 음원 방향 추정
- 구조대상자 음성 방향 기반 이동 방향 결정
- LX-16A 서보모터 8개 기반 4족보행 제어
- 2D LiDAR 기반 주변 환경 스캔
- RF2O Laser Odometry 기반 상대 위치 추정
- ROS 2 SLAM Toolbox 기반 2D 지도 생성
- RViz2 기반 LiDAR, TF, Map 시각화

## 하드웨어 구성

| 부품 | 역할 |
|---|---|
| NVIDIA Jetson | 로봇 제어, AI 연산, ROS 2 실행 |
| ReSpeaker XVF3800 | 6채널 음성 입력 및 음원 방향 추정 |
| LX-16A 서보모터 8개 | 4족보행 관절 제어 |
| CH341 USB-Serial Adapter | Jetson과 LX-16A 서보 버스 통신 |
| 2D LiDAR | 거리 데이터 수집 및 환경 인식 |
| 3D 프린팅 구조물 | 4족보행 로봇 프레임 제작 |

## 작품 소개 영상

> 영상 업로드 후 아래 링크를 실제 YouTube 링크로 변경합니다.

[![도마뱀 생체모방 기반 고난도 재난 접근 로봇](썸네일_이미지_URL)](YouTube_영상_URL)

## 핵심 소스코드

### 음성 방향 인식

ReSpeaker XVF3800의 다중 마이크 신호를 분석하여 구조 요청 키워드를 감지하고, GCC-PHAT 기반 TDOA 계산으로 음원 방향을 추정합니다.

```python
TARGET_KEYWORDS = ["살려", "도와", "도움", "구해"]

if any(keyword in text for keyword in TARGET_KEYWORDS):
    vector, angle = calculate_direction_vector(m1, m2, m3, m4)
    print(f"구조자 추정 방향: {angle:.1f}°")
```

### LX-16A 4족보행 모터 제어

```python
LEGS = {
    "front_left":  {"vertical_servo": 1, "horizontal_servo": 2},
    "front_right": {"vertical_servo": 3, "horizontal_servo": 4},
    "rear_left":   {"vertical_servo": 5, "horizontal_servo": 6},
    "rear_right":  {"vertical_servo": 7, "horizontal_servo": 8},
}
```

### ROS 2 LiDAR 및 SLAM 실행

```bash
# LiDAR 실행
ros2 launch sllidar_ros2 sllidar_a1_launch.py \
  serial_port:=/dev/ttyUSB1 \
  serial_baudrate:=115200

# SLAM 실행
ros2 launch slam_toolbox online_async_launch.py

# RViz2 실행
rviz2
```

## 팀원 소개

| 이름 | 역할 | 담당 업무 |
|---|---|---|
| 팀원 1 | 팀장 | 프로젝트 총괄 및 하드웨어 제작 |
| 팀원 2 | AI 개발 | 음성 인식 및 음원 방향 추정 |
| 팀원 3 | 로봇 제어 | LX-16A 모터 제어 및 보행 알고리즘 |
| 팀원 4 | SLAM | LiDAR, ROS 2, 지도 생성 및 시스템 통합 |

> 실제 팀원 이름과 역할로 수정 예정입니다.

## 향후 발전 방향

- 열감지 센서 기반 인체 및 화재 열원 탐지
- 풍속·풍향 센서 기반 위험 환경 분석
- 음원 방향 추정과 자율 경로 계획 통합
- 재난 현장 정보의 무선 전송
- 도마뱀 발바닥 구조를 참고한 경사면 및 벽면 이동 기능 고도화
