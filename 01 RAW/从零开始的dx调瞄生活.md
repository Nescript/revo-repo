# 自瞄调试详细流程

#### 调试前准备

确保车的重复度没有问题，拨盘是否卡弹，云台控制是否正常

##### 若拨盘双发：

调urdf中的拨盘offset，若不会调可以使用邪修法，找到rm_description/urdf/对应车名字/shooter.transmission.urdf.xaro，其中有以下部分代码

```
		<joint name="trigger_joint">
            <hardwareInterface>hardware_interface/EffortJointInterface</hardwareInterface>
            <offset>0.55</offset>
        </joint>
```

对offset的值进行调整，每次加0.1，（拨盘拨一发弹需要旋转0.7rad，因此offset的值0.7和0是等价的），直到拨盘每次发一颗弹，连续40-50次不会双发即可

#### imu调试

现在的imu调试，可以使用最新的orientation_controller，其中新增话题发布

```
/imu_assembly_error
```

话题下包括

```
std_msgs/Header header
float64 roll_error
float64 pitch_error
float64 yaw_error
```

将车摆放到比较正的位置，不要进行移动或者旋转，跑控制程序，拉取/imu_assembly_error话题内容，将rpy的值直接加到rm_description/urdf/对应车名字/对应车名字.urdf.xacro中

```
<xacro:IMU connected_to="pitch" imu_name="gimbal_imu"
                       xyz="-0.030 0.0 0.020"
                       rpy="0.08 0.08 0"/>
```

rpy的位置，注意是加上，如上文r原有数值是0.08，若此时roll_error数值为0.01附近，应该将r更改为0.09

因imu数据存在噪声，一般值添加到千分位就比较准确

调整完后打开rviz，选择odom系为参考系，添加tf，只保留odom和bask_link的tf可视化，观察两者的垂直程度，（理想情况两者的x轴和y轴（四根）均在同一平面上）

调整完的情况再重跑一次程序并拉取话题，可以看到error值已经比较小

#### 光心调试

光心的调试理论上来说是相机的装配误差，跑自瞄程序后，对静止靶进行击打，观察子弹落点，找到rm_description/urdf/对应车名字/对应车名字.urdf.xacro,需要修改的地点是

```
<xacro:camera_optical_frame xyz="0.177252 -0.054633 -0.003254"
                                        rpy="-1.55519075 0.027913 -1.5687212"
                                        frame_id="$(arg camera_optical_frame)"
                                        parent="pitch"
                                        camera_sim="false"/>
```

其中的rpy的值是需要修改的

注意是修改第一个camera_optical_frame，第二个camera_optical_frame是用于打符使用的

##### 口诀：上左远，下右近（远近的标准点为0）

若子弹的落点在装甲板的正上方，修改r的值，如上文-1.55，修改为-1.56，即数值更远离0

若在装甲板的正右方，修改y的值，如上文-1.56，修改为-1.55，即数值更接近0

一般0.01的改动大约是半个装甲板的距离，当落点大致在装甲板中心之后，可以对千分位进行微调

距离越远对光心的要求越高

#### 旧火控自瞄参数调试

ps：此部分涉及部分内容可能比较冗杂，可能会写的比较乱

> [!IMPORTANT]
>
> 在进行此项调试前，需要先把云台的pid调好，此处不作pid详细调法的完整展开。
>
> 但需要注意的是，以往的pid调法并不完全适用于高速跟随时候的云台状态，通过传统的云台pid调试方式后的pid数值，其实可以做微小更改，将yaw轴的位置环带宽略微给小，如pid_pos中的p数值为20，可以修改为15或者12(具体数值看高速和低速情况下的跟随效果，最好两者兼顾)
>
> 跟随效果的好坏通过话题中controllers/rm_gimbal_controllers/controllers/pitch(yaw)/pos_state中的set_point和process_value的贴合程度来判断（set_point是期望值，process_value是实际值）
>
> 并且在这里,process_value的实际跟随曲线是traject_set_piont，因此最规范应该是看这两个线的贴合程度

> [!WARNING]
>
> 注意pitch的响应要调整好，子弹的高低会导致落点那以判断（pitch没有高速情况下改小位置环带宽的要求）

> [!TIP]
>
> ##### 该tip内的参数一般不需要更改，此处做粗略介绍或者必要时更改
>
> （此处参数均在rm_config/config/rm_controllers中，后文中若无特殊介绍，参数均在此处）
>
> traject_set_piont同时也涉及到其他的参数，traject_set_point是经过规划之后的云台期望轨迹，其逻辑是只在规划处不同，其他地方和set_point完全一致
>
> 进入规划的条件通过时间预测来判断，此处不做详细展开，具体参数调试方法如下（一般不需要大幅度改动）
>
> traject_start_fit_：进入轨迹规划的时间阈值，单位是ms，具体逻辑是判断未来多少ms内会进行切换装甲板来进行提前规划，一般数值在40左右，一般不需要更改，如果存在traject_set_point进入轨迹规划的时间过于提前（结束了轨迹规划还没有到下一块板），就将该值调小，一般不会低于30或35
>
> yaw_max_acc：轨迹规划路径的最大加速度，此数的直接影响是轨迹规划的整体时长，经过测试，小哨的参数可以在200-250之间，果冻步大约为100出头
>
> traject_k_vel：该项是轨迹规划部分速度的系数，经过测试在0.6附近的效果是最好的，一般不需要更改

