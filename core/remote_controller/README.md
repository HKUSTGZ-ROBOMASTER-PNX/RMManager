# remote_controller

把遥控器的摇杆、键盘和鼠标输入转成底盘速度，同时发布底盘、机械臂的使能信号，以及每个按键的状态。也可以接入导航等其他节点发来的速度，再选择当前由谁控制底盘。

节点默认订阅 `rm_manager` 发布的 `/rm_manager/remote_control`，消息类型是 `rm_message/msg/RemoteControl`。键鼠状态也来自这条消息，需要上游节点提供。

## 启动

加载 ROS 2 环境后，在包含本仓库的工作空间根目录编译、启动：

```bash
colcon build --packages-up-to remote_controller
source install/setup.bash
ros2 run remote_controller remote_controller
```

默认先使用摇杆控制。需要 `rm_manager` 或其他节点持续提供遥控消息，才能正常更新速度、按键和使能状态。

自定义配置时，将下面内容保存为 `remote.yaml`：

```yaml
remote_controller:
  ros__parameters:
    remote_controller_topic: "/rm_manager/remote_control"
    cmd_vel_topic: "/mcknum_controller/reference"
    chasis_enable_topic: "/enable_chasis"
    arm_enable_topic: "/enable_arm"

    max_x: 0.5
    max_y: 0.5
    max_z: 0.5
    delta_x: 1.0
    delta_y: 1.0
    delta_z: 2.0

    control_source_switch_keys: ["KEYL"]
    watchdog_enabled: true
    watchdog_timeout: 1.0
```

在保存文件的目录运行：

```bash
ros2 run remote_controller remote_controller --ros-args --params-file remote.yaml
```

上面的 `delta_*` 比默认值小，速度变化会更缓。参数里的 `chasis` 是现有接口的拼写，配置时要照着写。

## 输入和输出

下面按默认节点名 `remote_controller`、没有命名空间的情况列出：

| 方向 | 话题 | 消息类型 | 用途 |
| --- | --- | --- | --- |
| 输入 | `/rm_manager/remote_control` | `rm_message/msg/RemoteControl` | 摇杆、键鼠和开关状态 |
| 输入 | `bridge_topics` 中配置的话题 | `geometry_msgs/msg/TwistStamped` | 外部节点提供的速度，默认没有 |
| 输出 | `/mcknum_controller/reference` | `geometry_msgs/msg/TwistStamped` | 底盘速度 |
| 输出 | `/enable_chasis` | `std_msgs/msg/Bool` | 底盘使能 |
| 输出 | `/enable_arm` | `std_msgs/msg/Bool` | 机械臂使能 |
| 输出 | `/remote_controller/button/<按键名>` | `std_msgs/msg/Bool` | 当前是否按下 |
| 输出 | `/remote_controller/wheel_less_400` | `std_msgs/msg/Bool` | `wheel < 400` |
| 输出 | `/remote_controller/wheel_bigger_1600` | `std_msgs/msg/Bool` | `wheel > 1600` |

速度输出只使用 `linear.x`、`linear.y`、`angular.z`，时间戳取节点当前时间，`frame_id` 为 `base_link`。外部速度也按这个坐标系使用，节点不会做坐标变换。

遥控输入订阅使用 Best Effort QoS。按键和滚轮状态在每次收到遥控消息时发布，即使当前选的是外部速度源也一样。按键名使用大写，支持：

```text
STOP KEYL KEYR KEYB PRESSL PRESSR PRESSMID
W S A D SHIFT CTRL Q E R F G Z X C V B
```

`KEYB` 和键盘的 `B` 是两个不同的输入。按键、滚轮话题和下文的服务都使用节点私有名称，改节点名或命名空间后，路径会跟着变。

## 谁来控制底盘

控制源的顺序固定为：`REMOTE_CHANNEL`（摇杆）、`KEYBOARD_MOUSE`（键鼠）、`bridge_topics` 中按配置顺序列出的各个话题。节点每次启动都从 `REMOTE_CHANNEL` 开始。

默认每按一次 `KEYL` 就切到下一个控制源，到末尾后回到开头。可以通过 `control_source_switch_keys` 配多个切换键，任意一个刚按下都会切换；同一条遥控消息里同时按下多个切换键，只切一次。

### 摇杆

