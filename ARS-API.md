# 登录
+ 无固定IP地址, 使用 DWII 软件自动发现 
+ 账号: _dwirobo_
+ 密码: _dwirobo.com_
*登录不上联系客服, 账号根据您要求的系统可能有所不同*

# 接口


# 安装库
```shell
pip install numpy pymavlink mavproxy
```

## 连接(如果您有 DWIOs 联系客服使用 DWIOs 面板)
+ 使用DWIAssistant工具

+ 修改IP地址为您的机器人IP默认192.168.255.233

![alt text](img/01631bc38928f03f35fb2b013ff4b12.png)

+ 点击配置页， 加载配置

![alt text](img/1753781974680.png)

+ 修改地址为二次开发的上位机地址

![alt text](img/bff74e3c81c62ccd93ecd185366d3d2.png)

+ 保存配置

![alt text](img/72e14933fd7de400c267be1b09dd743.png)

+ 重启机器人或者服务（以下演示重启服务）

![alt text](img/68613ded66c157217f995a8096204d5.png)

# 在上位机运行代码测试

以下代码运行后跳出无人机数据则为正常🎉
在机器上运行使用:  tcp:127.0.0.1:5777
在别的地方运行代码使用: tcp:机器人 ip 地址:5777
使用前要确保 机器人 ip 地址的dwios 面板可以打开, 因为是通过这个面板来转发的

```python
"""
如何通过UDP连接将pymavlink连接到自动驾驶仪的示例
"""
import time
from pymavlink import mavutil

master = mavutil.mavlink_connection('tcp:127.0.0.1:5777')

#确保连接是有效的
master.wait_heartbeat()

# 接收机器人数据消息
while True:
    try:
        print(master.recv_match().to_dict())
    except:
        pass
    time.sleep(0.1)
```
## 解锁电机
```python
"""
Example of how to Arm and Disarm an Autopilot with pymavlink
"""
from pymavlink import mavutil

master = mavutil.mavlink_connection('tcp:127.0.0.1:5777')
master.wait_heartbeat()

# 解锁电机
master.mav.command_long_send(
    master.target_system,
    master.target_component,
    mavutil.mavlink.MAV_CMD_COMPONENT_ARM_DISARM,
    0,
    1, 0, 0, 0, 0, 0, 0)

# 等待直到武装确认(可以手动检查master.motors_armed)
print("Waiting for the vehicle to arm")
master.motors_armed_wait()
print('Armed!')

# 上锁
# master.arducopter_disarm() 或者:
master.mav.command_long_send(
    master.target_system,
    master.target_component,
    mavutil.mavlink.MAV_CMD_COMPONENT_ARM_DISARM,
    0,
    0, 0, 0, 0, 0, 0, 0)

# 等待上锁确认
master.motors_disarmed_wait()
```
## 改变运动模式

```python
"""
如何使用pymavlink更改飞行模式的示例
"""
import sys
from pymavlink import mavutil

# 创建连接
master = mavutil.mavlink_connection('tcp:127.0.0.1:5777')
# 发送命令前等待心跳
master.wait_heartbeat()

# 选择一种模式
mode = 'STABILIZE'

# 检查模式是否可用
if mode not in master.mode_mapping():
    print('Unknown mode : {}'.format(mode))
    print('Try:', list(master.mode_mapping().keys()))
    sys.exit(1)

# 获取模式ID
mode_id = master.mode_mapping()[mode]
# 设置新模式
# master.mav.command_long_send(
#    master.target_system, master.target_component,
#    mavutil.mavlink.MAV_CMD_DO_SET_MODE, 0,
#    0, mode_id, 0, 0, 0, 0, 0) or:
# master.set_mode(mode_id) or:
master.mav.set_mode_send(
    master.target_system,
    mavutil.mavlink.MAV_MODE_FLAG_CUSTOM_MODE_ENABLED,
    mode_id)

while True:
    # 等待ACK命令
    ack_msg = master.recv_match(type='COMMAND_ACK', blocking=True)
    ack_msg = ack_msg.to_dict()

    # 如果确认的命令不是' set_mode '，继续等待
    if ack_msg['command'] != mavutil.mavlink.MAV_CMD_DO_SET_MODE:
        continue

    # 打印ACK结果!
    print(mavutil.mavlink.enums['MAV_RESULT'][ack_msg['result']].description)
    break
```

