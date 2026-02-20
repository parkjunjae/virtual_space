# sem_ws 기준 nvblox + Docker 운영 README

기준:
- 보드: Jetson (aarch64)
- ROS: Humble
- 워크스페이스: `~/sem_ws`
- 목표: RealSense 1대로 `nvblox` TSDF/mesh 3D 맵 생성 (초기: VSLAM 제외)

## 1) 전체 구조를 먼저 이해하기

### 1-1. 레이어 구조
1. 호스트 OS(우분투): 실제 파일이 있는 곳 (`~/sem_ws`)
2. Docker 이미지: 실행환경 템플릿
3. Docker 컨테이너: 이미지로 실행된 실제 셸
4. ROS 컴포넌트 컨테이너(`nvblox_container`): ROS composable node들을 담는 프로세스

핵심:
- `run_dev.sh`가 만드는 것은 3번(Docker 컨테이너)
- launch의 `run_standalone:=True`가 만드는 것은 4번(ROS 컴포넌트 컨테이너)

### 1-2. 경로가 다른 이유
- 호스트: `~/sem_ws`
- Docker 내부: `/workspaces/isaac_ros-dev`
- 같은 폴더를 마운트해서 이름만 다르게 보임

## 2) 명령어 의미 사전

### 2-1. 시스템 준비
```bash
sudo apt install -y docker.io
```
- Docker 엔진 설치

```bash
sudo usermod -aG docker $USER && newgrp docker
```
- 현재 유저를 Docker 그룹에 추가하고 즉시 반영

```bash
sudo apt install -y git-lfs && git lfs install
```
- `run_dev.sh`가 요구하는 Git LFS 설치

### 2-2. Isaac ROS 관련
```bash
./scripts/run_dev.sh -d ~/sem_ws
```
- `~/sem_ws`를 마운트해서 dev 컨테이너 실행
- 이미 실행 중이면 새로 만들지 않고 attach

```bash
./scripts/run_dev.sh -d ~/sem_ws -b
```
- `-b`는 이미지 빌드 스킵
- 이미지가 이미 있으면 빠르게 컨테이너 진입

### 2-3. 빌드/런타임 관련
```bash
sudo apt install -y docker-buildx
```
- BuildKit이 요구하는 buildx 플러그인 설치

```bash
sudo apt install -y nvidia-container-toolkit && sudo nvidia-ctk runtime configure --runtime=docker && sudo systemctl restart docker
```
- Docker에서 `--runtime nvidia` 사용 가능하게 설정

```bash
sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml && nvidia-ctk cdi list
```
- CDI 디바이스 파일 생성/확인
- `nvidia.com/gpu=all` 같은 CDI 장치 해석 실패 문제 대응

## 3) 지금까지 나온 에러와 정확한 의미

### A. `git: 'lfs' is not a git command`
- 의미: `git-lfs` 미설치

### B. `no space left on device`
- 의미: 루트 파티션 용량 부족 (Docker 레이어 저장 중 실패)

### C. `BuildKit is enabled but the buildx component is missing`
- 의미: BuildKit ON + buildx 플러그인 없음

### D. `unknown or invalid runtime name: nvidia`
- 의미: Docker에 NVIDIA runtime 미등록

### E. `unresolvable CDI devices nvidia.com/gpu=all`
- 의미: NVIDIA CDI 설정 누락/불일치

### F. `libisaac_ros_nitros_image_type.so: cannot open shared object file`
- 의미: `isaac_ros_nitros` 계열 패키지가 워크스페이스에 없거나 빌드 안 됨

### G. `file 'sensors/realsense.launch.py' was not found ...`
- 의미: package 방식에서 하위 폴더 launch 파일 해석 실패
- 해결: 절대경로 launch 파일로 실행

## 4) sem_ws 재구축 절차 (처음부터)

