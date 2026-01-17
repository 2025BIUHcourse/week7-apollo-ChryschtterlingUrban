# 第11组 Week 7 实验报告：Apollo 建图与定位技术实战

## 1. 实验任务
本周实验任务主要包括三个部分：
1. 虚拟车道线地图构建：基于车辆行驶轨迹构建适用于Apollo系统的地图

2. RTK定位模块部署：实现基于差分GPS的高精度定位

3. NDT定位与精度评估：实现基于激光雷达点云匹配的定位并进行精度分析


## 2. 实现过程（含主要步骤与截图）
### 2.1 虚拟车道线地图构建
#### 2.1.1 数据采集与环境准备
- 首先确认激光雷达外参标定已完成，参数文件位于：
```
/apollo/modules/calibration/data/shenlan_hooke/lidar_params/lidar_novatel_extrinsics.yaml
```
- 启动必要的传感器模块：
![apollo](../imges/apollo.jpg)
所有模块正常启动后显示的界面
- 录制车辆行驶轨迹数据：
```
cyber_recorder record -a -i 600 -o localization.record
```
#### 2.1.2 地图构建步骤
1. 提取轨迹路径：
```
./bazel-bin/modules/tools/map_gen/extract_path \
  ./path.txt \
  /data/bag/localization/*
```
2. 调整车道宽度：修改```map_gen_single_lane.py```中的参数：
```
# 第27行
LANE_WIDTH = 5.0  # 设置为5米宽度
```
3. 生成基础地图：
```
./bazel-bin/modules/tools/map_gen/map_gen_single_lane \
  ./path.txt \
  ./base_map.txt \
  1
```
4. 添加地图头部信息：
```
header {
  version:"0326"
  date:"20220326"
  projection {
    proj:"+proj=tmerc +lat_0={39.52} +lon_0={116.28} +k={-48.9} +ellps=WGS84 +no_defs"
  }
}
```
5. 格式转换与地图生成：
```
# 创建地图目录
mkdir -p modules/map/data/map_test

# 转换为二进制格式
./bazel-bin/modules/tools/create_map/convert_map_txt2bin \
  -i /apollo/modules/map/data/map_test/base_map.txt \
  -o /apollo/modules/map/data/map_test/base_map.bin

# 生成路由地图
bash scripts/generate_routing_topograph.sh --map_dir /apollo/modules/map/data/map_test

# 生成仿真地图
./bazel-bin/modules/map/tools/sim_map_generator \
  --map_dir=/apollo/modules/map/data/map_test \
  --output_dir=/apollo/modules/map/data/map_test
```
#### 2.1.3 地图可视化
- 安装必要的图形依赖：
```
sudo apt update
sudo apt-get install tcl-dev tk-dev python3-tk
```
- 可视化地图：
```
./bazel-bin/modules/tools/mapshow/mapshow \
  -m /apollo/modules/map/data/map_test/base_map.txt
```
- 重启Dreamview加载新地图：
```
bash scripts/bootstrap.sh restart
```

### 2.2 RTK定位模块部署
#### 2.2.1 DAG文件配置
创建RTK定位启动文件：```dag_streaming_rtk_localization.dag```
```
module_config {
  module_library: "/apollo/bazel-bin/modules/localization/rtk/librtk_localization_component.so"
  components {
    class_name: "RTKLocalizationComponent"
    config {
      name: "rtk_localization"
      config_file_path: "/apollo/modules/localization/conf/rtk_localization.pb.txt"
      readers: [
        {
          channel: "/apollo/sensor/gnss/odometry"
          qos_profile: {
            depth: 10
          }
          pending_queue_size: 50
        }
      ]
    }
  }
}
```
![dag](../imges/dag.jpg)
#### 2.2.2 配置文件修改
修改```localization.conf```关键配置：
```
# 第5行，指定地图路径
--map_dir=/apollo/modules/map/data/map_test

# 第115行，设置UTM区域
--local_utm_zone_id=49  # 海南地区
```
#### 2.2.3 启动与验证
1. 启动Transform和GNSS/IMU模块
2. 运行RTK定位：
```
mainboard -d modules/localization/dag/dag_streaming_rtk_localization.dag
```
3. 监控定位状态：
![monitor](../imges/monitor.jpg)
监控各通道运行状态
4. 检查定位输出：
![po](../imges/po.jpg)
INS状态显示组合导航正常
5. GNSS状态确认：
![wending](../imges/wending.jpg)
显示RTK稳定解，定位定向正常

