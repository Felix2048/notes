<!--
 * @Description: In User Settings Edit
 * @Author: your name
 * @Date: 2019-10-18 17:48:30
 * @LastEditTime: 2019-10-18 17:48:30
 * @LastEditors: your name
 -->


# The Dynamic Window Approach to Collision Avoidance

[TOC]

## Introduction

[This is a 2D navigation sample code with Dynamic Window Approach.](https://github.com/AtsushiSakai/PythonRobotics/blob/master/PathPlanning/DynamicWindowApproach/dynamic_window_approach.py)

![](../misc/1.dynamic_window_approach_animation.gif)

动态窗口方法(Dynamic window Approach，DWA)是一种常用的避障规划方法。

DWA是一种选择速度的方法。DWA结合了机器人的动力学特性，通过在速度空间 $(v, w) $ 中采样多组速度，并对该速度空间进行缩减，模拟机器人在这些速度下一小段时间间隔内的运动轨迹，要求：

- 机器人避开可能发生碰撞的每一个障碍物

- 机器人在该时间间隔内可以达到这一速度（受限于机器人的动态约束，i.e. 加速度）

- 机器人可以快速到达目标点

之后通过一个评价函数：

- 生成轨迹与参考路径的距离(贴合程度)
- 生成轨迹与参考路径终点的距离
- 生成轨迹上是否存在障碍物(若有则抛弃这条轨迹)

对这些轨迹评价，在速度空间中搜索最优轨迹所对应的机器人最优控制速度，并通过这一速度来驱动机器人运动。

### 优点

- 反应速度较快,计算不复杂,通过速度组合(线速度与角速度)可以快速得出下一时刻规划轨迹的最优解.
- 可以将优化由横向与纵向两个维度向一个维度优化

### 缺点

- 此算法是由机器人避障衍生而来,机器人的路径规划算法主要体现在较高的灵活性和其静态环境,譬如某些特种车辆的工作环境,是一个较大的开阔广场,因此是比较适用此算法的
- 无人驾驶则要求较高的稳定性和动态环境,主要是结合车道线信息来实现规划算法
- 较高的灵活性会极大的降低行驶的平稳性
- 可能是避障算法载体由机器人向无人车平台移植过程中会产生的主要问题



## Motion Model

### 模型1：直线轨迹模型

简单，程序中最常见。

#### 机器人不是全向移动的

即不能纵向移动，只能前进和旋转$\left(v_{t}, w_{t}\right)$。

计算机器人轨迹时，先考虑两个相邻时刻，如下图所示。

为简单起见，由于机器人相邻时刻$\Delta t$内，运动距离短，因此可以忽略加速度当作匀速直线运动处理，将两相邻点之间的运动轨迹看成直线。

![1561118635251](../misc/1.1561118635251.png)

则只需将该段距离分别投影在世界坐标系x轴和y轴上得到在世界坐标系中坐标移动的位移：
$$
\begin{array}{l}{\Delta x=v \Delta t \cos \left(\theta_{t}\right)} \\ {\Delta y=v \Delta t \sin \left(\theta_{t}\right) \\ {\theta_{t}=\theta_{t}+w \Delta t}}\end{array}
$$
以此类推，推算一段时间内的轨迹，只需要将这段时间的位移增量累计求和：
$$
\begin{array}{l}{x=x+v \Delta t \cos \left(\theta_{t}\right)} \\ {y=y+v \Delta t \sin \left(\theta_{t}\right)}\end{array}
$$

#### 机器人是全向运动的

即机器人有$y$轴速度，则同理，只需将机器人在机器人坐标y轴移动的距离投影到世界坐标系即可得：
$$
\begin{aligned} \Delta x &=v_{y} \Delta t \cos \left(\theta_{t}+\frac{\pi}{2}\right)=-v_{y} \Delta t \sin \left(\theta_{t}\right) \\ \Delta y &=v_{y} \Delta t \sin \left(\theta_{t}+\frac{\pi}{2}\right)=v_{y} \Delta t \cos \left(\theta_{t}\right) 
\\ \theta_{t} &=\theta_{t}+w \Delta t \end{aligned}
$$
推算一段时间内的轨迹，则对之前的公式进行修改得：
$$
\begin{array}{l}{x=x+v \Delta t \cos \left(\theta_{t}\right)-v_{y} \Delta t \sin \left(\theta_{t}\right)} \\ {y=y+v \Delta t \sin \left(\theta_{t}\right)+v_{y} \Delta t \cos \left(\theta_{t}\right)}\end{array}
$$
在ROS的轨迹推演中就使用的该公式，base_local_planner的轨迹采样程序也是使用的该公式（line 256 in simple_trajectory_generator.cpp）。



### 模型2：圆弧轨迹模型（DWA论文中的运动模型）

上面的计算中，假设相邻时间段内机器人的轨迹是直线，这是不准确的，更准确的做法是用圆弧来代替。

假设机器人不是全向运动的，且机器人的平移和旋转速度可以独立控制（扭矩有限）。通过推导出近似，将速度建模为时间上的分段常数函数，则机器人轨迹由有限多个圆圈段的序列组成。 这种表示对于碰撞检查非常方便，因为障碍物与圆圈的交叉点易于检查。

#### General Motion Equations

作出如下定义：

- $t$: 某一时刻$t$
- $\hat{t}$: 从$t$到$t^{\prime}$的时间间隔$\hat{t} \in [t, t^{\prime}]$
- $x(t)$: 时刻$t$下，机器人的$x$坐标
- $y(t)$: 时刻$t$下，机器人的$y$坐标
- $\theta(t)$: 时刻$t$下，机器人的朝向（orientation / heading direction）$\theta$
- $<x,y,\theta>$: 该三元组描述机器人的运动学配置（kinematic configuration）
- $v(t)$: 时刻$t$下，机器人的平移速度（translational velocity），其方向与机器人的朝向$\theta$相同
- $\omega(t)$: 时刻$t$下，机器人的旋转速度（rotational velocity）
- $\dot{v}(t)$: 时刻$t$下，机器人的平移加速度
- $\dot{\omega}(t)$: 时刻$t$下，机器人的旋转加速度

则有：
$$
\begin{array}{c}{x\left(t_{n}\right)=x\left(t_{0}\right)+\int_{t_{0}}^{t_{n}} v(t) \cdot \cos \theta(t) d t} \\ {y\left(t_{n}\right)=y\left(t_{0}\right)+\int_{t_{0}}^{t_{n}} v(t) \cdot \sin \theta(t) d t}\end{array}
$$
引入加速度和初始速度，则有（$y(t_n)$同理可得）：
$$
\begin{array}{l}{x\left(t_{n}\right)=x\left(t_{0}\right)+\int_{t_{0}}^{t_{n}}\left(v\left(t_{0}\right)+\int_{t_{0}}^{t} \dot{v}(\hat{t}) d \hat{t}\right)} \\ {\cdot \cos \left(\theta\left(t_{0}\right)+\int_{t_{0}}^{t}\left(\omega\left(t_{0}\right)+\int_{t_{0}}^{\hat{t}} \dot{\omega}(\tilde{t}) d \tilde{t}\right) d \tilde{t}\right) d t}\end{array}
$$
由于机器人只能由有限多个加速度命令控制，作出如下定义：

- $n$: 这一时间间隔$\hat{t}$内的时间刻度（time ticks）的总数，即有$t_i \in [t_1, t_n]$
- $\dot{v}_i$: 这一时间间隔$\hat{t}$内的某一时间$t_i$下，机器人的平移加速度，是常数
- $\dot{\omega}_i$: 这一时间间隔$\hat{t}$内的某一时间$t_i$下，机器人的选装加速度，是常数
- $\Delta t_t^l$: 即$\Delta t_{t}^{l}=t-t_{i}$，其中$i = 1, 2, ..., n$

则有：
$$
\begin{array}{l}{x\left(t_{n}\right)=x\left(t_{0}\right)+\sum_{i=0}^{n-1} \int_{t_{i}}^{t_{i+1}}\left(v\left(t_{i}\right)+\dot{v}_{i} \cdot \Delta_{t}^{i}\right)} \\ {\cos \left(\theta\left(t_{i}\right)+\omega\left(t_{i}\right) \cdot \Delta_{t}^{i}+\frac{1}{2} \dot{\omega}_{i} \cdot\left(\Delta_{t}^{i}\right)^{2}\right) d t}\end{array}
$$


#### Approximate Motion Equations

通过将时间间隔内的机器人速度近似为常数值来进行简化上式。

在这个假设下看到的那样，机器人的轨迹可以通过分段圆弧（和直线弧）来近似。

由于机器人在时间间隔$[t_i, t_{i+1}]$内的运动是平稳的，上式中$v\left(t_{i}\right)+\dot{v}_{i} \cdot \Delta_{t}^{i}$项可以由$v_{i} \in\left[v\left(t_{i}\right), v\left(t_{i+1}\right)\right]$进行近似。同理，$\theta\left(t_{i}\right)+\omega\left(t_{i}\right) \cdot \Delta_{t}^{i}+\frac{1}{2} \dot{\omega}_{i} \cdot\left(\Delta_{t}^{i}\right)^{2}$则可以由$\theta\left(t_{i}\right)+\omega_{i} \cdot \Delta_{t}^{i}$进行近似，其中$\omega_{i} \in\left[\omega\left(t_{i}\right), \omega\left(t_{i+1}\right)\right]$

则有：
$$
\begin{aligned}
x\left(t_{n}\right) 
&=x\left(t_{0}\right)+\sum_{i=0}^{n-1} \int_{t_{i}}^{t_{i+1}} v_{i} {\cos \left(\theta\left(t_{i}\right)+\omega_{i} \cdot\left(\hat{t}-t_{i}\right)\right) d \hat{t}} \\ 
&=x\left(t_{0}\right)+\sum_{i=0}^{n-1}\left(F_{x}^{i}\left(t_{i+1}\right)\right)
\end{aligned}
$$
其中：
$$
\begin{array}{l}{F_{x}^{i}(t)=} \\
\end{array}
\left\{\begin{array}{c}{\frac{v_{i}}{\omega_{i}}\left(\sin \theta\left(t_{i}\right)-\sin \left(\theta\left(t_{i}\right)+\omega_{i} \cdot\left(t-t_{i}\right)\right)\right), \omega_{i} \neq 0} \\ {v_{i} \cos \left(\theta\left(t_{i}\right)\right) \cdot t, \omega_{i}=0}\end{array}\right.
$$
同理可得：
$$
y\left(t_{n}\right)=y\left(t_{0}\right)+\sum_{i=0}^{n-1}\left(F_{y}^{i}\left(t_{i+1}\right)\right)
$$
其中：
$$
F_{y}^{i}(t)=\left\{\begin{aligned}-\frac{v_{i}}{\omega_{i}}\left(\cos \theta\left(t_{i}\right)-\cos \left(\theta\left(t_{i}\right)+\omega_{i} \cdot\left(t-t_{i}\right)\right)\right) & w_{i} \neq 0 \\ v_{i} \sin \left(\theta\left(t_{i}\right)\right) \cdot t, \omega_{i} &=0 \end{aligned}\right.
$$

当 $\omega_{i}=0$，机器人将沿直线运动。

当 $\omega_{i}\neq0$，机器人的运动轨迹可被描述为一个圆，满足：
$$
\begin{array}{l}{M_{x}^{i}=-\frac{v_{i}}{\omega_{i}} \cdot \sin \theta\left(t_{i}\right)} \\ {M_{y}^{i}=\frac{v_{i}}{\omega_{i}} \cdot \cos \theta\left(t_{i}\right)}\end{array} \\
\left(F_{x}^{i}-M_{x}^{i}\right)^{2}+\left(F_{y}^{i}-M_{y}^{i}\right)^{2}=\left(\frac{v_{i}}{\omega_{i}}\right)^{2}
$$
即机器人的第$i$个轨迹是一个以$M^i(M_x^i, M_y^i)$为圆心的圆，其半径为：$M_{r}^{i}=\frac{v_{i}}{\omega_{i}}$


### 模型3：自行车模型

作出以下假设：

- 车辆在垂直方向的运动被忽略掉了，也就是说我们描述的车辆是一个二维平面上的运动物体（可以等价与我们是站在天空中的俯视视角）
- 我们假设车辆的结构就像自行车一样，也就是说车辆的前面两个轮胎拥有一直的角度和转速等，同样后面的两个轮胎也是如此，那么前后的轮胎就可以各用一个轮胎来描述
- 我们假设车辆运动也和自行车一样，这意味着是前面的轮胎控制这车辆的转角

![pic](../misc/1.watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MDg4NDU3MA==,size_1,color_FFFFFF,t_70.jpg)

自行车运动学模型将前后轮胎分别用一个轮胎来描述并且将轮胎置于前后中心线上。假定车轮没有横向漂移且只有前向车轮是可以转向的。

由于限制该模型在平面上运动，前后轮的非完整约束方程为：
$$
\begin{array}{c}{\dot{x}_{f} \sin (\theta+\delta)-\dot{y} \cos (\theta+\delta)=0} \\ {\dot{x} \sin (\theta)-\dot{y} \cos (\theta)=0}\end{array}
$$
其中：

- $(x,y)$: 后轮的全局坐标
- $(x_f,y_f)$: 前轮的全局坐标
- $v$: 车辆的纵向速度
- $\theta$: 车辆在$yaw$方向的偏转角度
- $\delta$: 车辆的转向角度

由于车轮距离（前后轮胎之间的距离）为$L$，所以$(x_f,y_f)$可以表示为：
$$
\begin{array}{l}{x_{f}=x+L \cos (\theta)} \\ {y_{f}=y+L \sin (\theta)}\end{array}
$$
带入上式则有：
$$
\dot{x} \sin (\theta+\delta)-\dot{y} \cos (\theta+\delta)-\dot{\theta} L \cos (\delta) = 0
$$
$(\dot{x}, \dot{y})$可以通过纵向速度$v$来表示：
$$
\begin{array}{l}{\dot{x}=v \cos (\theta)} \\ {\dot{y}=v \sin (\theta)}\end{array}
$$
则有：
$$
\dot{\theta}=v \tan (\delta) / L
$$
又由于车辆的瞬时曲率半径R是由$v$以及$\theta$来决定的：
$$
R=v / \dot{\theta}
$$
结合上式，可得：
$$
\tan (\delta)=L / R
$$
最终，以上运动学模型可以通过矩阵形式表达出来：
$$
\left[\begin{array}{c}{\dot{x}} \\ {\dot{y}} \\ {\dot{\theta}} \\ {v} \\ {\dot{\delta}}\end{array}\right]=\left[\begin{array}{c}{\cos \theta} \\ {\sin \theta} \\ {(\tan (\delta) / L)} \\ {1} \\ {0}\end{array}\right] v+\left[\begin{array}{l}{0} \\ {0} \\ {0} \\ {0} \\ {1}\end{array}\right] \dot{\delta}
$$



## Velocity Space Search

根据现有的运动轨迹模型，可以推算出机器人的运动轨迹。通过对速度进行对组采样，可对推算生成的轨迹进行评价。



### Circular trajectories

动态窗口方法仅考虑由平移和旋转速度的对$(v，w)$唯一确定的圆形轨迹（curvatures），受自身最大速度最小速度的限制：
$$
\left\{v \in\left[v_{\min }, v_{\max }\right], w \in\left[w_{\min }, w_{\max }\right]\right\}
$$
得到一个二维的搜索空间$V_s$，且满足的轨迹要求满足轨迹不与障碍物相交。

例如，在论文中的场景：

![1561125765940](../misc/1.1561125765940.png)

可得到速度空间如下：

![1561125786961](../misc/1.1561125786961.png)

可以发现：

- 缩小的搜索空间是二维的，因此易于处理
- 每次时间间隔后都会重复搜索
- 如果没有给出新的命令，机器人速度将自动保持不变

为了使优化成为可能，动态窗口方法仅考虑第一时间间隔$t_0$，并假设剩余$n-1$个时间间隔内的速度是恒定的，即相当于假设$[t_1,t_n]$为零加速度。



### Admissible Velocities

为了保证安全，机器人能够在碰到障碍物前停下来，受限于机器人的最大加（减）速度，速度有一个范围：
$$
V_a = \left\{v, \omega | v \leq \sqrt{2 \cdot \operatorname{dist}(v, \omega) \cdot \dot{v}_{b}} \wedge \omega \leq \sqrt{2 \cdot \operatorname{dist}(v, \omega) \cdot \dot{\omega}_{b}}\right\}
$$
其中$dist(v,w)$为速度$(v,w)$对应轨迹上离障碍物最近的距离。

**Note**: 这个条件并不是在采样一开始就能得到的。需要我们模拟出来机器人轨迹以后，找到障碍物位置，计算出机器人到障碍物之间的距离，然后看当前采样的这对速度能否在碰到障碍物之前停下来，如果能够停下来，那这对速度就是可接受的admissible。如果不能停下来，这对速度就得抛弃掉。



### Dynamic Window (动态窗口)

依据机器人的加减速性能限定速度采用空间在一个可行的动态范围内，如图所示。

![1561126569175](../misc/1.1561126569175.png)

考虑到电机可发挥的有限的加速度，整个搜索空间减少到动态窗口，该窗口仅包含下一个时间间隔内可以达到的速度。
$$
V_d=\left\{(v, \omega) | v \varepsilon\left[v_{a}-\dot{v} \cdot t, v_{a}+\dot{v} \cdot t\right] \wedge \omega \varepsilon\left[\omega_{a}-\dot{\omega} \cdot t, \omega_{a}+\dot{\omega} \cdot t\right]\right\}
$$


动态窗口是以实际速度为中心的，它的扩展取决于可以施加的加速度。动态窗口外的所有轨迹都不能在下一个时间间隔内达到，因此可以不考虑避障。

动态窗口采样的轨迹如下图所示：

![img](../misc/1.Center.jpg)



### Resulting Search Space

缩减后的速度空间为：
$$
	V_{r}=V_{s} \cap V_{a} \cap V_{d}
$$



## Optimization

根据得到的采样空间$V_r$，采用评价函数的方式为每条轨迹进行评价：
$$
G(v, \omega)=\sigma(\alpha \operatorname{heading}(v, \omega)+\beta \cdot \operatorname{dist}(v, \omega)+\gamma \cdot v e l o c i t y(v, \omega))
$$

动态窗口外的评价设为$-1$。



### Target heading

方位角评价函数$heading(v,w) $测量机器人与目标方向的对齐，即评价机器人在当前设定的采样速度下，达到模拟轨迹末端时的朝向和目标之间的角度差距，差距越小，评价得分越高：

![1561127317943](../misc/1.1561127317943.png)

由于该方向随着不同的速度而变化，因此针对机器人的预测位置计算$\theta$。为了确定预测位置，我们假设机器人在下一个时间间隔内以所选速度移动。为了对目标航向进行实际测量，我们必须考虑旋转的dynamics。 因此，在机器人在下一个间隔之后施加最大减速度时将到达的位置处计算$\theta$。 

当机器人绕过障碍物时，这使得机器人的行为平稳地转向目标。

![1561127389462](../misc/1.1561127389462.png)

其中，非允许（non admissible）速度的值设置为零



### Clearance

$dist(v,\omega)$表示与curvature相交的最近障碍物的距离。若没有障碍物，则将该值设置为大常数。

![1561128808269](../misc/1.1561128808269.png)



### Velocity

$velocity (v,\omega)$用于评估机器人在相应轨迹上的进度，它只是对平移速度$v$的投影。

![1561129010960](../misc/1.1561129010960.png)



### Smoothing

平滑处理，即归一化，上面三个部分计算出来以后不是直接相加。而是每个部分在归一化以后，再相加。

通过最大化间隙和速度，机器人将始终进入自由空间，但没有动力朝着目标位置移动。 通过单独最大化目标航向，机器人很快就会被阻挡其路径的第一个障碍物阻挡，无法绕过它。 

总结起来三者构成的评价函数的物理意义是：在局部导航过程中，使得机器人避开障碍，朝着目标以较快速度行驶，缺一不可。通过组合所有三个组件，机器人在上面列出的约束条件下尽可能快地绕过碰撞，同时仍然朝着达到目标的方向前进。

![1561129144757](../misc/1.1561129144757.png)

归一化过程如下：
$$
\begin{aligned} \text {normal head}(i) &=\frac{\text {head}(i)}{\sum_{i=1}^{n} h e a d(i)} \\ \text {normal}_{-} d i s t(i) &=\frac{\operatorname{dist}(i)}{\sum_{i=1}^{n} \operatorname{dist}(i)} \\ \text {normal volocity}(i) &=\frac{v e l o c i t y(i)}{\sum_{i=1}^{n} v e \operatorname{locity}(i)} \end{aligned}
$$
其中，$n$为采样的所有轨迹，$i$为待评价的当前轨迹。

归一化的目的是smoothing：例如障碍物距离，机器人传感器检测到的最小障碍物距离在二维空间中是不连续的，这条轨迹能够遇到障碍，旁边那边不一定能遇到。并且这条轨迹最小的障碍物距离是1m，旁边那条就是10m。那么障碍物距离的这种评价标准导致评价函数不连续，也会导致某个项在评价函数中太占优势，如这里的离障碍物距离10m相对于1m就太占优势。这样，都变成同一百分比了，每个障碍物最小距离是这100份中的一份。



### Role of the Current Velocity

**例**：

假设加速度为$50cm/sec^2$和$60deg/sec^2$，给定直线运动，平移速度分别为$75$和$40cm/sec$，图12示出了动态窗口$V_{d1}$和$V_{d2}$：

![1561129623377](../misc/1.1561129623377.png)

在图13和14中，示出了动态窗口的目标函数，动态窗口外的评估设为$-1$：

![1561129707806](../misc/1.1561129707806.png)

在第一种情况下，当前速度为$75cm/sec$时，机器人移动得太快，无法通过打开的门进行向右急转弯。 图13中，反映为导致向右转弯的速度不可接受。因此， 在动态窗口中的速度中，具有最大评估的速度产生直线运动，机器人沿着走廊继续前进，如图13中的垂直线和图12中的$V_{d1}$顶部的交叉标记所示。

![1561129733532](../misc/1.1561129733532.png)

在第二种情况下，如果当前速度是$40cm/sec$，则动态窗口$V_{d2}$包括通过门的轨迹（见图12和14）。 由于对角度和距离的更好评估，所选择的速度是向右转动最极端的速度，通过门到达目标。



### Dependency on the Accelerations

受限制于搜索空间，评估功能和动态窗口也取决于给定的加速度。

**例**：

考虑加速度为$20 cm / sec^2$和$30 deg / sec^2$的示例的速度空间，如图15所示。

![1561130593879](../misc/1.1561130593879.png)

由于加速度小，允许速度的空间小于图4。

![1561125786961](../misc/1.1561125786961.png)

因此，我们只考虑速度至$60cm/sec^2$和$50deg/sec^2$。 具有$40cm / sec$的平移速度的直线运动的动态窗口由白色区域表示。 该窗口的大小减小，因为它取决于加速度。 

图16包含整个目标函数：

![1561130864310](../misc/1.1561130864310.png)

限制在动态窗口中的速度的空间的评估如图17所示：

![1561130992047](../misc/1.1561130992047.png)

同理，此时机器人速度太快而不能急剧转入门，直线沿走廊运动的速度具有最大的评估。



## Algorithm

![1561131145000](../misc/1.1561131145000.png)

### Implementation Details

- Rotate away mode：在极少数情况下，我们观察到机器人卡在局部最小值。 如果没有允许的轨迹允许机器人平移，则就是这种情况。 当这种情况发生时。 这很容易被发现。 机器人远离障碍物旋转，直到它能够再次平移。
- Speed dependent side clearance：为了使机器人的速度根据侧面间隙适应障碍物，我们在机器人周围引入了一个安全边缘，它随机器人的平移速度线性增长。 因此，机器人将通过走廊高速行进，并在通过狭窄的门行驶时减速。 同时，可以考虑前面在关于近似误差上限的部分中描述的近似误差的可能偏差。



## References

1. Fox, D., Burgard, W., & Thrun, S. (1997). [The Dynamic Window Approach to Collision Avoidance]( https://doi.org/10.1109/100.580977). *IEEE Robotics & Automation Magazine*, 4(1), 23–33.
2. [这道题我不会做啊啊啊](https://me.csdn.net/weixin_40884570). 路径规划与避障算法——DWA算法流程: [1](https://blog.csdn.net/weixin_40884570/article/details/82764989), [2](https://blog.csdn.net/weixin_40884570/article/details/85004550), [3](https://blog.csdn.net/weixin_40884570/article/details/91345150)
3. [张天佳](https://www.zhihu.com/people/zhang-tian-jia-33). [ros导航-动态窗口方法（*Dynamic* *Window* *Approach*）](https://zhuanlan.zhihu.com/p/67335058)
4. [白巧克力亦唯心](https://me.csdn.net/heyijia0327). [机器人局部避障的动态窗口法(dynamic window approach)](https://blog.csdn.net/heyijia0327/article/details/44983551)
5. [peakzuo](https://blog.csdn.net/peakzuo). [DWA算法分析](https://blog.csdn.net/peakzuo/article/details/86487923)
6. [苏碧落](https://me.csdn.net/subiluo). [Dynamic Window Approach_机器人局部避障的动态窗口法](https://blog.csdn.net/subiluo/article/details/81912732)
7. <http://wiki.ros.org/dwa_local_planner>
8. <https://github.com/AtsushiSakai/PythonRobotics>