### 4-1. 워크스페이스/소스
```bash
mkdir -p ~/sem_ws/src && cd ~/sem_ws/src && git clone -b release-3.2 https://github.com/NVIDIA-ISAAC-ROS/isaac_ros_common.git && git clone --recursive -b release-3.2 https://github.com/NVIDIA-ISAAC-ROS/isaac_ros_nvblox.git && git clone -b release-3.2 https://github.com/NVIDIA-ISAAC-ROS/isaac_ros_visual_slam.git && git clone -b release-3.2 https://github.com/NVIDIA-ISAAC-ROS/isaac_ros_nitros.git
```

### 4-2. isaac_ros_common 설정
```bash
cd ~/sem_ws/src/isaac_ros_common/scripts && printf "CONFIG_IMAGE_KEY=ros2_humble.realsense\n" > .isaac_ros_common-config
```

### 4-3. dev 컨테이너 실행
```bash
cd ~/sem_ws/src/isaac_ros_common && ./scripts/run_dev.sh -d ~/sem_ws
```

### 4-4. 컨테이너 내부 빌드
```bash
source /opt/ros/humble/setup.bash && cd /workspaces/isaac_ros-dev && rosdep update && rosdep install -i -r --from-paths src --rosdistro humble -y && colcon build --symlink-install --packages-up-to nvblox_examples_bringup && source /workspaces/isaac_ros-dev/install/setup.bash
```

위 명령을 컨테이너 내부에서 실행하는 이유:
- 컨테이너는 Isaac ROS가 기대하는 CUDA/TensorRT/ROS 조합이 고정된 환경
- 호스트에서 직접 빌드하면 라이브러리 버전 충돌 가능성이 큼

중요:
- 새 터미널(새 컨테이너 셸)마다 `source`를 다시 해야 함
- `source /opt/ros/humble/setup.bash` + `source /workspaces/isaac_ros-dev/install/setup.bash` 둘 다 필요


### 4-5. nvblox 실행 전 라이브러리 경로 설정(필수)
아래 3줄은 `libgxf_isaac_optimizer.so` 로드 실패를 막기 위해 `nvblox` 실행 터미널에서 먼저 실행:

```bash
source /opt/ros/humble/setup.bash
source /workspaces/isaac_ros-dev/install/setup.bash
export LD_LIBRARY_PATH="$(find /workspaces/isaac_ros-dev/install -type d -path '*/share/*/gxf/lib' | tr '\n' ':')${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}"
```

## 5) VSLAM 제외 실행 순서 (4터미널, relay 분리)

각 터미널 공통 준비:
```bash
cd ~/sem_ws/src/isaac_ros_common && ./scripts/run_dev.sh -d ~/sem_ws -b && source /opt/ros/humble/setup.bash && source /workspaces/isaac_ros-dev/install/setup.bash
```

터미널 A (카메라 단독):
```bash
ros2 launch realsense2_camera rs_launch.py camera_name:=camera0 enable_color:=true enable_depth:=true enable_infra1:=false enable_infra2:=false enable_gyro:=false enable_accel:=false pointcloud.enable:=false depth_module.profile:=640x480x15 rgb_camera.profile:=640x480x15 align_depth.enable:=true enable_sync:=true initial_reset:=true
```

터미널 B (relay만 실행):
```bash
# 없으면 1회 설치
# sudo apt update && sudo apt install -y ros-humble-topic-tools
ros2 run topic_tools relay /camera0/depth/image_rect_raw /camera0/realsense_splitter_node/output/depth
```

터미널 C 실행 전(같은 터미널에서 1회):
```bash
source /opt/ros/humble/setup.bash
source /workspaces/isaac_ros-dev/install/setup.bash
export LD_LIBRARY_PATH="$(find /workspaces/isaac_ros-dev/install -type d -path '*/share/*/gxf/lib' | tr '\n' ':')${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}"
```

터미널 C (nvblox):
```bash
ros2 launch /workspaces/isaac_ros-dev/src/isaac_ros_nvblox/nvblox_examples/nvblox_examples_bringup/launch/perception/nvblox.launch.py camera:=realsense mode:=static num_cameras:=1 container_name:=nvblox_container run_standalone:=True
```

