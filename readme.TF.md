我们创建TF树，rviz2没有出现模型的原因和解决：

| 现象 | 原因 |
|---|---|
| RobotModel 的 Links 为空 | RobotModel 没有真正收到或接受 robot description，通常是 Description Source、Topic 或 QoS 配置错误 |
| Links 有内容，但 3D 中没有模型 | Fixed Frame 不在 TF 树中，或 TF 不完整 |
| 机械臂显示正常，只有手消失 | 手的 meshes/... 相对路径无法解析 |
| 终端报 mesh 文件不存在 | mesh 加载失败，但不会导致整个 URDF/TF 被丢弃 |

最可能的问题是 RViz 配置里 /robot_description 的订阅 QoS 仍是 Volatile。robot_state_publisher 使用 Transient Local 发布；如果 RViz 在消息发布之后才建立订阅，Volatile 订阅收不到历史消息。ros2 topic echo 只能证明发布端正常，不能证明 RViz 的订阅配置正确。

另外，手部模型原来使用了 44 处 meshes/xxx.STL。它们会相对于 launch / RViz 进程的工作目录解析，而不是相对于 URDF 文件解析，因此手部 mesh 确实无法加载。但这只会让手部 visual 消失，不会导致整个 RobotModel 被解析器丢弃。

修复了工作空间中的以下文件：

- display.launch.py (my_robot_description/launch/display.launch.py:12)：直接展开 xacro，并显式加载 RViz 配置
- display.rviz (my_robot_description/rviz/display.rviz:50)：Topic /robot_description，QoS Transient Local，Fixed Frame base_link
- linkerhand_g20_left.urdf (my_robot_description/urdf/linkerhand_g20_left.urdf:29)：44 个手部 mesh 全部改成 package://
- robot.urdf (my_robot_description/urdf/robot.urdf:241)：同步修复了生成文件中的路径
- CMakeLists.txt (my_robot_description/CMakeLists.txt:24)：安装新增的 rviz 目录
- package.xml (my_robot_description/package.xml:12)：补齐运行时依赖

验证结果：30 个 link、29 个 joint、60 个 mesh，全部 mesh 路径可解析，check_urdf 成功。

## 重新构建并运行

source /opt/ros/humble/setup.bash
cd ~/rm_ws

colcon build --symlink-install --packages-select my_robot_description
source install/setup.bash

ros2 launch my_robot_description display.launch.py

以后每开一个新终端，至少需要执行：

source /opt/ros/humble/setup.bash
source ~/rm_ws/install/setup.bash

完整 launch 文件如下：

import os

import xacro
from ament_index_python.packages import get_package_share_directory
from launch import LaunchDescription
from launch.actions import DeclareLaunchArgument
from launch.conditions import IfCondition
from launch.substitutions import LaunchConfiguration
from launch_ros.actions import Node


def generate_launch_description():
    package_share = get_package_share_directory('my_robot_description')
    xacro_file = os.path.join(package_share, 'urdf', 'rm75_with_l20.urdf.xacro')
    rviz_config = os.path.join(package_share, 'rviz', 'display.rviz')

    # 在启动节点前完成 xacro 展开，配置错误会直接让 launch 报错。
    robot_description = xacro.process_file(xacro_file).toxml()
    description_parameter = {'robot_description': robot_description}
    use_rviz = LaunchConfiguration('use_rviz')

    return LaunchDescription([
        DeclareLaunchArgument(
            'use_rviz',
            default_value='true',
            description='Whether to start RViz',
        ),
        Node(
            package='robot_state_publisher',
            executable='robot_state_publisher',
            name='robot_state_publisher',
            parameters=[description_parameter],
            output='screen',
        ),
        Node(
            package='joint_state_publisher_gui',
            executable='joint_state_publisher_gui',
            name='joint_state_publisher_gui',
            output='screen',
        ),
        Node(
            package='rviz2',
            executable='rviz2',
            name='rviz2',
            arguments=['-d', rviz_config],
            condition=IfCondition(use_rviz),
            output='screen',
        ),
    ])

这里不再读取手工生成的 robot.urdf，而是每次启动都从最新 xacro 生成模型，避免修改了 xacro 却仍在显示旧 URDF。

## RViz 中必须确认的设置

在左侧 Displays 面板中按以下项目检查：

1. Global Options → Fixed Frame 设置为 base_link。

   不要使用 /map，也不要写成 /base_link。当前模型根坐标系是 base_link。
2. 添加 RobotModel Display。
3. 设置：
   - Description Source: Topic
   - Description Topic → Value: /robot_description
   - Reliability Policy: Reliable
   - Durability Policy: Transient Local
   - History Policy: Keep Last
   - Depth: 1
4. 展开 RobotModel：
   - 确保 RobotModel 前面的复选框已勾选
   - 展开 Links
   - 勾选 All Links Enabled
   - 展开后应能看到 base_link、Link1 到 Link7、hand_base_link 等 30 个 link
   - 确认 Visual Enabled 为 true
5. 添加 TF Display：
   - 展开 Frames
   - 勾选 All Enabled
   - 可打开 Show Names 查看各坐标系
6. 如果之前保存过错误配置，建议在 RViz 中删除旧的 RobotModel Display 后重新添加，或者直接用：

rviz2 -d ~/rm_ws/install/my_robot_description/share/my_robot_description/rviz/display.rviz

使用新的 launch 文件时，这些设置已经自动加载。

## 进一步检查 Topic 和 TF

查看实际 QoS：

ros2 topic info -v /robot_description

发布端应该是：

Reliability: RELIABLE
Durability: TRANSIENT_LOCAL

如果 RViz 的订阅显示为 VOLATILE，它就不会在晚启动时获取历史 robot description。

检查关节状态：

ros2 topic echo /joint_states --once

检查机械臂 TF：

ros2 run tf2_ros tf2_echo base_link Link7

检查法兰到手基座的 TF：

ros2 run tf2_ros tf2_echo Link7 hand_base_link

只要 tf2_echo 持续输出变换，说明 Link7 → flange_to_hand_joint → hand_base_link 已经接通。

## 手部 mesh 的 xacro 写法和绝对路径

推荐直接使用 package://，这样不依赖当前工作目录：

<mesh filename="package://my_robot_description/meshes/hand_base_link.STL"/>

如果希望使用 xacro 宏统一管理，可以在 rm75_with_l20.urdf.xacro 的 include 之前定义：

<xacro:property
    name="hand_mesh_prefix"
    value="package://my_robot_description/meshes/" />

<xacro:include
    filename="$(find my_robot_description)/urdf/linkerhand_g20_left.urdf" />

然后在手部 URDF 中写：

<mesh filename="${hand_mesh_prefix}hand_base_link.STL"/>

绝对路径也能工作，例如：

<mesh filename="file:///home/yangyang/rm_ws/install/my_robot_description/share/my_robot_description/meshes/hand_base_link.STL"/>

但不推荐，因为重新安装、移动工作空间或换机器后都会失效。

robot_state_publisher 关于 base_link has inertia 的 KDL 警告不影响本次可视化；只有后续做 IK 或 MoveIt 时，才建议额外增加一个无惯性的 world dummy link。
  Worked for 5m 13s • 11:04 PM
