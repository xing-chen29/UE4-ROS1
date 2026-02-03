# 一、虚幻引擎源下载和ros1安装
# 二、ROSIntegration下载与配置运行
## 1、配置ROSBridge
要启用虚幻和ROS之间的通信，需要一个正在运行的ROSBridge和bson_mode。
注意：请使用 rosbridge 版本=>0.8.0 以获得完整的 BSON 支持。
安装rosbridge的推荐方法是在ROS工作空间使用源代码进行编译，即把rosbridge作为其中一个功能包，按照如下命令顺序执行。
~~~bash
sudo apt-get install ros-ROS1_DISTRO-rosauth # 将 ROS1_DISTRO 替换为ROS对应的版本名称
~~~
~~~bash
cd ~/ros_workspace/ # 替换 ros_workspace 为工作空间目录名称
source devel/setup.bash
cd src/
git clone -b ros1 https://github.com/RobotWebTools/rosbridge_suite.git
cd ..
catkin_make
source devel/setup.bash
~~~
此外，ROSIntegration使用包含在PyMongo包中的BSON，可以单独安装
~~~
sudo pip3 install pymongo
~~~
## 2、配置ROSIntegration
### 2.1 下载安装ROSIntegration
ROSIntegration github下载地址：`https://github.com/code-iai/ROSIntegration`

创建一个新的C++虚幻项目（假设命名为unreal_engine_project），或打开现有项目，关闭项目

在项目文件夹根目录创建Plugins文件夹，将解压出的ROSIntegration文件夹复制进Plugins文件夹
此时，ROSIntegration在虚幻项目中的文件结构如下：
`unreal_engine_project（unreal_engine_project 为真实的项目名称）/Plugins/ROSIntegration/ROSIntegration.uplugin`

### 2.2 前置准备
重新打开刚才新建的c++项目，并且会重新编译

在内容浏览器中查找（在内容浏览器的右下角启用“查看选项”>“显示插件内容”）
点击“添加/导入”按钮下方的三条线按钮，展开左侧区域
选中“ROSIntegration“>“Classes”，右键ROSIntegrationGameInstance，点击下图黄色选项
打开新的C++类/蓝图对象（推荐蓝图），并更改ROSBridgeSeverHost 和ROSBridgeServerPort。

如果是主机虚幻4虚拟机启动ros1，则ROSBridgeSeverHost改为虚拟机ipv4地址；如果是本地运行的ROSBridge，则ROSBridgeSeverHost改为127.0.0.1即可。
ROSBridgeServerPort默认9090，不用改。

打开“地图和模式”>“项目设置”，并将游戏实例设置为与新的游戏实例对象匹配，比如<u>My</u>ROSIntegrationGameInstance，而不是插件中的ROSIntegrationGameInstance

## 3、使用ROSIntegration
### 3.1 创建一个新的C++ Actor
要进行与 ROS 的简单发布/订阅通信，需要在创建一个新的C++ Actor，而非中文的角色（Charactor）。
接着创建 SamplePubliser
### 3.2 vs2019中编辑对应文件
~~~
SamplePublisher.h
SamplePublisher.cpp
unreal_engine_project.Build.cs
~~~

#### 3.2.1 `SamplePublisher.h`
以下代码必须在#include "SamplePublisher.generated.h"之前，否则会报错
~~~cpp
#include "ROSIntegration/Classes/RI/Topic.h"
#include "ROSIntegration/Classes/ROSIntegrationGameInstance.h"
#include "ROSIntegration/Public/std_msgs/String.h"
~~~

#### 3.2.2 `SamplePublisher.cpp`
以下代码放置在BeginPlay()函数中
~~~cpp

UTopic *ExampleTopic = NewObject<UTopic>(UTopic::StaticClass());
UROSIntegrationGameInstance* rosinst = Cast<UROSIntegrationGameInstance>(GetGameInstance());
ExampleTopic->Init(rosinst->ROSIntegrationCore, TEXT("/example_topic"), TEXT("std_msgs/String"));

ExampleTopic->Advertise();

TSharedPtr<ROSMessages::std_msgs::String> StringMessage(new ROSMessages::std_msgs::String("This is an example"));
ExampleTopic->Publish(StringMessage);
~~~
#### 3.2.3 `unreal_engine_project.Build.cs`

进入unreal_engine_project/Source/unreal_engine_project目录（unreal_engine_project 为真实的项目名称），打开unreal_engine_project.Build.cs文件

找到:
~~~c#
PublicDependencyModuleNames.AddRange(new string[] { "Core", "CoreUObject", "Engine", "InputCore" });

~~~
添加ROSIntegrationy依赖，形如：
~~~c#
PublicDependencyModuleNames.AddRange(new string[] { "Core", "CoreUObject", "Engine", "InputCore", "ROSIntegration" });
~~~

### 3.3 虚拟机创建工作空间