## 手动控制
```python
"""
发送MANUAL_CONTROL消息到自动驾驶仪的示例
此消息能够完全取代操纵杆输入。
"""
# Import mavutil
from pymavlink import mavutil

# 创建连接
master = mavutil.mavlink_connection('tcp:127.0.0.1:5777')
# 请等待心跳后再发送命令
master.wait_heartbeat()

# 发送一个正的x，负的y，负的z，
# 正旋转，无按钮。
# https://mavlink.io/en/messages/common.html#MANUAL_CONTROL
# 警告:由于一些遗留的解决方案，z将在[0-1000]之间工作。
# 其中0为全反转，500为无输出，1000为全油门。
# X,y和r在[-1000到1000]之间。.
master.mav.manual_control_send(
    master.target_system,
    500,
    -500,
    250,
    500,
    0)

# 激活按钮0(第一个按钮)，3(第四个按钮)和7(第八个按钮)
# 可以在QGC的操纵杆菜单中检查和配置此按钮
buttons = 1 + 1 << 3 + 1 << 7
master.mav.manual_control_send(
    master.target_system,
    0,
    0,
    500, # 500 means neutral throttle
    0,
    buttons)
```

## 相机云台控制

```python
"""
使用 pymavlink 控制摄像头云台的示例
"""

import time
# 导入 mavutil 模块
from pymavlink import mavutil

# 创建连接
master = mavutil.mavlink_connection('tcp:127.0.0.1:5777')
# 在发送命令之前等待一个心跳信号
master.wait_heartbeat()


def look_at(tilt, roll=0, pan=0):
    """
    移动云台到给定位置
    参数:
        tilt (float): 俯仰角度，以百分度为单位（0 表示向前）
        roll (float, 可选): 横滚角度，以百分度为单位（0 表示向前）
        pan  (float, 可选): 偏航角度，以百分度为单位（0 表示向前）
    """
    master.mav.command_long_send(
        master.target_system,
        master.target_component,
        mavutil.mavlink.MAV_CMD_DO_MOUNT_CONTROL,
        1,
        tilt,
        roll,
        pan,
        0, 0, 0,
        mavutil.mavlink.MAV_MOUNT_MODE_MAVLINK_TARGETING)


# 循环上下移动摄像头
while True:
    for angle in range(-50, 50):
        look_at(angle * 100)
        time.sleep(0.1)
    for angle in range(-50, 50):
        look_at(-angle * 100)
        time.sleep(0.1)

```


## 设置 Servo PWM
```python
"""
如何使用pymavlink直接控制Pixhawk伺服输出的示例。
"""
import time
# Import mavutil
from pymavlink import mavutil

def set_servo_pwm(servo_n, microseconds):
    """设置AUX 'servo_n'输出PWM脉冲宽度。
    使用 https://mavlink.io/en/messages/common.html#MAV_CMD_DO_SET_SERVO
    'servo_n' 是要设置的AUX端口(假设端口配置为伺服)。
    'microseconds' 是要设置输出的PWM脉冲宽度。一般在1100到1900微秒之间。
    """
    # master.set_servo(servo_n+8, microseconds) 或者:
    master.mav.command_long_send(
        master.target_system, master.target_component,
        mavutil.mavlink.MAV_CMD_DO_SET_SERVO,
        0,            # 第一次传送这个命令
        servo_n + 8,  # 伺服实例，由8个主输出偏移
        microseconds, # PWM脉宽
        0,0,0,0,0     # 没有使用的参数
    )

# 创建连接
master = mavutil.mavlink_connection('tcp:127.0.0.1:5777')
# 请等待心跳后再发送命令
master.wait_heartbeat()

# 命令servo_1在2秒内以50秒的速度从最小到最大
for us in range(1100, 1900, 50):
    set_servo_pwm(1, us)
    time.sleep(0.125)
```