##### 其他参数相关介绍

- swtich_angle_offset：跟随模式下切换到下一块装甲板判断阈值的相关参数，该值越大，越早切到下一块装甲					板，一般为0.7左右，几乎不需要更改

准备完上述内容后，可以进行自瞄参数的调整

因为存在ban_shoot(禁止发弹)，allow_shoot(必定发弹，也就是提前枪)，jugde_by_error(根据误差判断是否发弹）参数的影响，若是无法分辨是jugde__by_error还是allow_shoot导致的发弹可以使用以下调试方法

##### 分步调试jugde_by_error和allow_shoot

打开rm_ws/src/rm_controllers/rm_gimbal_controllers/bullet_solver.cpp，找到

```
uint8_t BulletSolver::judgeShootBeforehand(double v_yaw, int id)
{
  if (!track_target_)
    shoot_beforehand_cmd_ = rm_msgs::ShootBeforehandCmd::JUDGE_BY_ERROR;
  else if (std::abs(v_yaw) > config_.min_shoot_beforehand_vel)
  {
    int ban_shoot_delay_number = static_cast<int>(55 + config_.delay * 1000);
    if (id == 6)
    {
      ban_shoot_delay_number = static_cast<int>(55 + config_.outpost_delay * 1000);
    }
    ban_shoot_delay_number = ban_shoot_delay_number > 149 ? 149 : ban_shoot_delay_number;
    if (selected_armor_[ban_shoot_delay_number] != selected_armor_[0])
    {
      ban_shoot_count_++;
      shoot_beforehand_cmd_ = rm_msgs::ShootBeforehandCmd::BAN_SHOOT;
      if (ban_shoot_count_ == 1)
      {
        ban_shoot_start_time_ = ros::Time::now();
      }
    }
    else if (using_traject_ && ros::Time::now().toSec() < ban_shoot_start_time_.toSec() + switchtime)
    {
      shoot_beforehand_cmd_ = rm_msgs::ShootBeforehandCmd::BAN_SHOOT;
    }
    else if (ros::Time::now().toSec() > ban_shoot_start_time_.toSec() + switchtime &&
             ros::Time::now().toSec() < ban_shoot_start_time_.toSec() + switchtime + config_.delay)
    {
      shoot_beforehand_cmd_ = rm_msgs::ShootBeforehandCmd::ALLOW_SHOOT;
      ban_shoot_count_ = 0;
    }
    else
    {
      shoot_beforehand_cmd_ = rm_msgs::ShootBeforehandCmd::JUDGE_BY_ERROR;
      ban_shoot_count_ = 0;
    }
  }
  else
    shoot_beforehand_cmd_ = rm_msgs::ShootBeforehandCmd::JUDGE_BY_ERROR;
  return static_cast<uint8_t>(shoot_beforehand_cmd_);
}
```

该函数内容，将其中的ALLOW_SHOOT修改成BAN_SHOOT，也就是将提前枪禁止掉

> [!NOTE]
>
> ps：这么修改的原因是ALLOW_SHOOT是默认跟随效果好的做的提前发弹，所以先禁止提前枪，将跟随枪（jugde_by_error）调好（但这么调试的前提是delay的参数要有值，该参数的值一般在0.3附近）

修改完对云台代码进行重新编译，并重跑程序

此时可以开启靶车进行调试，距离大约2-3.5米均可，将对面车旋转速度分别控制在6 rad/s, 8 rad/s, 10 rad/s（拉取track话题中的“v_yaw”数值进行判断）进行跟随枪的击打

> [!NOTE]
>
> 10rad为上限是因为果冻步兵的云台跟随效果极限在10.几的旋转速度，无法跟随到11-12转的目标因此果冻步的中心阈值是11rad/s,而小哨的跟随效果较好，可以跟随到12rad/s的目标
>
> 中心模式的进入阈值在rm_config/rm_controllers/ .yaml文件中
>
> ```
> max_track_target_vel + switch_hysteresis //中心模式的进入阈值
> max_track_target_vel - swtich_hysteresis //进入中心模式后中心模式的退出阈值
> ```

用手机拍摄慢动作，最好正对着靶车，观察子弹落点，以装甲板从左往右运动为例子

此时若子弹在装甲板的右侧（预测的时间过长）

将track_rotate_target_delay的值调小，一般调试步长为0.01

若此时子弹在装甲板的左侧（预测的时间过短）

将值调大

- track_rotate_target_delay的值一般在-0.03到0.03左右

进行参数调整，直到命中率可观，击打点较正，保存参数

将刚刚修改为ban_shoot的值重新修改成allow_shoot，再将judge_by_error的值修改成ban_shoot

保存，编译，重跑程序

重新对车进行击打，观察此时的击打效果，以装甲板从左往右运动为例子

此时若是打在装甲板的右侧，就是提前枪太早击打了，将delay的值改小，调试步长为0.005或者更小

一般这种情况不会出现打在装甲板左侧的情况，如果有，就是pid数值不好导致云台超调无法很好跟随

- delay的值一般在0.02到0.03左右



> [!CAUTION]
>
> 之后将judge__by_error重新修改回来，看完整形态下的击打效果和命中率（一般不会有什么问题）
>



#### 中心模式的调试

中心模式的调试较为简单，但因为过于简单，会导致对车体的要求更高，即发弹延迟

> [!TIP]
>
> 中心模式的参数center_delay在理论上就是发弹延迟的数值，在实际调试中也不会出现很大的偏差
>
> 上赛季的发弹延迟大约在0.08左右，这赛季的车似乎拨盘比较卡，均值在0.11左右，具体看具体车辆，也可以现场拉取曲线看发弹延迟（英雄的值会特别大，大约0.3-0.4）
>
> 分别拉取
>
> - controllers/shooter_controllers/command/mode
>
> 的和
>
> - controllers/shooter_controllers/friction/friction_left/left_fricion_wheel/state/process
>
> 分别是电脑下发的命令和摩擦轮的实际转速，不开瞄，观察从发命令到摩擦轮实际掉速需要的时间

观察子弹落点，以装甲板从左往右运动为例子

若打在撞击板右方，则开火时间过于提前，center_delay的值改小，修改步长一般是0.002（细调）到0.01（粗调）



#### 平移靶的调试

此处和旋转跟随靶的调试逻辑一样，靶车平移，看子弹落点

以装甲板从左往右运动为例子

若子弹在装甲板的右方，预测时间过长（将值改小），修改

- track_move_target_delay，步长一般在0.01左右，


- track_move_target_delay一般没有很具体的固定数值，因为这是对平移量的补偿量，可能会因为不同的车导致在参数上有些差异，但最近的调试值在0.03到0.11之间






结束了所有调试之后，保存参数，综合看如陀螺平移靶的命中率等等，对参数微调（具体的调试方式见上文）



#### 其他有关参数介绍

在rm_config/config/rm_manual中还存在

```
wheel_speed_15/wheel_spedd_18/wheel_speed_30,
```

这些决定摩擦论转速，在最正式的调试中，需要调整子弹初速度到23附近来进行调试，该参数的调试步长为5-10，初速度通过主控看

```
track_armor_error_tolerance
```

跟随模式下的error容忍，我一般给在0.07（和上场一致）

```
untrack_armor_error_tolerance
```

中心模式下的容忍，一般给0.08（但最近测试中果冻步的发弹延迟不太稳定，目前果冻步上的值是0.06）

```
max_track_target_vel
```

用于manual中火控的error选择，和rm_config/config/rm_controllers中的一致就可以了

#### 完成了上述部分内容就算调完了一份完整的瞄了（只要过程中不出现任何问题的话）



### 怠惰if线

后文为接入同济mpc接口后的调试方法，更简单粗暴(但此处的平移和陀螺是耦合在一起的，无法对平移量进行单独的补偿，目前还没有解决办法，可能需要通过调试视觉ekf的相关参数？)

首先同样做自瞄前的准备流程，imu，拨盘，光心

```
yaw_offset: 0.0
pitch_offset: 0.0 
decision_speed: 8 
high_speed_delay_time: 0.015 
low_speed_delay_time: 0.015 
```

此处做简略介绍

在光心调完之后，yaw_offset和pitch_offset两个值几乎是不需要更改的

high_speed_delay_time和low_speed_delay_time是等价于我们自己的tarck_rotate_delay_time

通过decision_speed的值进行切换（rad/s）

至于轨迹规划部分，需要根据车的云台响应去对应调mpc的QR矩阵，一般只改Q

除此之外，同济mpc的火控部分的参数只有

```
  auto shoot_offset_ = 2;
  plan.fire =
    std::hypot(
      traj(0, HALF_HORIZON + shoot_offset_) - yaw_solver_->work->x(0, HALF_HORIZON + shoot_offset_),
      traj(2, HALF_HORIZON + shoot_offset_) -
        pitch_solver_->work->x(0, HALF_HORIZON + shoot_offset_)) < fire_thresh_;
  debug_xyza_valid = true;
```

中的shoot_offset，单位是10ms，等价我们的delay参数