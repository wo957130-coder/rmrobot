# 🤖 RM75 机械臂与 L20 灵巧手协同抓取项目实战记录 (ROS 2 Humble)

本项目记录了从零开始，将睿尔曼 RM75-6F 机械臂与灵心巧手 L20-11 左手进行底层驱动打通、三维模型拼接（URDF/TF 树）以及协同抓取控制的完整工程实践。

## 📌 1. 项目软硬件环境
- **操作系统**: Ubuntu 22.04 (双系统)
- **ROS 版本**: ROS 2 Humble
- **机械臂**: 睿尔曼 RM75-6F (7自由度)
- **灵巧手**: 灵心巧手 L20-11 (左手，G20 工业协议)
- **通信接口**: 机械臂 (网线直连 TCP)，灵巧手 (USB转CAN)

---

## 🛠️ 2. 基础环境与依赖安装

```bash
# 更新系统软件源索引
sudo apt update

# 安装 Python 控制 CAN 总线的依赖库
sudo apt install python3-can -y

# 安装用于调试和监听底层 CAN 数据的工具（candump等）
sudo apt install can-utils -y

# 安装 RViz 中用于手动拖动关节的 GUI 插件（若无此包，启动 RViz 会报错找不到包）
sudo apt install ros-humble-joint-state-publisher-gui -y
