# RMManager

通过图传链路与电管链路实现与裁判系统的通信。

## 仓库架构

核心package

* [core/rm_message](core/rm_message) 裁判系统对应bag对应的topic定义
* [core/rm_manager](core/rm_manager) 负责与裁判系统通信，并且将裁判系统发来的数据转换为 topic 发送出去，或者接收 topic 消息，将其发送到串口中
* [core/rm_ui](core/rm_ui) 通过 topic 与 rm_manager 通信，通过 service 接受来自其他 node 的ui绘制请求，绘制ui，并且维护ui状态。

功能包

* [core/rm_button_action](core/rm_button_action) 提供一些关于类型为 Bool 的topic的一些支持
* [core/remote_controller](core/remote_controller) 将图传的遥控输入转化为速度与案件状态

应用包：主要是解析自定义控制器数据流

* [core/rm_custom_controller_imu](core/rm_custom_controller_imu) 解算型自定义控制器协议
* [core/rm_custom_controller_state](core/rm_custom_controller_state) 一一映射型自定义控制器协议