进入ROS工作空间的src目录，创建测试功能包：
~~~
catkin_create_pkg ue_test std_msgs rospy roscpp
~~~
编译并source：
~~~
catkin_make
source devel/setup.bash
~~~
创建一个监听者cpp文件：
~~~
cd ue_test/src
touch listener.cpp
~~~
打开cpp并键入如下代码：
~~~cpp
#include "ros/ros.h"
#include "std_msgs/String.h"

void chatterCallback(const std_msgs::String::ConstPtr& msg)
{
  ROS_INFO("I heard: [%s]", msg->data.c_str());
}

int main(int argc, char **argv)
{

  ros::init(argc, argv, "listener");//初始化
  ros::NodeHandle n;//节点句柄
  ros::Subscriber sub = n.subscribe("/example_topic", 1000, chatterCallback);//创建发布对象
  ros::spin();//回调

  return 0;
}
~~~
~~~cpp
/*
#include "ros/ros.h"
#include "std_msgs/String.h"
 * This tutorial demonstrates simple receipt of messages over the ROS system.

void chatterCallback(const std_msgs::String::ConstPtr& msg)
{
  ROS_INFO("I heard: [%s]", msg->data.c_str());
}

int main(int argc, char **argv)
{
  *****************************************************************************
   * The ros::init() function needs to see argc and argv so that it can perform
   * any ROS arguments and name remapping that were provided at the command line.
   * For programmatic remappings you can use a different version of init() which takes
   * remappings directly, but for most command-line programs, passing argc and argv is
   * the easiest way to do it.  The third argument to init() is the name of the node.
   *
   * You must call one of the versions of ros::init() before using any other
   * part of the ROS system.
  *****************************************************************************
  ros::init(argc, argv, "listener");
  *****************************************************************************
   * NodeHandle is the main access point to communications with the ROS system.
   * The first NodeHandle constructed will fully initialize this node, and the last
   * NodeHandle destructed will close down the node.
  *****************************************************************************
  ros::NodeHandle n;
  *****************************************************************************
   * The subscribe() call is how you tell ROS that you want to receive messages
   * on a given topic.  This invokes a call to the ROS
   * master node, which keeps a registry of who is publishing and who
   * is subscribing.  Messages are passed to a callback function, here
   * called chatterCallback.  subscribe() returns a Subscriber object that you
   * must hold on to until you want to unsubscribe.  When all copies of the Subscriber
   * object go out of scope, this callback will automatically be unsubscribed from
   * this topic.
   *
   * The second parameter to the subscribe() function is the size of the message
   * queue.  If messages are arriving faster than they are being processed, this
   * is the number of messages that will be buffered up before beginning to throw
   * away the oldest ones.
  *****************************************************************************
  ros::Subscriber sub = n.subscribe("/example_topic", 1000, chatterCallback);
  *****************************************************************************
   * ros::spin() will enter a loop, pumping callbacks.  With this version, all
   * callbacks will be called from within this thread (the main one).  ros::spin()
   * will exit when Ctrl-C is pressed, or the node is shutdown by the master.
  *****************************************************************************
  ros::spin();
  return 0;
}*/
~~~
在CMakeLists.txt添加：
~~~
add_executable(listener src/listener.cpp)
target_link_libraries(listener ${catkin_LIBRARIES})
add_dependencies(listener listener)
~~~
## 4、测试ROSIntegration
启动rosbridge
~~~
roslaunch rosbridge_server rosbridge_tcp.launch bson_only_mode:=True
~~~
运行新建功能包的监听者
~~~
# rosrun <your package> listener 
# 比如
rosrun ue_test listener 
~~~
将在UE中新建的SamplePublisher托入三维世界中，并点击运行

此时可以看到：
~~~
[INFO] [1588662504.536355639]: I heard: [This is an example]
~~~

# 三、ROSIntegrationVision下载与配置运行
## 1、配置ROSIntegrationVision
### 1.1 下载ROSIntegrationVision
github下载地址：`https://github.com/code-iai/ROSIntegrationVision`
将ROSIntegrationVision文件夹放置在虚幻引擎项目文件Plugins文件夹下

重新打开项目会重新编译

## 2、前置准备
### 2.1 `VisionActor.cpp`
注意：使用时需要先在VisionActor.cpp中作如下修改
文件目录：`unreal_engine_project（unreal_engine_project 为真实的项目名称）\Plugins\ROSIntegrationVision\Source\ROSIntegrationVision\Private\VisionActor.cpp`
~~~cpp
AVisionActor::AVisionActor() : AActor()
{
	UE_LOG(LogTemp, Warning, TEXT("VisionActor CTOR"));

	// Set this actor to call Tick() every frame.  You can turn this off to improve performance if you don't need it.
	PrimaryActorTick.bCanEverTick = true;
    
    RootComponent = CreateDefaultSubobject<USceneComponent>(TEXT("Root"));
    SetRootComponent(RootComponent);
    
    vision = CreateDefaultSubobject<UVisionComponent>(TEXT("Vision"));
    vision->DisableTFPublishing = true;   // 添加
    //vision->ParentLink = "/world";   注释掉
    vision->ParentLink = "desired_link";  // 添加
    vision->SetupAttachment(RootComponent);
}
~~~
### 2.2 ROSIntegrationVision文件夹
文件目录：`unreal_engine_project（unreal_engine_project 为真实的项目名称）\Plugins`
将ROSIntegrationVision插件中的Binaries和Intermediate文件夹删除，重新打开项目，使引擎重新编译插件