摇杆中值按 `1024` 计算，偏移量除以 `660` 后乘上对应的速度参数：

| 输出 | 输入换算 |
| --- | --- |
| `linear.x` | `(chanal2 - 1024) / 660 × max_x` |
| `linear.y` | `-(chanal3 - 1024) / 660 × max_y` |
| `angular.z` | `-(chanal0 - 1024) / 660 × max_z` |

`chanal1` 不参与底盘速度计算。这里的 `chanal` 也是消息字段的实际拼写。摇杆路径没有额外限幅，输入超出预期范围时，速度也可能超过 `max_*`。

### 键盘和鼠标

默认 `W/S` 前后移动，`A/D` 左右移动，鼠标横向移动控制转向。`SHIFT` 将三个方向的速度都乘以 `1.5`，`CTRL` 乘以 `0.5`；同时按住时优先使用 `SHIFT`。

鼠标转向先按 `mousex × keyboard_mousex_scale` 换算，并限制在 `±max_z`，再乘加速或减速倍率。因此按住 `SHIFT` 时，速度可以超过 `max_*` 的基础值。

`KEYB` 每按一次会切换键鼠速度的暂停状态。暂停时不再发布键鼠速度，也不会主动发零速度；再次按下恢复发布。

### 外部速度

在 `remote.yaml` 的 `ros__parameters` 下加入：

```yaml
    bridge_topics: ["/navigation/cmd_vel", "/manual/cmd_vel"]
    control_source_switch_list: ["REMOTE_CHANNEL", "/navigation/cmd_vel"]
```

这会订阅两个 `TwistStamped` 话题，但只转发当前选中的那个。输入的三个速度分量先限制在对应的 `±max_*` 内，再限制速度变化快慢。其他分量和输入消息头不会透传。

`control_source_switch_list` 只决定按键切换能到达哪些控制源。上例中按键会在摇杆和导航之间循环，服务仍然可以选键鼠或 `/manual/cmd_vel`。列表默认是 `[]`，表示按键可以切到全部控制源。列表的书写顺序不会改变前面所说的固定切换顺序，也不会改变启动时的控制源。

`bridge_topics` 不能含空名称或重复项；`control_source_switch_list` 的每一项必须是已有控制源，也不能重复。外部控制源名称就是 `bridge_topics` 中填写的话题字符串。

### 用服务查询和切换

```bash
# 查看所有控制源
ros2 service call /remote_controller/get_chassis_control_source_list \
  rm_message/srv/GetChassisControlSourceList '{}'

# 查看当前控制源
ros2 service call /remote_controller/get_current_chassis_control_source \
  rm_message/srv/GetCurrentChassisControlSource '{}'

# 切到键鼠控制
ros2 service call /remote_controller/set_current_chassis_control_source \
  rm_message/srv/SetCurrentChassisControlSource '{source: KEYBOARD_MOUSE}'
```

切到外部速度时，将 `source` 换成已配置的话题名，例如 `'/navigation/cmd_vel'`。设置服务返回 `success`、`message` 和 `current_source`；名称不存在时会返回失败，保留原控制源。

切换本身不会发零速度，也不会清空之前的速度记录，下一条被选中输入的消息到来后才更新速度。

## 使能和断流时会怎样

每收到一条遥控消息，都按其中的 `cut` 更新使能：

| `cut` | 底盘使能 | 机械臂使能 |
| --- | --- | --- |
| `1` | `true` | `false` |
| `2` | `true` | `true` |
| 其他值 | `false` | `false` |

使能和速度是分开发送的，节点不会因为使能为 `false` 就停止发布速度。下游控制器需要处理使能信号。`STOP` 在这里作为普通按键发布，不直接负责停车。

断流检测默认开启，每 `100 ms` 检查一次，超时时间默认 `1 s`。超时后会发一次底盘、机械臂的 `false` 和一条零速度，并清零内部速度记录；恢复收到有效输入后，下一次断流才会再次触发。

这里的“有效输入”包括**任何遥控消息**，以及**当前选中外部速度源的消息**。所以切到导航后，如果遥控消息一直在来，即使导航停止发速度，也不会触发这个超时。反过来，选中的外部速度持续到来，也能让断流检测保持正常。

外部速度消息本身不会发布使能为 `true` 的信号，使能仍由遥控消息的 `cut` 决定。超时时也不会把按键状态全部改成松开。

