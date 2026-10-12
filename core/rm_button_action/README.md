# rm_button_action

把一个按键的 `Bool` 状态，变成你想发出去的消息。比如：按住时发 `true`、松开时发 `false`，或者每按一次，在几个设定值之间轮换。

一个节点监听一个按键，输入是 `std_msgs/msg/Bool`：`true` 表示按下，`false` 表示松开。输出支持 `Bool`、`Float32` 和 `Float64`，消息内容由参数指定。

## 先跑起来

加载 ROS 2 环境后，在包含本仓库的工作空间根目录编译：

```bash
colcon build --packages-up-to rm_button_action
source install/setup.bash
ros2 launch rm_button_action example.launch.py
```

这个 launch 会加载 [params/example.yaml](params/example.yaml)，监听 `/button_state`。先在另一个已加载环境的终端观察输出：

```bash
ros2 topic echo /action/press_1 std_msgs/msg/Float32
```

再开一个终端，依次模拟松开、按下：

```bash
ros2 topic pub --once /button_state std_msgs/msg/Bool '{data: false}'
ros2 topic pub --once /button_state std_msgs/msg/Bool '{data: true}'
```

这次按下会向 `/action/press_1` 发出 `0.2`。再松开、按下一次，则向 `/action/press_2` 发出 `0.4`；第三次按下又回到 `/action/press_1`。

## 按住和“刚按下”有什么区别

节点有四组动作，可以同时配置：

| 参数前缀 | 什么时候发送 | 可以配几个动作 |
| --- | --- | --- |
| `press_state_action_*` | 每收到一条 `true` | 一个 |
| `release_state_action_*` | 每收到一条 `false` | 一个 |
| `press_edge_action_*` | 从 `false` 变成 `true`，也就是刚按下 | 一个列表，每次只执行其中一项，依次循环 |
| `release_edge_action_*` | 从 `true` 变成 `false`，也就是刚松开 | 一个列表，每次只执行其中一项，依次循环 |

“按住时发送”的频率取决于输入消息的频率，节点没有定时补发。输入停了，输出也会停。

第一次收到消息时，只执行对应的按住或松开动作，并记住当前状态，**不会执行刚按下或刚松开的动作**。因此手工测试时，要先发一次 `false`，再发 `true`。

如果一次输入同时满足状态动作和变化动作，会先执行状态动作。按下列表和松开列表各自计数，互不影响。

## 例子一：按住开启，松开关闭

将下面内容保存为 `button.yaml`。这里根据 `/button_state` 的按键状态，向 `/tool_enable` 发送开启或关闭消息：

```yaml
rm_button_action:
  ros__parameters:
    input_topic: "/button_state"

    press_state_action_topic: "/tool_enable"
    press_state_action_type: "bool"
    press_state_action_bool_value: true

    release_state_action_topic: "/tool_enable"
    release_state_action_type: "bool"
    release_state_action_bool_value: false
```

在保存文件的目录启动：

```bash
ros2 run rm_button_action rm_button_action --ros-args --params-file button.yaml
```

可以用前面的 `ros2 topic pub` 命令模拟按键输入。接入自己的按键话题时，把 `input_topic` 改成实际的话题名即可。

## 例子二：按一下开，再按一下关

下面也是一份完整配置，可以保存为 `button.yaml`，用上面的命令启动：

```yaml
rm_button_action:
  ros__parameters:
    input_topic: "/button_state"

    press_edge_action_topics: ["/tool_enable", "/tool_enable"]
    press_edge_action_types: ["bool", "bool"]
    press_edge_action_bool_values: [true, false]
    press_edge_action_float_values: [0.0, 0.0]
    press_edge_start_index: 0
```

收到初始松开状态后，第一次按下发 `true`，第二次按下发 `false`，之后循环。一直按住不会走到下一项，必须松开后再按。这里没有配置松开动作，所以松开时不发消息。

想循环发送数值，就把 `types` 改成 `float32` 或 `float64`，并在 `float_values` 里填写数值；`bool_values` 仍要保留相同数量的占位值。

## 参数怎么填

`input_topic` 默认是 `/button_state`。其他动作默认都关闭，所以直接运行节点、不传配置时，不会产生动作输出。

状态动作使用以下后缀，分别接在 `press_state_action_` 或 `release_state_action_` 后面：

| 后缀 | 含义 | 默认值 |
| --- | --- | --- |
| `topic` | 输出话题，空字符串表示关闭 | `""` |
| `type` | `none`、`bool`、`float32` 或 `float64`；`none` 表示关闭 | `none` |
| `bool_value` | `bool` 消息的内容 | 按下为 `true`，松开为 `false` |
| `float_value` | `float32` 或 `float64` 消息的内容 | `0.0` |

变化动作使用以下后缀，分别接在 `press_edge_action_` 或 `release_edge_action_` 后面：

| 后缀 | 含义 | 默认值 |
| --- | --- | --- |
| `topics` | 每一项的输出话题 | `[]` |
| `types` | 每一项的消息类型，取值同上 | `[]` |
| `bool_values` | 每一项的布尔值 | `[]` |
| `float_values` | 每一项的数值 | `[]` |

四个列表按位置一一对应，**长度必须相同**，用不到的值也要填占位值，否则节点会启动失败。某项的类型为 `none` 或话题为空时，该次不发消息，但仍会轮到下一项。

`press_edge_start_index` 和 `release_edge_start_index` 决定各自从第几项开始，默认都是 `0`，即第一项。列表非空时，索引必须落在列表范围内。重启节点后，计数也会回到这个起点。

完整参数定义见 [src/rm_button_action.yaml](src/rm_button_action.yaml)。它是构建时用的参数定义文件；自己写运行配置时，请采用上面带 `ros__parameters` 的格式。当前代码只在启动时读取配置，修改后需要重启节点。

## 多个按键和组件加载

要给多个按键配不同动作，就启动多个本节点实例，分别订阅各自的按键话题，并给它们不同的节点名和配置；YAML 顶层节点名也要对应修改。

本节点只负责发消息，实际动作由订阅输出话题的节点完成。它不会检查设备是否执行成功，也不会在输入断开后自动补发松开或关闭消息。

如果使用组件容器，也可以加载组件 `rm_button_action::RmButtonAction`。
