# ROS2 C++ 项目实战指南：从零创建到 OpenCV 集成

本文档详细总结了如何从零开始创建一个 ROS2 C++ 功能包，重点讲解如何正确配置 `CMakeLists.txt` 和 `package.xml` 以引入第三方库（OpenCV），并实现基于摄像头的实时人脸识别功能。

---

## 1. 创建功能包 (Create Package)

首先进入工作空间的 `src` 目录。假设工作空间根目录为 `~/Embodied_AI_Project/my_opencv`。

```bash
# 1. 进入 src 目录
cd ~/Embodied_AI_Project/my_opencv/src

# 2. 创建功能包
# --build-type ament_cmake: 指定为 C++ 项目
# --dependencies rclcpp: 自动添加 ROS2 C++ 客户端库依赖
# --node-name node_face_detect: 自动创建一个名为 node_face_detect.cpp 的源文件
ros2 pkg create --build-type ament_cmake --dependencies rclcpp --node-name node_face_detect my_opencv_demo
```

## 2. 配置第三方库依赖 (Configuration)

这是 C++ 开发中最容易出错的环节。我们需要分别修改 `package.xml`（针对 ROS 依赖系统）和 `CMakeLists.txt`（针对编译系统）。

---
### 2.1 修改 package.xml
此文件告诉 rosdep 需要安装哪些系统库。

* 文件路径: `src/my_opencv_demo/package.xml`

* 操作: 在 `<depend>rclcpp</depend>` 下方添加 OpenCV 依赖。

```XML
<depend>rclcpp</depend>
<depend>opencv2</depend>
```
### 2.2 修改 CMakeLists.txt
此文件告诉编译器如何找到 OpenCV 的头文件和库文件。

* 文件路径: `src/my_opencv_demo/CMakeLists.txt`

* 操作: 将内容修改为如下配置
```CMake
# 1. 寻找依赖库
# 【关键点1】CMake 查找 OpenCV 时必须注意大小写，且不带数字 "2"
find_package(OpenCV REQUIRED)

# 2. 包含头文件目录
ament_target_dependencies(
  node_apple_detect
  "rclcpp"
  "OpenCV"
)
```
## 3. 编写人脸识别代码 (Source Code)
实现功能：驱动摄像头，使用定时器回调（Timer Callback）获取视频帧，并使用 Haar 级联分类器识别。

* 文件路径: `src/my_opencv_demo/src/node_face_detect.cpp`
```C++
#include "rclcpp/rclcpp.hpp"
#include "opencv2/opencv.hpp"

class FaceDetectNode : public rclcpp::Node
{
public:
    FaceDetectNode() : Node("node_face_detect")
    {
        RCLCPP_INFO(this->get_logger(), "正在启动摄像头人脸识别节点...");

        // 1. 打开默认摄像头 (ID 0)
        cap_.open(0);
        if (!cap_.isOpened()) {
            RCLCPP_ERROR(this->get_logger(), "错误：无法打开摄像头！");
            return;
        }

        // 2. 加载 Haar Cascade 模型
        // 【注意】建议使用绝对路径，避免运行时找不到文件
        // 请根据实际情况修改用户名 (drgh)
        std::string xml_path = "/home/drgh/Embodied_AI_Project/my_opencv/src/my_opencv_demo/haarcascade_frontalface_default.xml";
        
        if (!face_cascade_.load(xml_path)) {
            RCLCPP_ERROR(this->get_logger(), "错误：无法加载模型文件！路径: %s", xml_path.c_str());
            return;
        }

        RCLCPP_INFO(this->get_logger(), "初始化完成，开始运行...");

        // 3. 创建定时器，以 30Hz (约33ms) 频率运行
        timer_ = this->create_wall_timer(
            std::chrono::milliseconds(33),
            std::bind(&FaceDetectNode::timer_callback, this));
    }

private:
    void timer_callback()
    {
        cv::Mat frame;
        cap_ >> frame; // 读取一帧
        
        if (frame.empty()) return;

        // 4. 图像预处理 (转灰度 + 缩小以提速)
        cv::Mat gray, small_frame;
        cv::cvtColor(frame, gray, cv::COLOR_BGR2GRAY);
        double scale = 0.5;
        cv::resize(gray, small_frame, cv::Size(), scale, scale);
        cv::equalizeHist(small_frame, small_frame); // 直方图均衡化增强对比度

        // 5. 人脸检测
        std::vector<cv::Rect> faces;
        // 参数: input, output, scaleFactor, minNeighbors, flags, minSize
        face_cascade_.detectMultiScale(small_frame, faces, 1.1, 3, 0, cv::Size(30, 30));

        // 6. 绘制结果
        for (const auto& face : faces)
        {
            // 还原坐标 (因为是在缩小版上检测的)
            cv::Rect real_face(face.x/scale, face.y/scale, face.width/scale, face.height/scale);
            
            // 画框
            cv::rectangle(frame, real_face, cv::Scalar(0, 255, 0), 2);
            // 写字
            cv::putText(frame, "Person", cv::Point(real_face.x, real_face.y - 10),
                        cv::FONT_HERSHEY_SIMPLEX, 0.9, cv::Scalar(0, 255, 0), 2);
        }

        // 7. 显示图像
        cv::imshow("Face Detect", frame);
        // 【关键】必须调用 waitKey，否则窗口不会刷新
        cv::waitKey(1); 
    }

    rclcpp::TimerBase::SharedPtr timer_;
    cv::VideoCapture cap_;
    cv::CascadeClassifier face_cascade_;
};

int main(int argc, char * argv[])
{
    rclcpp::init(argc, argv);
    rclcpp::spin(std::make_shared<FaceDetectNode>());
    rclcpp::shutdown();
    return 0;
}
```

## 4. 准备模型文件

代码中用到的人脸特征库文件需要单独下载。
```bash
# 进入代码所在目录
cd ~/Embodied_AI_Project/my_opencv/src/my_opencv_demo

# 下载官方模型文件
wget [https://raw.githubusercontent.com/opencv/opencv/master/data/haarcascades/haarcascade_frontalface_default.xml](https://raw.githubusercontent.com/opencv/opencv/master/data/haarcascades/haarcascade_frontalface_default.xml)

```

## 5. 编译与运行 (Build & Run)
原则：编译命令必须在工作空间根目录下执行。
```bash
# 1. 回到工作空间根目录
cd ~/Embodied_AI_Project/my_opencv

# 2. 编译项目
colcon build --packages-select my_opencv_demo

# 3. 刷新环境变量 (让系统识别新编译的包)
source install/setup.bash

# 4. 运行节点
ros2 run my_opencv_demo node_face_detect

```