## 常用参数

| 参数 | 默认值 | 作用 |
| --- | --- | --- |
| `remote_controller_topic` | `/rm_manager/remote_control` | 遥控输入 |
| `cmd_vel_topic` | `/mcknum_controller/reference` | 速度输出 |
| `chasis_enable_topic` / `arm_enable_topic` | `/enable_chasis` / `/enable_arm` | 使能输出 |
| `max_x` / `max_y` / `max_z` | 均为 `0.5` | 基础速度；平移单位是 m/s，转动是 rad/s |
| `delta_x` / `delta_y` / `delta_z` | 均为 `1000.0` | 速度变化上限；平移单位是 m/s²，转动是 rad/s²，默认基本不限制 |
| `keyboard_forward_key` / `keyboard_backward_key` | `W` / `S` | 前进、后退键 |
| `keyboard_left_key` / `keyboard_right_key` | `A` / `D` | 左移、右移键 |
| `keyboard_up_key` / `keyboard_down_key` | `SHIFT` / `CTRL` | 加速、减速键，这里的 up/down 不是升降 |
| `keyboard_up_ratio` / `keyboard_down_ratio` | `1.5` / `0.5` | 加速、减速倍率 |
| `keyboard_mousex_scale` | `-0.05` | 鼠标横向输入到转向速度的比例 |
| `control_source_switch_keys` | `["KEYL"]` | 切换控制源的按键列表，不能为空 |
| `control_source_switch_list` | `[]` | 按键可切到的控制源，空表示全部 |
| `bridge_topics` | `[]` | 外部速度输入列表 |
| `watchdog_enabled` / `watchdog_timeout` | `true` / `1.0` | 是否检测断流，以及超时秒数 |

三个速度来源都使用 `delta_*` 限制速度变化，但目标速度回到零时会直接发零，不走减速过程。它不是完整的平滑停车功能。

旧参数 `control_source_switch_key` 仍可设置单个切换键，默认 `KEYL`。如果显式配置了复数形式 `control_source_switch_keys`，则优先使用复数形式。

完整参数及取值范围见 [src/remote_controller_parameters.yaml](src/remote_controller_parameters.yaml)。这个文件用于构建时生成参数代码，运行配置请采用前面的 `ros__parameters` 格式。当前节点只在启动时读取参数，修改配置后需要重启；部分参数虽然允许通过参数服务修改，也不会更新正在使用的配置。

## 简单的按键动作

节点支持按键触发 `Bool` 消息。在 `ros__parameters` 下添加下面四个等长列表，就能让 `R` 刚按下时向 `/tool_enable` 发 `true`：

```yaml
    target_bottons: ["R"]
    on_state: ["ON_REALSED_OFF"]
    on_action_topic: ["/tool_enable"]
    on_action_content: [true]
```

`target_bottons` 和 `ON_REALSED` 都保留了代码中的拼写，配置时不要改成 `buttons` 或 `RELEASED`。四个列表默认均为空，按相同位置组成一条规则，长度不同会启动失败。

| `on_state` | 实际触发时机 |
| --- | --- |
| `ON_PRESSED` | 按住期间，每条遥控消息触发一次 |
| `ON_REALSED` | 松开期间，每条遥控消息触发一次 |
| `ON_REALSED_OFF` | 刚按下，状态从 `0` 变成 `1` |
| `ON_PRESSED_OFF` | 刚松开，状态从 `1` 变成 `0` |
| `ON_TOGGLED` | 每按一次翻转内部开关；开关为开时，每条遥控消息触发一次 |

每次触发都发送配置的固定 `on_action_content`。`ON_TOGGLED` 关闭时只是停止触发，不会自动发 `false`。这里初始按键状态视为松开，第一条遥控消息若已按下，也会算一次按下。

## 没反应时先看哪里

```bash
# 输入有没有持续到达
ros2 topic hz /rm_manager/remote_control

# 节点有没有发速度
ros2 topic echo /mcknum_controller/reference

# 某个按键是否被识别
ros2 topic echo /remote_controller/button/KEYR
```

如果有输入却没有预期速度，先用上面的服务确认当前控制源；键鼠模式下还要检查是否被 `KEYB` 暂停。有速度但设备不动时，再查看使能话题和下游控制器的订阅配置。自定义了话题或节点名时，排查命令里的路径也要对应修改。