## 设置目标深度和姿态
```python
# 示例：如何在深度保持模式下使用pymavlink设置目标深度
import time
import math
# 导入 mavutil 模块
from pymavlink import mavutil
# 姿态相关的导入
from pymavlink.quaternion import QuaternionBase

def set_target_depth(depth):
    """ 在深度保持模式下设置目标深度。

    使用 https://mavlink.io/en/messages/common.html#SET_POSITION_TARGET_GLOBAL_INT

    'depth' 技术上是一个高度，所以以负数表示相对水面以下的米数
        -> set_target_depth(-1.5) # 设置目标深度为水面下1.5米。

    """
    master.mav.set_position_target_global_int_send(
        int(1e3 * (time.time() - boot_time)), # 自启动以来的毫秒数
        master.target_system, master.target_component,
        coordinate_frame=mavutil.mavlink.MAV_FRAME_GLOBAL_INT,
        type_mask=( # 除了 Z 位置以外，忽略所有其他数据
            mavutil.mavlink.POSITION_TARGET_TYPEMASK_X_IGNORE |
            mavutil.mavlink.POSITION_TARGET_TYPEMASK_Y_IGNORE |
            # 不要忽略 Z 数据
            mavutil.mavlink.POSITION_TARGET_TYPEMASK_VX_IGNORE |
            mavutil.mavlink.POSITION_TARGET_TYPEMASK_VY_IGNORE |
            mavutil.mavlink.POSITION_TARGET_TYPEMASK_VZ_IGNORE |
            mavutil.mavlink.POSITION_TARGET_TYPEMASK_AX_IGNORE |
            mavutil.mavlink.POSITION_TARGET_TYPEMASK_AY_IGNORE |
            mavutil.mavlink.POSITION_TARGET_TYPEMASK_AZ_IGNORE |
            # 不要设置力
            mavutil.mavlink.POSITION_TARGET_TYPEMASK_FORCE_SET |
            mavutil.mavlink.POSITION_TARGET_TYPEMASK_YAW_IGNORE |
            mavutil.mavlink.POSITION_TARGET_TYPEMASK_YAW_RATE_IGNORE
        ), lat_int=0, lon_int=0, alt=depth, # (x, y WGS84 坐标系 - 不使用), z [米]
        vx=0, vy=0, vz=0, # NED 坐标系中的速度 [米/秒] (不使用)
        afx=0, afy=0, afz=0, yaw=0, yaw_rate=0
        # NED 坐标系中的加速度 [N], 偏航角, 偏航角速度
        # (所有这些功能目前不支持，GCS Mavlink 中忽略)
    )

def set_target_attitude(roll, pitch, yaw):
    """ 在深度保持模式下设置目标姿态。

    'roll'、'pitch' 和 'yaw' 是以度为单位的角度。

    """
    master.mav.set_attitude_target_send(
        int(1e3 * (time.time() - boot_time)), # 自启动以来的毫秒数
        master.target_system, master.target_component,
        # 允许深度保持模式控制油门
        mavutil.mavlink.ATTITUDE_TARGET_TYPEMASK_THROTTLE_IGNORE,
        # -> 姿态四元数 (w, x, y, z | 零旋转是 1, 0, 0, 0)
        QuaternionBase([math.radians(angle) for angle in (roll, pitch, yaw)]),
        0, 0, 0, 0 # 横滚速率、俯仰速率、偏航速率、推力
    )

# 创建连接
master = mavutil.mavlink_connection('tcp:127.0.0.1:5777')
boot_time = time.time()
# 在发送命令之前等待心跳信号
master.wait_heartbeat()

# 配备 ArduSub 自动驾驶仪并等待确认
master.arducopter_arm()
master.motors_armed_wait()

# 设置所需的操作模式
DEPTH_HOLD = 'ALT_HOLD'
DEPTH_HOLD_MODE = master.mode_mapping()[DEPTH_HOLD]
while not master.wait_heartbeat().custom_mode == DEPTH_HOLD_MODE:
    master.set_mode(DEPTH_HOLD)

# 设置深度目标
set_target_depth(-0.5)

# 进行旋转
# (从0度到500度以每秒10度的步长设置目标偏航角，每秒更新一次)
roll_angle = pitch_angle = 0
for yaw_angle in range(0, 500, 10):
    set_target_attitude(roll_angle, pitch_angle, yaw_angle)
    time.sleep(1) # 等待一秒

# 以3倍大的步幅反向旋转
for yaw_angle in range(500, 0, -30):
    set_target_attitude(roll_angle, pitch_angle, yaw_angle)
    time.sleep(1)

# 清理（解除武装）结束
master.arducopter_disarm()
master.motors_disarmed_wait()

```

## 读取所有参数
```python
"""
如何使用pymavlink从自动驾驶仪读取所有参数的示例
"""
# 禁用“Broad exception”警告
# pylint: disable=W0703
import time
import sys

# Import mavutil
from pymavlink import mavutil


# 创建连接
master = mavutil.mavlink_connection('tcp:127.0.0.1:5777')
# 请等待心跳后再发送命令
master.wait_heartbeat()

# 请求所有参数
master.mav.param_request_list_send(
    master.target_system, master.target_component
)
while True:
    time.sleep(0.01)
    try:
        message = master.recv_match(type='PARAM_VALUE', blocking=True).to_dict()
        print('name: %s\tvalue: %d' % (message['param_id'],
                                       message['param_value']))
    except Exception as error:
        print(error)
        sys.exit(0)
```

