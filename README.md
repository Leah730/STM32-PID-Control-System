# STM32-PID-Control-System
STM32 based inverted pendulum control system using PID algorithm
基于STM32的倒立摆自动起摆与平衡控制系统
##  项目介绍
本项目基于STM32微控制器，实现倒立摆系统的自动起摆和稳定控制
##  主要研究内容：
-STM32底层外设驱动开发
-PWM电机控制
-编码器数据采集
-PID闭环控制算法
-参数调试与系统优化
##  Hardware
**MCU** 
STM32f103
**Development Environment** 
Keil MDK
**Language**
C Language
##  Control Algorithm
系统采用PID闭环控制算法。

通过实时采集摆杆角度反馈，与目标平衡角度进行比较，计算误差并调整电机输出，实现倒立摆稳定控制。

PID控制参数：

- P（比例）：提高系统相应速度 
- I（积分）：消除稳态误差
- D（微分）：抑制系统震荡，提高稳定性
