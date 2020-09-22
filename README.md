# RxROS2 benchmarks

## Introduction

RxROS2 is new API for ROS2 based on the paradigm of reactive programming. Reactive programming is an alternative to callback-based programming for implementing concurrent message passing systems that emphasizes explicit data flow over control flow. It makes the message flow and transformations of messages easy to capture in one place. It eliminates problems with deadlocks and understanding how callbacks interact. It helps you to make your nodes functional, concise, and testable. RxROS2 aspires to the slogan ‘concurrency made easy’.


## Contents

This repository hosts the benchmarks we ran against the implementation of RxROS2.


## Acknowledgement

This projects has received funding from the European Union's Horizon 2020 research and innovation programme under grant agreement No 732287.

![](https://rosin-project.eu/wp-content/uploads/2017/03/EU-Flag-1.png)<br>
[https://rosin-project.eu](https://rosin-project.eu)

## Dependencies

In order to run these benchmarks, make sure to have installed RxROS2 itself.

Refer to the readme of the main RxROS2 repository for more instructions.


## Performance Measurements

In this section we will take a closer look at the performance of RxROS2. The test will compare a minimal subscriber/publisher program written in RxROS2 and plain ROS2. The test consists of two nodes that publish data and subscribe to data from the other node. The data contains a timestamp that will be updated at the moment the data is published and the latency will be calculated at the moment the other node receives the data. Besides the latency, CPU load, Memory consumption and number of used treads will be measured. Several tests will be performed with various amounts of data.

### Test setup
The tests will be performed on a RaspberryPi 4B with 4 Mbytes of RAM. The following software has been installed on the machine:

* Ubuntu 18.04.4 server image.
* ROS2 Eloquent Elusor (desktop version)
* RxCpp v2
* Latest version of RxROS2

The software has been installed as is. There has been no changes to RaspberryPi 4B to boost performance, nor has there been installed a special software to boost or optimize performance. The Ubuntu 18.04.4 and ROS2 packages has been updated via the Ubuntu package storage to the latest version.

### Test programs
To measure CPU load and RAM consumption the Linux `top` command has been used. To measure the number of used threads the `/proc/<pid>/status` file has been used and to measure latency the following programs has been used. The first is a plain ROS2 program and the other is a similar program written in RxROS2:

```cpp
#include <cstdio>
#include <chrono>
#include "rclcpp/rclcpp.hpp"
#include "std_msgs/msg/string.hpp"
#include "ros2_msg/msg/test.hpp"
using namespace std::chrono_literals;

class T1 : public rclcpp::Node
{
private:
    rclcpp::Subscription<ros2_msg::msg::Test>::SharedPtr subscription;
    rclcpp::Publisher<ros2_msg::msg::Test>::SharedPtr publisher;
    rclcpp::TimerBase::SharedPtr timer;
    size_t msg_no;
    const std::string bolb_data = std::string(1048576, '#');

    static auto age_of(const rclcpp::Time& time) {
        auto curr_time_nano = rclcpp::Clock().now().nanoseconds();
        auto time_nano = time.nanoseconds();
        return curr_time_nano-time_nano;
    }

    static auto mk_test_msg(const int msg_no, const std::string& data) {
        ros2_msg::msg::Test msg;
        msg.msg_no = msg_no;
        msg.data = data;
        msg.time_stamp = rclcpp::Clock().now();
        return msg;
    }

public:
    T1(): Node("T1"), msg_no(0) {
        RCLCPP_INFO(this->get_logger(), "Starting CppNode T1:");

        subscription = this->create_subscription<ros2_msg::msg::Test>( "/T2T", 10,
            [this](ros2_msg::msg::Test::UniquePtr msg) {
                RCLCPP_INFO(this->get_logger(), "%d,%d,%d", msg->msg_no, msg->data.length(), age_of(msg->time_stamp));
            });

        publisher = this->create_publisher<ros2_msg::msg::Test>("/T1T", 10);

        auto timer_callback = [this]() -> void {
            auto msg = mk_test_msg(this->msg_no++, this->bolb_data);
            this->publisher->publish(msg);
        };
        timer = this->create_wall_timer(50ms, timer_callback);
    }
};


int main(int argc, char * argv[])
{
    rclcpp::init(argc, argv);
    rclcpp::spin(std::make_shared<T1>());
    rclcpp::shutdown();
    return 0;
}
```

And the equivalent program written in RxROS2:

```cpp
#include <cstdio>
#include <rxros/rxros2.h>
#include "std_msgs/msg/string.hpp"
#include "ros2_msg/msg/test.hpp"
using namespace rxcpp::operators;
using namespace rxros2::operators;

struct T2: public rxros2::Node {
    T2(): rxros2::Node("T2") {};
    const std::string bolb_data = std::string(1048576, '#');

    static auto age_of(const rclcpp::Time& time) {
        auto curr_time_nano = rclcpp::Clock().now().nanoseconds();
        auto time_nano = time.nanoseconds();
        return curr_time_nano-time_nano;
    }

    static auto mk_test_msg(const int msg_no, const std::string& data) {
        ros2_msg::msg::Test msg;
        msg.msg_no = msg_no;
        msg.data = data;
        msg.time_stamp = rclcpp::Clock().now();
        return msg;
    }

    void run() {
        RCLCPP_INFO(this->get_logger(), "Starting RxCppNode T2:");

        rxros2::observable::from_topic<ros2_msg::msg::Test>(this, "/T1T")
            .subscribe ( [this] (const ros2_msg::msg::Test::SharedPtr msg) {
                RCLCPP_INFO(this->get_logger(), "%d,%d,%d", msg->msg_no, msg->data.length(), age_of(msg->time_stamp));});

        rxcpp::observable<>::interval (std::chrono::milliseconds(50))
            | map ([&](int i) { return mk_test_msg(i, bolb_data); })
            | publish_to_topic<ros2_msg::msg::Test> (this, "/T2T");
    }
};

int main(int argc, char **argv) {
    rclcpp::init(argc, argv);
    auto t2 = std::make_shared<T2>();
    t2->start();
    rclcpp::spin(t2);
    rclcpp::shutdown();
    return 0;
}
```

### Test results
The test results for ROS2 and RxROS2 (C++) are shown in the following table:

|Test|CPU min %|CPU max %|MEM min %|MEM max %|THREADS|LATENCY min ms|LATENCY max ms|LATENCY avg ms|
|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
|ros2 100B 40Hz|5.0|5.3|0.4|0.4|7|0.2527|1.0918|0.3690|
|rxros2 100B 40Hz|5.3|5.6|0.4|0.5|8|0.3458|0.8041|0.3918|
||||||||||
|ros2 1K 40Hz|5.3|5.3|0.4|0.4|7|0.2208|0.6157|0.3843|
|rxros2 1K 40Hz|5.3|5.6|0.4|0.4|8|0.2353|0.7893|0.3904|
||||||||||
|ros2 1M 40Hz|36.4|37.1|0.6|0.6|7|5.5996|27.6453|11.8814|
|rxros2 1M 40Hz|33.1|33.1|0.6|0.7|8|4.9858|29.7769|10.8161|
||||||||||
|ros2 16M 4Hz|33.4|35.1|3.0|3.4|7|48.3979|634.0808|82.0909|
|rxros2 16M 4Hz|23.8|24.2|2.6|2.6|8|40.1948|97.8250|42.6458|

The table shows 8 distinct test. The table is organized in 4 groups that consist of a ROS2 test followed by a similar test performed by a RxROS2 program. Test "ros2 100B 40Hz" means we are running two plain ROS2 programs that each publish 100 Bytes of data and subscribes to the data published by the other node. The data is published with a rate of 20Hz per note, or 40Hz in total for the two nodes.

From the table it looks like RxROS2 is outperforming plain ROS2. The larger data messages we use the more distinct is the performance improvement of RxROS2. This is an unexpected result! The expected result is that ROS2 would slightly outperform a similar RxROS2 program due to the more abstract/high level nature of the RxROS2 operators. So what is causing this outcome? If we look at the plain ROS2 program we see that the spinner, subscriber and publisher is all working in the same thread. The RxROS2 program work in a slightly different manner. During startup of the node the node is actually started up in a new thread by the t2->start() command:

```cpp
int main(int argc, char **argv) {
    rclcpp::init(argc, argv);
    auto t2 = std::make_shared<T2>();
    t2->start();
    rclcpp::spin(t2);
    rclcpp::shutdown();
    return 0;
}
```

This means that the `run` method will be called in a separate thread:

```cpp
void run() {
    RCLCPP_INFO(this->get_logger(), "Starting RxCppNode T2:");

    rxros2::observable::from_topic<ros2_msg::msg::Test>(this, "/T1T")
        .subscribe ( [this] (const ros2_msg::msg::Test::SharedPtr msg) {
            RCLCPP_INFO(this->get_logger(), "%d,%d,%d", msg->msg_no, msg->data.length(), age_of(msg->time_stamp));});

    rxcpp::observable<>::interval (std::chrono::milliseconds(50))
        | map ([&](int i) { return mk_test_msg(i, bolb_data); })
        | publish_to_topic<ros2_msg::msg::Test> (this, "/T2T");
}
```

The thread will however terminate very fast as the observable `from_topic` and `interval` are non-blocking. In fact, the `interval` observable is actually executing in a dedicated thread (scheduler), which means that topics will be published in a dedicated thread whereas the spinner, subscription and processing of topics are performed in the main thread. This is the main difference between the RxROS2 test program and the plain ROS2 program. The RxROS2 program uses one more thread than the plain ROS2 program to separate the publishing and subscription of topics.