## 3、使用ROSIntegrationVision
### 3.1 主机虚幻
在内容浏览器ROSIntegrationVision/ROSIntegrationVision/Private中包含VisionActor C++文件，将其托入三维世界中即可现实摄像头图像信息
### 3.2 虚拟机
和ROSIntegration大致相同

3.2.1 启动rosbridge
~~~
roslaunch rosbridge_server rosbridge_tcp.launch bson_only_mode:=True
~~~
3.2.2 运行新建功能包的监听者
~~~
# rosrun <your package> listener 
# 比如
rosrun ue_test listener 
~~~
3.2.3 打开rviz
3.2.4 添加内容，选择By topic
### 3.3 可能遇到的问题
如果在运行rosbridge时遇到如下问题
~~~
[ERROR] [1668582258.304523]:[Client o] [id:publish:/unreal_ros/camera_info:50]
lish:1
Message
type sensor_msgs/CameraInfo does not have a field d
~~~
可以修改ROSIntegration/Source/ROSIntegration/Private/Conversion/Messages/sensor_msgs/SensorMsgsCameraInfoConverter.h文件
替换
~~~cpp
	static void _bson_append_camera_info(bson_t *b, const ROSMessages::sensor_msgs::CameraInfo *msg)
	{
		// assert(CastMsg->D.Num() == 5); // TODO: use Unreal assertions
		assert(CastMsg->K.Num() == 9); // TODO: use Unreal assertions
		assert(CastMsg->R.Num() == 9);
		assert(CastMsg->P.Num() == 12);
		
		UStdMsgsHeaderConverter::_bson_append_child_header(b, "header", &msg->header);
		BSON_APPEND_INT32(b, "height", msg->height);
		BSON_APPEND_INT32(b, "width", msg->width);
		BSON_APPEND_UTF8(b, "distortion_model", TCHAR_TO_UTF8(*msg->distortion_model));
		_bson_append_double_tarray(b, "d", msg->D);
		_bson_append_double_tarray(b, "k", msg->K);
		_bson_append_double_tarray(b, "r", msg->R);
		_bson_append_double_tarray(b, "p", msg->P);	
		BSON_APPEND_INT32(b, "binning_x", msg->binning_x);
		BSON_APPEND_INT32(b, "binning_y", msg->binning_y);
		USensorMsgsRegionOfInterestConverter::_bson_append_child_roi(b, "roi", &msg->roi);
	}
~~~
为
~~~cpp
	static void _bson_append_camera_info(bson_t *b, const ROSMessages::sensor_msgs::CameraInfo *msg)
	{
		// assert(CastMsg->D.Num() == 5); // TODO: use Unreal assertions
		assert(CastMsg->K.Num() == 9); // TODO: use Unreal assertions
		assert(CastMsg->R.Num() == 9);
		assert(CastMsg->P.Num() == 12);
		
		UStdMsgsHeaderConverter::_bson_append_child_header(b, "header", &msg->header);
		BSON_APPEND_INT32(b, "height", msg->height);
		BSON_APPEND_INT32(b, "width", msg->width);
		BSON_APPEND_UTF8(b, "distortion_model", TCHAR_TO_UTF8(*msg->distortion_model));
		_bson_append_double_tarray(b, "D", msg->D); // 替换
		_bson_append_double_tarray(b, "K", msg->K); // 替换
		_bson_append_double_tarray(b, "R", msg->R); // 替换
		_bson_append_double_tarray(b, "P", msg->P);	// 替换
		BSON_APPEND_INT32(b, "binning_x", msg->binning_x);
		BSON_APPEND_INT32(b, "binning_y", msg->binning_y);
		USensorMsgsRegionOfInterestConverter::_bson_append_child_roi(b, "roi", &msg->roi);
	}
~~~
如果相机图象FPS较低，可以考虑修改VisionComponent.cpp中 Framerate(1) 为 Framerate(100)
~~~cpp
UVisionComponent::UVisionComponent() :
Width(640),
Height(480),
Framerate(100),    // change 1 to 100
UseEngineFramerate(false),
ServerPort(10000),
FrameTime(1.0f / Framerate),
TimePassed(0),
ColorsUsed(0)
~~~