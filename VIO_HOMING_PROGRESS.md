# VIO Homing — текущий прогресс

Дата: 28.09.2026

Документ фиксирует текущее состояние стенда и выполненные работы по интеграции Jetson Nano, Pixhawk 6C Mini, PX4, ROS Melodic и VINS-Fusion.

## 1. Текущая аппаратная схема

```text
MacBook
├── Ethernet ─────────────── Jetson Nano
│                            192.168.137.2
│                            eth0
│
└── USB-C ───────────────── Pixhawk 6C Mini
                             │
                             └── TELEM2 / UART
                                  │
                                  └── Jetson J41
                                      /dev/ttyTHS1
```

Подключение TELEM2 к Jetson:

```text
Pixhawk TELEM2          Jetson Nano J41
TX pin 2   ---------->  pin 10 RX
RX pin 3   <----------  pin 8  TX
GND pin 6  -----------  pin 6  GND
```

Питание +5V с TELEM2 на Jetson не используется.

---

## 2. Сеть Mac ↔ Jetson

На Jetson настроен статический IPv4:

```text
Jetson: 192.168.137.2/24
Mac:    192.168.137.1/24
Gateway Jetson: 192.168.137.1
```

SSH работает:

```bash
ssh jetson@192.168.137.2
```

Jetson в интернет для текущей работы не выводится. Необходимые Python-пакеты передавались офлайн с Mac через `scp`.

---

## 3. ROS / VINS-Fusion

На Jetson:

```text
Ubuntu 18.04.6 LTS
JetPack 4.6.2
ROS Melodic
Python 2.7.17 для ROS
Python 3.6.9 дополнительно
```

Catkin workspace:

```text
/home/jetson/catkin_ws
```

Исходники VINS-Fusion:

```text
/home/jetson/catkin_ws/src/VINS-Fusion
```

VINS-Fusion успешно собран. Доступны:

```text
vins_node
loop_fusion_node
global_fusion_node
```

---

## 4. Репозиторий `vio_homing` превращён в ROS-пакет

Репозиторий:

```text
/home/jetson/catkin_ws/src/vio_homing
```

Добавлены:

```text
package.xml
CMakeLists.txt
launch/
scripts/
config/
```

Пакет успешно собирается через:

```bash
cd ~/catkin_ws
catkin_make
```

ROS видит пакет:

```bash
rospack find vio_homing
```

---

## 5. MAVLink Pixhawk ↔ Jetson

Физический UART проверен и работает.

Первоначально использовалась скорость `921600`, но поток содержал большое количество повреждённых MAVLink-пакетов (`BAD_DATA`).

Рабочая стабильная скорость:

```text
115200 baud
```

Jetson:

```text
/dev/ttyTHS1
```

PX4 / TELEM2:

```text
/dev/ttyS3
```

Подтверждена двусторонняя MAVLink-связь:

```text
Pixhawk -> Jetson
Jetson  -> Pixhawk
```

В частности, с Jetson успешно запрашивался и принимался `AUTOPILOT_VERSION`.

---

## 6. Постоянная конфигурация PX4 для companion computer

TELEM2 настроен как отдельный MAVLink-канал companion computer.

Используемая конфигурация:

```text
MAV_1_CONFIG     = TELEM 2
MAV_1_MODE       = Custom
MAV_1_RATE       = 10000
MAV_1_FLOW_CTRL  = 0
SER_TEL2_BAUD    = 115200
MAV_PROTO_VER    = 2
```

Чтобы после загрузки PX4 не вводить вручную команды через QGroundControl, на microSD создан:

```text
/fs/microsd/etc/extras.txt
```

Содержимое:

```sh
set +e

mavlink stream -d /dev/ttyS3 -s HEARTBEAT -r 1
mavlink stream -d /dev/ttyS3 -s HIGHRES_IMU -r 100
mavlink stream -d /dev/ttyS3 -s SYS_STATUS -r 1
mavlink stream -d /dev/ttyS3 -s LOCAL_POSITION_NED -r 10

set -e
```

После перезагрузки Pixhawk поток `HIGHRES_IMU` поднимается автоматически.

Проверенная фактическая частота:

```text
~100 Hz
```

---

## 7. Offline pymavlink

Так как Jetson работает без доступа в интернет, зависимости устанавливались офлайн.

Для Python 3 установлены:

```text
future 1.0.0
pyserial 3.5
pymavlink 2.4.41
```

Для Python 2 / ROS Melodic установлены:

```text
pip 20.3.4
future 0.18.3
pymavlink 2.4.31
```

Проверено совместное использование в Python 2:

```python
import rospy
import serial
import numpy
import future
from pymavlink import mavutil
```

---

## 8. MAVLink → ROS IMU bridge

Создан ROS/Python bridge:

```text
scripts/mavlink_imu_bridge.py
```

Он:

1. открывает `/dev/ttyTHS1` на `115200`;
2. принимает MAVLink от PX4;
3. читает `HIGHRES_IMU`;
4. выполняет преобразование осей MAVLink FRD → ROS FLU;
5. публикует `sensor_msgs/Imu`;
6. выполняет синхронизацию времени Pixhawk ↔ Jetson через MAVLink `TIMESYNC`;
7. принимает ROS odometry и отправляет её обратно в PX4 через MAVLink `ODOMETRY`.

ROS IMU topic:

```text
/pixhawk/imu
```

Преобразование осей IMU:

```text
MAVLink FRD        ROS FLU
X forward    ->    X forward
Y right      ->   -Y left
Z down       ->   -Z up
```

---

## 9. Проверка частоты и timestamp IMU

Проведён тест 1000 сообщений.

Результат:

