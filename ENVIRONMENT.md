# Окружение
### Док описывает настройки и окружение используемой для разработки Jetson Nano 4GB

### Общее
**ОС:** [JetPack 4.6.2 (Ubuntu 18)](https://developer.nvidia.com/embedded/jetpack-sdk-462)

## ROS
Установлен **ROS Melodic**. Источник: [JetsonHacksNano/isntallROS](https://github.com/JetsonHacksNano/installROS)

Запущены оба скрипта:
- installROS.sh
- setupCatkinWorkspace.sh

## Рабочее пространство 
Рабочее пространство Catkin (**Catkin Workspace**) находится по пути:
`/home/jetson/catkin_ws`

Данный репозиторий ([TheMrKan/vio-homing](https://github.com/TheMrKan/vio-homing)) склонирован сюда: `catkin_ws/src/vio-homing`.
В нем предполагается держать весь наш исходный код, конфиги и т. д. 

В `catkin_ws/src/VINS-Fusion` находится форк оригинального проекта от Андрея.


## Другое
- **Ceres Solver v1.12.0** собран, лежит в `/home/jetson/ceres-bin/`