### 2.3 NDT定位与精度评估
#### 2.3.1 NDT地图构建
```
# 生成NDT定位地图
bash supplement/ndt_map_creator.sh \
  data/bag/localization \
  /apollo/modules/calibration/data/shenlan_hooke/lidar_params/lidar_novatel_extrinsics.yaml \
  49 \
  /apollo/modules/map/data/map_test/ndt_map \
  lidar

# 生成MSF可视化地图
bash supplement/msf_map_creator.sh \
  data/bag/localization \
  /apollo/modules/calibration/data/shenlan_hooke/lidar_params/lidar_novatel_extrinsics.yaml \
  49 \
  /apollo/modules/map/data/map_test \
  lidar
```
#### 2.3.2 NDT定位可视化
1. 启动NDT定位：
```
mainboard -d modules/localization/dag/dag_streaming_ndt_localization.dag
```
2. 启动可视化组件：
```
mainboard -d modules/localization/dag/dag_streaming_msf_visualizer.dag
```
3. 可视化界面：
![tu2](../imges/tu2.jpg)
可视化界面显示各模块状态
4. 地图预览：
![tu](../imges/tu.jpg)
NDT地图点云分布情况
#### 2.3.3 精度评估
1. 录制测试数据：
```
cyber_recorder record -a -i 600 -o ndt.record
```
2. 运行评估脚本：
```
bash scripts/msf_local_evaluation.sh data/bag/ndt
```
3. 评估结果：
![j](../imges/j.jpg)
定位精度评估数据
**关键指标分析：**
- 横向精度：平均0.084m，标准差0.0025m

- 纵向精度：平均0.099m，标准差0.0062m

- 鲁棒性指标（<30cm）：横向100%，纵向100%

- 高精度指标（<10cm）：横向100%，纵向59.6%

## 3. 遇到的问题与解决方法
### 3.1 地图构建中的问题
#### 问题1：Docker容器重启后软件源修改失效
- **现象：** 在Docker中使用```dev_start.sh```启动后，之前修改的软件源配置被重置
- **原因分析：** ```dev_start.sh```会重建Docker容器，导致所有修改丢失
- **解决方法：**
1. 使用容器ID直接启动现有容器：
```
docker start [容器ID]
```
2. 或者将软件源修改固化到Dockerfile中（需要重新构建镜像）
#### 问题2：地图生成脚本权限问题
- **现象：** 执行脚本时提示权限不足
- **解决方法：**
```
chmod +x supplement/*.sh
chmod +x scripts/*.sh
```
### 3.2 RTK定位部署问题
#### 问题1：缺少必要的GNSS话题
- **现象：** 部分数据包缺少```/apollo/sensor/gnss/odometry```通道
- **原因分析：** 部分数据集仅提供```/apollo/localization/pose```数据
- **解决方法：** 使用工具生成缺失的话题
```
# 生成ins_stat话题
./bazel-bin/modules/tools/sensor_calibration/ins_stat_publisher

# 生成odometry话题
./bazel-bin/modules/tools/sensor_calibration/odom_publisher
```
如果工具不存在，重新编译：
```
./apollo.sh build_opt tools
```
#### 问题2：UTM区域配置错误
- **现象：** 定位结果坐标异常
- **解决方法：** 确认并正确设置```local_utm_zone_id``
海南地区：49
### 3.3 NDT定位可视化问题
#### 问题1：可视化界面不显示地图
- **现象：** 启动可视化组件后地图无法显示，但监控显示定位正常
- **解决方法：** 清除缓存文件
```
rm -rf cyber/data/map_visual
```
重新启动可视化组件
#### 问题2：点云匹配效果不佳
- **现象：** 可视化界面中点云与地图匹配有明显偏差
- **可能原因：**
  1. 激光雷达外参标定不准确

  2. 地图构建质量差

  3. 定位初始化偏差大
- **解决方法：**
  1. 重新进行传感器标定

  2. 检查地图构建数据质量

  3. 调整定位初始化参数
### 3.4 精度评估中的数据处理问题
#### 问题1：评估脚本报错
- **现象：** 运行```msf_local_evaluation.sh```时出现文件不存在错误
- **解决方法：** 确保数据路径正确，并检查依赖关系
```
# 检查脚本执行权限
ls -la scripts/msf_local_evaluation.sh

# 检查数据目录
ls -la data/bag/ndt/
```
## 4.总结与心得
**地图体系理解：**

1. **基础地图（base_map）：** 包含完整道路和车道几何信息，是其他地图的基础

2. **路由地图（routing_map）：** 仅包含车道拓扑结构，用于路径规划

3. **仿真地图（sim_map）：** 轻量级地图，用于Dreamview可视化

4. **定位图层（NDT/MSF）：** 专门用于定位算法的地图格式

**定位技术理解：**
1. **RTK定位：**

   - 原理：通过基站差分消除GNSS误差

   - 优势：厘米级精度，无需复杂地图

   - 局限：依赖卫星信号，受环境影响大

2. **NDT定位：**
   - 原理：基于正态分布变换的点云匹配

   - 优势：不依赖GNSS，适合城市峡谷环境

   - 局限：需要预先构建高精度点云地图

**系统架构理解：**
1. **CyberRT框架：** 理解DAG文件配置、组件通信机制

2. **传感器融合：** 掌握多传感器数据同步与融合方法

3. **定位评估：** 学习定位精度的量化评估方法

**报告撰写人：陈一诺**  
**提交日期：2025年1月10日**  
**GitHub仓库：https://github.com/2025BIUHcourse/week7-apollo-ChryschtterlingUrban**