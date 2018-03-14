# Apollo 2.0 感知模块：障碍物感知阅读笔记

本文档结合代码详细地解释感知模块中障碍物感知的流程与功能，也可以官方参考文档([Apollo 2.0 Obstacles Perception](https://github.com/ApolloAuto/apollo/blob/master/docs/specs/3d_obstacle_perception_cn.md))

## <a name="障碍物感知">障碍物感知: 3D Obstacles Perception</a>

![img](https://github.com/YannZyl/Apollo-Note/blob/master/images/perception_obstacles_framework.png)

上图为子节点之间的关系与数据流动，障碍物感知共存在三个子节点(线程)，分别为：

- 激光雷达处理子节点 LidarProcessSubnode
- 雷达处理子节点 RadarProcessSubnode
- 融合子节点 FusionSubnode

每个子节点的输入数据与输出数据在边上标出。

(1) 激光雷达子节点LidarProcessSubnode::OnPointCloud以ROS消息订阅与发布机制触发回调函数，处理结果保存在LidarObjectData共享数据容器中。主要解决的问题有：

- 高精地图ROI过滤器(HDMap ROI Filter)
- 基于卷积神经网络分割(CNN Segmentation)
- MinBox 障碍物边框构建(MinBox Builder)
- HM对象跟踪(HM Object Tracker)

(2) 雷达子节点RadarProcessSubnode::OnRadar同样以ROS消息订阅与发布机制触发回调函数，处理结果保存在RadarObjectData共享数据容器中。

(3) 融合子节点FusionSubnode::ProcEvents以自定义ProcEvents+EventManeger消息处理机制，从LidarObjectData和RadarObjectData共享数据容器中提取数据，融合并存储在FusionObjectData共享容器中。

### <a name="激光雷达感知">激光雷达感知 Perception: Lidar Obstacles PErception</a>

首先对激光雷达障碍物感知模块使用的数据结构做一个简单地分析，其核心数据结构为Object和SensorObject，可以分析这两类数据结构，如下表:

| 名称 | 备注 |
| ---- | ---- |
| pcl_util::PointCloudPtr cloud | PCL点云数据，原始sensor_msgs::PointCloud2需经过转化 |
| PolygonDType polygon | 物体凸包/角点，PolygonDType数据类型为pcl_util::PointCloud |
| Eigen::Vector3d direction | 物体主方向，右手坐标系，默认X轴方向，即Eigen::Vector3d(1, 0, 0) |
| double theta | 物体偏航角，默认0.0，即无偏向，向正前方 |
| Eigen::Vector3d center | 物体中心，默认自身坐标系中心，即Eigen::Vector3d::Zero() |
| double length, width, height | 物体有向标定框尺寸，length为朝向主方向轴的大小 |
| std::vector<float> shape_features | 形状特征，用于物体跟踪 |
| float score | 前景概率 |
| ObjectType type | 物体类型，行人、自信车、车辆、未知等 |
| Eigen::Vector3d velocity | 物体运动速度，用于物体跟踪 |
| double tracking_time | 跟踪时间 |
| double latest_tracked_time | 最近一次跟踪时间戳 |
| Eigen::Vector3d anchor_point | 稳定的锚点，比如说重心 |

SensorObject继承了Object类，主要数据结构如下：

| 名称 | 备注 |
| ---- | ---- |
| SensorType sensor_type | 感知数据来源，64/16线激光雷达、雷达、摄像头、未知等 |
| std::string sensor_id | 设备id |
| double timestamp | 时间戳 |
| SeqId seq_num | 数据序列，每次回调seq_num自增1 |
| std::vector<ObjectPtr> objects | 重点，感知数据中的多个物体数据 |
| Eigen::Matrix4d sensor2world_pose | 感知设备自身坐标系相对世界坐标系的系数矩阵 |
| LaneObjectsPtr lane_objects | 车道线信息 |

从上面两张表格基本可以看出激光雷达模块需要做的工作，Object类很完善地存储了单个物体的一些静态与动态信息，静态信息包括物体点云数据，凸包及标定框信息，坐标系坐标/偏向/尺寸信息，物体的类别信息等；动态信息主要是在跟踪过程中物体速度，中心变化等信息。而在一次激光雷达扫描过程中，得到的信息中包含了多类物体，因此SensorObject类中存储了Object的一个向量，刻画了本次扫描分离的物体集合，同时包含了一些标记信息如感知来源、设备id、时间戳、序列等，最后还包含了本次激光雷达扫描到的车道线信息。SensorObject类包含了每次激光雷达扫描得到的所有信息。

![img](https://github.com/YannZyl/Apollo-Note/blob/master/images/perception_obstacles_lidar.png)

上图展示了激光雷达处理的4个模块，分别为高精地图ROI过滤器、基于卷积神经网络的障碍物分割、MinBox 障碍物边框构建与HM对象跟踪。接下去将一层层解剖代码，分析各个模块的流程的方法。

#### <a name="高精地图ROI过滤">高精地图ROI过滤器</a>

高精地图ROI过滤器是回调的第一个过程，从最开始的模块框架图可以看到，LidarProcessSubnode子节点接受的输入数据类型是ROS原始的点云数据类型，sensor_msgs::PointCloud2，简单地看一下这个数据结构，也可以参考官方文档[PointCloud2](http://docs.ros.org/api/sensor_msgs/html/msg/PointCloud2.html)

```
Header header

# 2D structure of the point cloud. If the cloud is unordered, height is
# 1 and width is the length of the point cloud.
uint32 height
uint32 width

# Describes the channels and their layout in the binary data blob.
PointField[] fields

bool    is_bigendian # Is this data bigendian?
uint32  point_step   # Length of a point in bytes
uint32  row_step     # Length of a row in bytes
uint8[] data         # Actual point data, size is (row_step*height)

bool is_dense        # True if there are no invalid points
```

sensor_msgs::PointCloud2与第一个版本sensor_msgs::PointCloud有一些区别，支持任意的N维数据，并且支持任意的基础数据类型，也支持密集存储。从上面的说明可以看到，PointCloud2可以支持2D数据结构，每个点N维，每行X个通道，共H列，可以存储图像信息等。fields存储了各个通道的名称，例如x,y,z,r,g,b等，在这里我们使用的PointCloud2类型每个通道只需要xyz三维即可，表示坐标系中的位置。

还有一个细节，激光雷达获取的点云是ROS原始的sensor_msgs::PointClouds类型，而实际处理过程中使用的更多的是PCL库的pcl::PointCloud<T>类型，需要在代码中做一个转换，使用pcl_conversions的pcl::fromROSMsg和pcl::toROSMsg函数即可方便的实现相互转换。

(1) 数据转换与存储

在进行高精地图ROI过滤的过程中，第一步是接收来自激光雷达的原始点云数据，设备id，时间戳ts等信息，并将其信息存入上述的SensorObject类中。存储过程中代码中值得关注的两个点分别是传感器到世界坐标系的转换矩阵velodyne_trans以及sensor_msgs::PointCloud2到PCL::PointCloud的转换。

```
/// file in apollo/modules/perception/obstacle/onboard/lidar_process_subnode.cc
void LidarProcessSubnode::OnPointCloud(const sensor_msgs::PointCloud2& message) {
  ...
  /// get velodyne2world transfrom
  std::shared_ptr<Matrix4d> velodyne_trans = std::make_shared<Matrix4d>();
  if (!GetVelodyneTrans(kTimeStamp, velodyne_trans.get())) {
    ...
  }
  out_sensor_objects->sensor2world_pose = *velodyne_trans;

  PointCloudPtr point_cloud(new PointCloud);
  TransPointCloudToPCL(message, &point_cloud);
}
```

(2) ROI生成

当获得原始的点云数据并转换成PCL格式以后，下一步需要从点云中检索ROI区域，这些ROI区域包含路面与路口的驾驶区域。

```
/// file in apollo/modules/perception/obstacle/onboard/lidar_process_subnode.cc
void LidarProcessSubnode::OnPointCloud(const sensor_msgs::PointCloud2& message) {
  /// get velodyne2world transfrom
  if (!GetVelodyneTrans(kTimeStamp, velodyne_trans.get())) {
  	...
  }
  /// call hdmap to get ROI
  HdmapStructPtr hdmap = nullptr;
  if (hdmap_input_) {
    PointD velodyne_pose = {0.0, 0.0, 0.0, 0};  // (0,0,0)
    Affine3d temp_trans(*velodyne_trans);
    PointD velodyne_pose_world = pcl::transformPoint(velodyne_pose, temp_trans);
    hdmap.reset(new HdmapStruct);
    hdmap_input_->GetROI(velodyne_pose_world, FLAGS_map_radius, &hdmap);
    PERF_BLOCK_END("lidar_get_roi_from_hdmap");
  }
}
```

路面与路口的驾驶区域需要查询高精地图来完成，该阶段首先使用tf进行坐标系的转换(车辆/局部坐标系到世界坐标系的变换矩阵)，配合velodyne_pose计算得到velodyne_pose_world世界坐标系坐标，坐标系具体情况请参考[Apollo坐标系研究](https://github.com/YannZyl/Apollo-Note/blob/master/docs/apollo_coordinate.md)。真正获取ROI使用的是GetROI函数。具体的路口，车道存储形式请参考[高精地图模块](https://github.com/YannZyl/Apollo-Note/blob/master/docs/hdmap.md)

这个分析分析一下比较GetVelodyneTrans函数，这个函数的实现过程我们可以先简要的看一下在做功能分析：

```
/// file in apollo/modules/perception/onboard/transform_input.cc
bool GetVelodyneTrans(const double query_time, Eigen::Matrix4d* trans) {
  ...
  // Step1: lidar refer to novatel(GPS/IMU)
  if (!tf2_buffer.canTransform(FLAGS_lidar_tf2_frame_id, FLAGS_lidar_tf2_child_frame_id, query_stamp, ...) {
  	...
  }
  geometry_msgs::TransformStamped transform_stamped;
  try {
    transform_stamped = tf2_buffer.lookupTransform(FLAGS_lidar_tf2_frame_id, FLAGS_lidar_tf2_child_frame_id, query_stamp);
  } 
  Eigen::Affine3d affine_lidar_3d;
  tf::transformMsgToEigen(transform_stamped.transform, affine_lidar_3d);
  Eigen::Matrix4d lidar2novatel_trans = affine_lidar_3d.matrix();

  // Step2 notavel(GPS/IMU) refer to world coordinate system
  if (!tf2_buffer.canTransform(FLAGS_localization_tf2_frame_id, FLAGS_localization_tf2_child_frame_id, ...) {
    ...
  }
  try {
    transform_stamped = tf2_buffer.lookupTransform(FLAGS_localization_tf2_frame_id, FLAGS_localization_tf2_child_frame_id, query_stamp);
  } 
  Eigen::Affine3d affine_localization_3d;
  tf::transformMsgToEigen(transform_stamped.transform, affine_localization_3d);
  Eigen::Matrix4d novatel2world_trans = affine_localization_3d.matrix();

  *trans = novatel2world_trans * lidar2novatel_trans;
}
```

点云数据由lidar获取，所以数据都是以激光雷达参考系作为主要参考系，在查询高精地图的时候需要世界坐标系数据。因此获取变换矩阵分为两步，第一步获取激光雷达lidar坐标系到惯测单元IMU坐标系的变换矩阵；第二步，获取惯测单元IMU坐标系到世界坐标系变换矩阵。从上述的代码中我们明显可以看到有两部分相似度很高的代码组成:

- 计算仿射变换矩阵lidar2novatel_trans，激光雷达lidar坐标系到惯测IMU坐标系(车辆坐标系)变换矩阵。这个矩阵虽然通过ROS的tf模块调用lookupTransform函数计算完成，但是实际是外参决定，在运行过程中保持不变。
```
/// file in https://github.com/ApolloAuto/apollo/blob/master/modules/localization/msf/params/velodyne_params/velodyne64_novatel_extrinsics_example.yaml
child_frame_id: velodyne64
transform:
  translation:
    x: -0.0581372003122598
    y: 1.459274166013735
    z: 1.24965
  rotation:
    x: 0.02748694630673456
    y: -0.03223034579615043
    z: 0.7065742186090662
    w: 0.706369978261802
header:
  seq: 0
  stamp:
    secs: 1512689414
    nsecs: 0
  frame_id: novatel
```

- 计算仿射变换矩阵novatel2world_trans，惯测单元IMU坐标系(车辆坐标系)相对于世界坐标系的仿射变换矩阵。最后矩阵相乘得到激光雷达lidar坐标系到世界坐标系的仿射变换矩阵。

- 计算仿射变换矩阵lidar2world_trans，最终两个矩阵相乘得到激光雷达lidar坐标系到世界坐标系的变换矩阵。

(3) 高精地图ROI过滤器

高精地图 ROI 过滤器（往下简称“过滤器”）处理在ROI之外的激光雷达点，去除背景对象，如路边建筑物和树木等，剩余的点云留待后续处理。该过程共有三个子过程。

- 坐标变换。对于(高精地图ROI)过滤器来说，高精地图数据接口被定义为一系列多边形集合，每个集合由世界坐标系点组成有序点集。高精地图ROI点查询需要点云和多边形处在相同的坐标系，为此，Apollo将输入点云和HDMap多边形变换为来自激光雷达传感器位置的地方坐标系。这个阶段使用到的变换矩阵就是以上的lidar2world矩阵。看了官方文档，并配合具体的代码，可能会存在一些疑惑。这里给出一些变换的研究心得。

坐标变换的实现是在HdmapROIFilter::Filter函数中完成。具体的变换过程如下：

```
/// file in apollo/modules/perception/obstacle/lidar/roi_filter/hdmap_roi_filter/hdmap_roi_filter.cc
bool HdmapROIFilter::Filter(const pcl_util::PointCloudPtr& cloud,
                            const ROIFilterOptions& roi_filter_options,
                            pcl_util::PointIndices* roi_indices) {
  Eigen::Affine3d temp_trans(*(roi_filter_options.velodyne_trans));
  std::vector<PolygonDType> polygons;
  MergeHdmapStructToPolygons(roi_filter_options.hdmap, &polygons);
  ...
  // Transform polygon and point to local coordinates
  pcl_util::PointCloudPtr cloud_local(new pcl_util::PointCloud);
  std::vector<PolygonType> polygons_local;
  TransformFrame(cloud, temp_trans, polygons, &polygons_local, cloud_local);
  ...
}

void HdmapROIFilter::TransformFrame(
    const pcl_util::PointCloudConstPtr& cloud, const Eigen::Affine3d& vel_pose,
    const std::vector<PolygonDType>& polygons_world,
    std::vector<PolygonType>* polygons_local,
    pcl_util::PointCloudPtr cloud_local) {
  ...
}

```

上面代码中MergeHdmapStructToPolygons函数负责把路口和路面的点云并入到多边形集合polygons，这里roi_filter_options里面的数据都是经过高精地图查询得到的路口和路面信息，是基于世界坐标系的，所以结果合并后polygons也是世界坐标系的数据。这里还需要注意的一点输入的cloud是基于lidar坐标系的点云数据，而下面代码还需要转换成cloud_local、polygons_local，按照注释解释是局部坐标系，那么这个局部坐标系到底是什么坐标系？如果看得懂TransformFrame函数，可以不难发现：**这个所谓"local coordinate system"，其实跟lidar坐标系很相近，他表示以lidar为原点的ENU坐标系**,这个坐标系是以X(东)-Y(北)-Z(天)为坐标轴的二维投影坐标系。在TransformFrame函数中，

```
  Eigen::Vector3d vel_location = vel_pose.translation();
  Eigen::Matrix3d vel_rot = vel_pose.linear();
  Eigen::Vector3d x_axis = vel_rot.row(0);
  Eigen::Vector3d y_axis = vel_rot.row(1);
```

vel_location是lidar坐标系相对世界坐标系的平移，vec_rot则是lidar坐标系相对世界坐标系的旋转矩阵。那么从lidar坐标系到世界坐标系的坐标变换其实很简单，假设在lidar坐标系中有一个坐标点P(x1,y1,z1)，那么该点在世界坐标系下的坐标P_hat为: P_hat = vec_rot * P + vec_location. 了解了这个变换，接下来观察cloud和polygons的变换代码：

```
  polygons_local->resize(polygons_world.size());
  for (size_t i = 0; i < polygons_local->size(); ++i) {
    const auto& polygon_world = polygons_world[i];
    auto& polygon_local = polygons_local->at(i);
    polygon_local.resize(polygon_world.size());
    for (size_t j = 0; j < polygon_local.size(); ++j) {
      polygon_local[j].x = polygon_world[j].x - vel_location.x();
      polygon_local[j].y = polygon_world[j].y - vel_location.y();
    }
  }
```

开始的时候也是很奇怪，为什么最后变换的形式是P_local = P_world - translation. 后来经过研究**猜测(有待后续深入阅读证实)**路口和路面多边形信息只经过平移达到新的局部ENU坐标系，可以推测其实世界坐标系也是ENU坐标系(具有方向性)，所以两个坐标系之间没有旋转，直接平移就可以从世界坐标系变换到局部ENU坐标系。

```
  cloud_local->resize(cloud->size());
  for (size_t i = 0; i < cloud_local->size(); ++i) {
    const auto& pt = cloud->points[i];
    auto& local_pt = cloud_local->points[i];
    Eigen::Vector3d e_pt(pt.x, pt.y, pt.z);
    local_pt.x = x_axis.dot(e_pt);
    local_pt.y = y_axis.dot(e_pt);
  }
```

上述cloud变换再次验证了世界坐标系也是ENU坐标系类型的说话，局部ENU坐标系和lidar坐标系共原点，但是有一个旋转角度，lidar坐标系到世界坐标系变换的旋转矩阵是vel_rot，那么到局部ENU坐标系的旋转矩阵也是vel_rot，而且原点相同，所以两个坐标系的平移矩阵其实是0。最终变换到局部ENU坐标系的公式就是P_hat = vec_rot * P + 0。也就是上面看到的公式。

另外补充一点猜测世界坐标系也是ENU类型坐标系的证据：

```
/// file in apollo/modules/perception/traffic_light/onboard/hdmap_input.cc
bool HDMapInput::GetSignals(const Eigen::Matrix4d &pointd, std::vector<apollo::hdmap::Signal> *signals) {
  auto hdmap = HDMapUtil::BaseMapPtr();
  std::vector<hdmap::SignalInfoConstPtr> forward_signals;
  apollo::common::PointENU point;
  point.set_x(pointd(0, 3));
  point.set_y(pointd(1, 3));
  point.set_z(pointd(2, 3));
  int result = hdmap->GetForwardNearestSignalsOnLane( point, FLAGS_query_signal_range, &forward_signals);
  ...
}
```

在交通信号灯感知模块中，有一个功能是根据当前车的位置，去查询高精地图，获取该位置处的信号灯信息。上面的代码中GetSignals函数实现了这个功能，输入pointd就是我们上述使用到的lidar2world_trans变换矩阵，下面的代码point是PointENU类型的点，并设置为变换矩阵的平移向量去世界坐标系查询(实际是去查询是世界坐标系下面车坐标对应位置的信号灯)。因此可以初步判断，世界坐标系是ENU类型坐标系。最后有个问题，转换到局部ENU坐标系有什么作用，以及效果。简单地说一下ENU坐标系是带有方向性的，所以转换到该坐标系下的polygeons_world和cloud_local其实是有东南西北的意思，从Apollo官方文档可以看到效果如下，得到当前位置下ROI区域内外的一个矩形框，这时候路面和路口就有东西南北走向的意义。

![img](https://github.com/YannZyl/Apollo-Note/blob/master/images/roi_lookup_table.png)

思考：为什么不在车辆坐标系(IMU坐标系)或者lidar坐标系下面进行接下去的分割操作？(这两个坐标系的方向都参考车头的方向，是没有东南西北这些地理位置信息的)。