터미널 D (RViz):
```bash
ros2 launch /workspaces/isaac_ros-dev/src/isaac_ros_nvblox/nvblox_examples/nvblox_examples_bringup/launch/visualization/rviz.launch.py camera:=realsense mode:=static
```

## 6) 필수 조건
- 카메라 드라이버 중복 실행 금지
- TF 필수
  - `odom -> base_link` (동적)
  - `base_link -> camera0_link` (정적)

## 7) 점검 명령
```bash
ros2 topic hz /camera0/depth/image_rect_raw
```

```bash
ros2 topic hz /nvblox_node/mesh
```

```bash
ros2 run tf2_ros tf2_echo odom camera0_link
```

## 8) 운영 중 자주 쓰는 복구

패키지 DB 복구:
```bash
sudo dpkg --configure -a && sudo apt --fix-broken install -y
```

디스크 정리:
```bash
rm -rf ~/.local/share/Trash/files/* ~/.local/share/Trash/info/* && pip cache purge 2>/dev/null || true && sudo apt clean && docker system prune -f
```

NITROS 누락 복구(한 줄):
```bash
cd /workspaces/isaac_ros-dev/src && [ -d isaac_ros_nitros ] || git clone -b release-3.2 https://github.com/NVIDIA-ISAAC-ROS/isaac_ros_nitros.git && cd /workspaces/isaac_ros-dev && source /opt/ros/humble/setup.bash && rosdep update && rosdep install -i -r --from-paths src --rosdistro humble -y && colcon build --symlink-install --packages-up-to nvblox_examples_bringup && source /workspaces/isaac_ros-dev/install/setup.bash
```

Docker 권한 확인:
```bash
id -nG | tr ' ' '\n' | grep '^docker$' && docker ps
```

컨테이너 셸 종료:
- `exit` 또는 `Ctrl+D`


## 9) nvblox 원리 (PT용 쉬운 설명)

한 줄 요약:
- `nvblox`는 "깊이영상 + 카메라 자세(TF)"를 시간에 따라 누적해서 3D 공간을 voxel로 재구성하는 실시간 맵 엔진.

원리 핵심 3가지:
1. `TSDF` (표면 복원)
- 각 voxel에 "표면까지의 signed distance"를 저장/누적 평균.
- 여러 프레임이 쌓일수록 노이즈가 줄고 표면이 매끄러워짐.
2. `ESDF` (거리장)
- TSDF에서 "장애물까지의 최단거리"를 계산.
- 경로계획/충돌회피에서 바로 사용 가능.
3. `Mesh` (시각화)
- TSDF의 0-crossing(표면 경계)에서 삼각형 mesh를 생성.
- RViz에서 실제 구조처럼 보이는 3D 표면 확인 가능.

동작 흐름(발표용):
1. 카메라가 `depth image`를 발행
2. TF가 해당 프레임 시점의 카메라 위치/자세를 제공
3. nvblox가 depth를 3D ray로 back-project
4. ray를 voxel 격자에 적분해 TSDF 갱신
5. 주기적으로 ESDF/mesh를 업데이트하고 토픽으로 출력

왜 TF가 핵심인가:
- 같은 물체라도 프레임마다 카메라 위치가 달라 좌표변환이 필수
- TF가 없으면 서로 다른 프레임을 같은 공간으로 정렬할 수 없어 누적이 실패
- 그래서 `Lookup transform failed ...`가 뜨면 맵이 멈춤

현재 구성에서 각 노드 역할:
- `realsense2_camera`: 깊이/컬러 원본 공급
- `topic_tools relay`: nvblox가 기대하는 depth 토픽 이름으로 맞춤
- `nvblox_node`: TSDF 적분 + ESDF + mesh 생성
- `rviz`: 결과 확인

PT에서 강조 포인트:
- 입력 품질: depth 해상도/FPS/노이즈
- 위치 품질: TF 품질(`odom -> base_link -> camera0_link`)
- 연산 품질: voxel size/업데이트 주기(정밀도 vs 속도)