```text
Samples:        1000
Mean dt:        0.010001 s
Mean rate:      99.99 Hz
Std deviation:  0.002325 s
Min dt:         0.004427 s
Max dt:         0.015055 s
Gaps > 15 ms:   1
Bad timestamps: 0
```

Также `rostopic hz /pixhawk/imu` показывает среднюю частоту около 100 Hz.

Главный результат:

```text
частота ~100 Hz
монотонные timestamps
Bad timestamps = 0
```

---

## 10. TIMESYNC Pixhawk ↔ Jetson

В bridge реализован MAVLink `TIMESYNC`.

Схема:

```text
PX4 clock
   │
   │ TIMESYNC
   ▼
Jetson clock
   │
   ▼
ROS header.stamp
```

Используется медиана последних 30 валидных измерений offset.

Пример наблюдаемого offset:

```text
-1790603484775.xxx ms
```

Большое абсолютное значение ожидаемо: PX4 и Jetson используют разные временные эпохи.

Offset в тесте менялся в пределах нескольких миллисекунд и считался достаточно стабильным для первого стендового VIO-прототипа.

---

## 11. ROS Odometry → MAVLink ODOMETRY → PX4

Реализован обратный канал:

```text
ROS /vio/odom_frd
        │
        ▼
mavlink_imu_bridge.py
        │
        ▼
MAVLink ODOMETRY
        │
        ▼
Pixhawk / PX4
        │
        ▼
vehicle_visual_odometry
```

Для теста создан:

```text
scripts/test_odometry.py
```

Он публикует тестовую odometry с частотой 30 Hz:

```text
position = [1.23, 0.0, 0.0]
orientation = identity quaternion
linear velocity = 0
angular velocity = 0
```

Используемые MAVLink frames:

```text
pose frame:     MAV_FRAME_LOCAL_FRD
velocity frame: MAV_FRAME_BODY_FRD
estimator:      MAV_ESTIMATOR_TYPE_VIO
```

В PX4 через:

```text
listener vehicle_visual_odometry 5
```

успешно получено:

```text
position: [1.23000, 0.00000, 0.00000]
q: [1.00000, 0.00000, 0.00000, 0.00000]
velocity: [0.00000, 0.00000, 0.00000]
angular_velocity: [0.00000, 0.00000, 0.00000]
```

Это подтверждает рабочий транспорт:

```text
ROS -> Jetson -> MAVLink -> PX4
```

External vision пока НЕ включён в EKF2 как источник навигации. На текущем этапе проверялся только приём ODOMETRY в PX4.

Последняя версия bridge также была переработана так, чтобы каждое новое ROS odometry-сообщение отправлялось в MAVLink один раз. Финальную проверку соответствия `ODOM RX ≈ ODOM TX ≈ 30 Hz` после этой правки необходимо выполнить отдельно.

---

## 12. Что уже закрыто

Готово:

- Ethernet Mac ↔ Jetson;
- SSH-доступ к Jetson;
- ROS Melodic;
- собранный VINS-Fusion;
- `vio_homing` оформлен как ROS-пакет;
- физический UART Pixhawk ↔ Jetson;
- стабильный MAVLink на 115200;
- автоматическая настройка MAVLink streams после загрузки PX4;
- `HIGHRES_IMU` ~100 Hz;
- offline pymavlink;
- MAVLink → ROS IMU;
- TIMESYNC PX4 ↔ Jetson;
- ROS IMU timestamps;
- ROS Odometry → MAVLink ODOMETRY;
- приём `vehicle_visual_odometry` на PX4.

---

## 13. Следующие задачи

### 1. Проверить последнюю версию ODOMETRY bridge

Цель:

```text
ODOM RX ≈ 30 Hz
ODOM TX ≈ 30 Hz
TX ERR = 0
```

### 2. Автозапуск companion stack на Jetson

После включения питания Jetson должен автоматически запускать:

```text
ROS master
mavlink_imu_bridge
camera driver
VINS-Fusion
vio_homing
```

### 3. Подключить USB-камеру

После появления камеры:

- определить `/dev/video*`;
- создать ROS image topic;
- проверить частоту кадров;
- выполнить intrinsic calibration.

### 4. Camera ↔ IMU extrinsics

Необходимо определить взаимное положение и ориентацию камеры и IMU/Pixhawk.

### 5. Настроить VINS-Fusion

Входы:

```text
/pixhawk/imu
camera image topic
```

Выход:

```text
VIO pose / odometry
```

### 6. Преобразование VINS coordinates → PX4 FRD

Перед отправкой реальной VINS odometry необходимо явно реализовать преобразование координат и ориентации в систему PX4.

### 7. Подключить external vision к EKF2

Только после проверки VINS output:

```text
VINS
  ↓
ODOMETRY
  ↓
vehicle_visual_odometry
  ↓
EKF2
```

### 8. Реализовать homing state machine

Целевая логика:

```text
NORMAL
  ↓
loss of GNSS / link
  ↓
VIO_SWITCH
  ↓
HOMING
  ↓
last reliable-link point
  ↓
WAIT_LINK >= 60 s
  ↓
LAND
```

---

## 14. Текущий итог

На текущем этапе полностью подтверждена базовая двусторонняя интеграция Jetson ↔ Pixhawk:

```text
Pixhawk IMU
   ↓
HIGHRES_IMU 100 Hz
   ↓
MAVLink / TELEM2
   ↓
Jetson
   ↓
ROS /pixhawk/imu
   ↓
future VINS-Fusion

future VINS odometry
   ↓
ROS /vio/odom_frd
   ↓
MAVLink ODOMETRY
   ↓
PX4 vehicle_visual_odometry
```

То есть транспортный слой и временная синхронизация в основном готовы. Следующий крупный этап — камера, VINS-Fusion и интеграция реальной VIO odometry в PX4 EKF2.
