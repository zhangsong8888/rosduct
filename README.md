# rosduct

*ROSduct*, ROS信息的管道。ROSduct充当代理，通过[rosbridge协议]（https://github.com/RobotWebTools/rosbridge-suite/blob/develop/rosbridge-protocol.md）将ROS主题、服务和参数从远程“roscore”公开到本地“roscore”。

假设你的网络中有一个启用了ROS的机器人，你想与它通信，但是你的网络配置不允许直接通信（例如，从Docker容器内部）。使用ROSduct，您可以配置一组主题、服务和参数（操作服务器也是，因为它们在内部实现为主题），以便在本地roscore中公开，以便透明地向机器人发送和接收ROS流量。

TODO: Image explaining it.

# Usage

用主题发布者、订阅者、要访问的服务服务器、要公开的服务服务器和参数填充YAML文件。也是ROSbridge websocket服务器的IP和端口。
```yaml
# ROSbridge websocket server info
rosbridge_ip: 192.168.1.31
rosbridge_port: 9090
# Topics being published in the robot to expose locally
remote_topics: [
                    ['/joint_states', 'sensor_msgs/JointState'], 
                    ['/tf', 'tf2_msgs/TFMessage'],
                    ['/scan', 'sensor_msgs/LaserScan']
                    ]
# Topics being published in the local roscore to expose remotely
local_topics: [
                    ['/test1', 'std_msgs/String'],
                    ['/closest_point', 'sensor_msgs/LaserScan']
                    ]
# Services running in the robot to expose locally
remote_services: [
                    ['/rosout/get_loggers', 'roscpp/GetLoggers']
                    ]
# Services running locally to expose to the robot
local_services: [
                    ['/add_two_ints', 'beginner_tutorials/AddTwoInts']
                    ]
# Parameters to be sync, they will be polled to stay in sync
parameters: ['/robot_description']
parameter_polling_hz: 1
```

**Note**: Don't add to remote or local topics the topic `/rosout`.

# Example usage with Docker
这个工具主要是用来解决Docker容器的问题。如果您运行的Docker容器需要与ROS robot进行双向通信，并且您使用的是Linux，那么您只需在“Docker run”命令中添加“--net host”（运行后）。但是如果你使用的是Mac[这行不通]（https://github.com/docker/for Mac/issues/68）。为了解决这个问题，你可以使用这个软件包。

Just get in your Docker image rosduct:

```bash
mkdir -p ~/rosduct_ws/src
cd ~/rosduct_ws/src
git clone https://github.com/uts-magic-lab/rosduct
cd ..
catkin_make
. devel/setup.bash
```

创建一个launchfile，将其配置为公开所需的主题/服务。例如，对于与“move_base”交互的工具，可以有如下启动文件：
```xml
<launch>
  <node pkg="rosduct" name="rosduct" type="rosduct_main.py" output="screen">
    <rosparam>
    # ROSbridge websocket server info
    rosbridge_ip: YOUR.ROBOT.IP.HERE
    rosbridge_port: 9090
    # Topics being published in the robot to expose locally
    remote_topics: [
                        ['/amcl_pose', 'geometry_msgs/PoseWithCovarianceStamped'], 
                        ['/move_base/feedback', 'move_base_msgs/MoveBaseActionFeedback'],
                        ['/move_base/status', 'actionlib_msgs/GoalStatusArray'],
                        ['/move_base/result', 'move_base_msgs/MoveBaseActionResult'],
                        ]
    # Topics being published in the local roscore to expose remotely
    local_topics: [
                        ['/move_base/goal', 'move_base_msgs/MoveBaseActionGoal'],
                        ['/mover_base/cancel', 'actionlib_msgs/GoalID']
                        ]
    # Services running in the robot to expose locally
    remote_services: [
                        ['/move_base/clear_costmaps', 'std_srvs/Empty']
                        ]
    # Services running locally to expose to the robot
    local_services: []
    # Parameters to be sync, they will be polled to stay in sync
    #parameters: []
    #parameter_polling_hz: 1

    </rosparam>
  </node>
</launch>
```

所以你运行你的Docker镜像暴露端口9090（用于rosbridge通信）`Docker run-p 9090:9090-it你的Docker镜像`并且在运行你的ROS节点之前运行上一个launchfile。

要构建配置，可以执行“rosnode info YOUR_NODE”，并检查发布（`local_topics`）和订阅（`remote_topics`）以及服务（`local_Services`）。要填充远程服务，您需要知道节点调用的服务。
