# 点云关键点的 特征描述

    如果要对一个三维点云进行描述，光有点云的位置是不够的，常常需要计算一些额外的参数，
    比如法线方向、曲率、文理特征、颜色、领域中心距、协方差矩阵、熵等等。
    如同图像的特征（sifi surf orb）一样，我们需要使用类似的方式来描述三维点云的特征。

    常用的特征描述算法有：
    法线和曲率计算 normal_3d_feature 、
    特征值分析、
    PFH  点特征直方图描述子 nk2、
    FPFH 快速点特征直方图描述子 FPFH是PFH的简化形式 nk、
    3D Shape Context、 文理特征
    Spin Image
    VFH视点特征直方图(Viewpoint Feature Histogram)
    NARF关键点  pcl::NarfKeypoint narf特征 pcl::NarfDescriptor
    RoPs特征(Rotational Projection Statistics) 
    (GASD）全局一致的空间分布描述子特征 Globally Aligned Spatial Distribution (GASD) descriptors


## 【1 】估计表面法线的解决方案就变成了分析一个协方差矩阵的特征矢量和特征值
    （或者PCA—主成分分析），这个协方差矩阵从查询点的近邻元素中创建。
    更具体地说，对于每一个点Pi,对应的协方差矩阵。
    #include <pcl/features/normal_3d.h>//法线特征
    --------------------------------------------------------------
    // 创建法线估计类====================================
      pcl::NormalEstimation<pcl::PointXYZ, pcl::Normal> ne;
      ne.setInputCloud (cloud_ptr);
    // 添加搜索算法 kdtree search  最近的几个点 估计平面 协方差矩阵PCA分解 求解法线
      pcl::search::KdTree<pcl::PointXYZ>::Ptr tree (new pcl::search::KdTree<pcl::PointXYZ> ());
      ne.setSearchMethod (tree);
      // 输出点云 带有法线描述
      pcl::PointCloud<pcl::Normal>::Ptr cloud_normals_ptr (new pcl::PointCloud<pcl::Normal>);
      pcl::PointCloud<pcl::Normal>& cloud_normals = *cloud_normals_ptr;
      ne.setRadiusSearch (0.03);//半径内搜索临近点
      //ne.setKSearch(8);       //其二 指定临近点数量
      // 计算表面法线特征
      ne.compute (cloud_normals);

    -------------------------------------------------------------
## 【2】使用积分图计算一个有序点云的法线，注意该方法只适用于有序点云
    表面法线是几何体表面的重要属性，在很多领域都有大量应用，
    例如：在进行光照渲染时产生符合可视习惯的效果时需要表面法线信息才能正常进行，
    对于一个已知的几何体表面，根据垂直于点表面的矢量，因此推断表面某一点的法线方向通常比较简单。
    头文件
    #include <pcl/features/integral_image_normal.h>
    ---------------------------------------------------------
      // 估计的法线 normals
      pcl::PointCloud<pcl::Normal>::Ptr normals_ptr (new pcl::PointCloud<pcl::Normal>);
      pcl::PointCloud<pcl::Normal>& normals = *normals_ptr;
            // 积分图像方法 估计点云表面法线 
      pcl::IntegralImageNormalEstimation<pcl::PointXYZ, pcl::Normal> ne;

    /*
    预测方法
    enum NormalEstimationMethod
    {
      COVARIANCE_MATRIX, 从最近邻的协方差矩阵创建了9个积分图去计算一个点的法线
      AVERAGE_3D_GRADIENT, 创建了6个积分图去计算3D梯度里面竖直和水平方向的光滑部分，同时利用两个梯度的卷积来计算法线。
      AVERAGE_DEPTH_CHANGE 造了一个单一的积分图，从平均深度的变化中来计算法线。
    };
    */ 
      ne.setNormalEstimationMethod (ne.AVERAGE_3D_GRADIENT);
      ne.setMaxDepthChangeFactor(0.02f);
      ne.setNormalSmoothingSize(10.0f);
      ne.setInputCloud(cloud_ptr);
      ne.compute(normals);

    --------------------------------------------

##  【3】 点特征直方图(PFH)描述子
    计算法线---计算临近点对角度差值-----直方图--
    点特征直方图(PFH)描述子
    点特征直方图(Point Feature Histograms)
    正如点特征表示法所示，表面法线和曲率估计是某个点周围的几何特征基本表示法。
    虽然计算非常快速容易，但是无法获得太多信息，因为它们只使用很少的
    几个参数值来近似表示一个点的k邻域的几何特征。然而大部分场景中包含许多特征点，
    这些特征点有相同的或者非常相近的特征值，因此采用点特征表示法，
    其直接结果就减少了全局的特征信息。

    http://www.pclcn.org/study/shownews.php?lang=cn&id=101

    是基于点与其k邻域之间的关系以及它们的估计法线，
    简言之，它考虑估计法线方向之间所有的相互作用，
    试图捕获最好的样本表面变化情况，以描述样本的几何特征。

    每一点对，原有12个参数，6个坐标值，6个坐标姿态（基于法线）
    PHF计算没一点对的 相对坐标角度差值三个值和 坐标点之间的欧氏距离 d
    从12个参数减少到4个参数
    默认PFH的实现使用5个区间分类（例如：四个特征值中的每个都使用5个区间来统计），
    其中不包括距离（在上文中已经解释过了——但是如果有需要的话，
    也可以通过用户调用computePairFeatures方法来获得距离值），
    这样就组成了一个125浮点数元素的特征向量（15），
    其保存在一个pcl::PFHSignature125的点类型中。

    头文件
    #include <pcl/features/pfh.h>
    #include <pcl/features/normal_3d.h>//法线特征
    ---------------------------------------------------
    // =====计算法线========创建法线估计类====================================
      pcl::NormalEstimation<pcl::PointXYZ, pcl::Normal> ne;
      ne.setInputCloud (cloud_ptr);

    // 添加搜索算法 kdtree search  最近的几个点 估计平面 协方差矩阵PCA分解 求解法线
      pcl::search::KdTree<pcl::PointXYZ>::Ptr tree (new pcl::search::KdTree<pcl::PointXYZ> ());
      ne.setSearchMethod (tree);//设置近邻搜索算法 
      // 输出点云 带有法线描述
      pcl::PointCloud<pcl::Normal>::Ptr cloud_normals_ptr (new pcl::PointCloud<pcl::Normal>);
      pcl::PointCloud<pcl::Normal>& cloud_normals = *cloud_normals_ptr;
      // Use all neighbors in a sphere of radius 3cm
      ne.setRadiusSearch (0.03);//半价内搜索临近点 3cm
      // 计算表面法线特征
      ne.compute (cloud_normals);

    //=======创建PFH估计对象pfh，并将输入点云数据集cloud和法线normals传递给它=================
      pcl::PFHEstimation<pcl::PointXYZ, pcl::Normal, pcl::PFHSignature125> pfh;// phf特征估计其器
      pfh.setInputCloud (cloud_ptr);
      pfh.setInputNormals (cloud_normals_ptr);
     //如果点云是类型为PointNormal,则执行pfh.setInputNormals (cloud);
     //创建一个空的kd树表示法，并把它传递给PFH估计对象。
     //基于已给的输入数据集，建立kdtree
      pcl::search::KdTree<pcl::PointXYZ>::Ptr tree2 (new pcl::search::KdTree<pcl::PointXYZ> ());
      //pcl::KdTreeFLANN<pcl::PointXYZ>::Ptr tree2 (new pcl::KdTreeFLANN<pcl::PointXYZ> ()); //-- older call for PCL 1.5-
      pfh.setSearchMethod (tree2);//设置近邻搜索算法 
      //输出数据集
      pcl::PointCloud<pcl::PFHSignature125>::Ptr pfh_fe_ptr (new pcl::PointCloud<pcl::PFHSignature125> ());//phf特征
     //使用半径在5厘米范围内的所有邻元素。
      //注意：此处使用的半径必须要大于估计表面法线时使用的半径!!!
      pfh.setRadiusSearch (0.05);
      //计算pfh特征值
      pfh.compute (*pfh_fe_ptr);
    ----------------------------------------------------------------------
## 【4】phf点特征直方图 计算复杂度还是太高
    计算法线---计算临近点对角度差值-----直方图--
    因此存在一个O(nk^2) 的计算复杂性。
    k个点之间相互的点对 k×k条连接线

    每一点对，原有12个参数，6个坐标值，6个坐标姿态（基于法线）
    PHF计算没一点对的 相对坐标角度差值三个值和 坐标点之间的欧氏距离 d
    从12个参数减少到4个参数

    快速点特征直方图FPFH（Fast Point Feature Histograms）把算法的计算复杂度降低到了O(nk) ，
    但是任然保留了PFH大部分的识别特性。
    查询点和周围k个点的连线 的4参数特征
    也就是1×k=k个线

    默认的FPFH实现使用11个统计子区间（例如：四个特征值中的每个都将它的参数区间分割为11个），
    特征直方图被分别计算然后合并得出了浮点值的一个33元素的特征向量，
    这些保存在一个pcl::FPFHSignature33点类型中。

    头文件
    #include <pcl/features/fpfh.h>
    #include <pcl/features/normal_3d.h>//法线特征
    ----------------------------------------------------
    // =====计算法线========创建法线估计类====================================
      pcl::NormalEstimation<pcl::PointXYZ, pcl::Normal> ne;
      ne.setInputCloud (cloud_ptr);
    /*
     法线估计类NormalEstimation的实际计算调用程序内部执行以下操作：
    对点云P中的每个点p
      1.得到p点的最近邻元素
      2.计算p点的表面法线n
      3.检查n的方向是否一致指向视点，如果不是则翻转

     在PCL内估计一点集对应的协方差矩阵，可以使用以下函数调用实现：
    //定义每个表面小块的3x3协方差矩阵的存储对象
    Eigen::Matrix3fcovariance_matrix;
    //定义一个表面小块的质心坐标16-字节对齐存储对象
    Eigen::Vector4fxyz_centroid;
    //估计质心坐标
    compute3DCentroid(cloud,xyz_centroid);
    //计算3x3协方差矩阵
    computeCovarianceMatrix(cloud,xyz_centroid,covariance_matrix);
    */
    // 添加搜索算法 kdtree search  最近的几个点 估计平面 协方差矩阵PCA分解 求解法线
      pcl::search::KdTree<pcl::PointXYZ>::Ptr tree (new pcl::search::KdTree<pcl::PointXYZ> ());
      ne.setSearchMethod (tree);//设置近邻搜索算法 
        // 输出点云 带有法线描述
      pcl::PointCloud<pcl::Normal>::Ptr cloud_normals_ptr (new pcl::PointCloud<pcl::Normal>);
      pcl::PointCloud<pcl::Normal>& cloud_normals = *cloud_normals_ptr;
        // Use all neighbors in a sphere of radius 3cm
      ne.setRadiusSearch (0.03);//半价内搜索临近点 3cm
        // 计算表面法线特征
      ne.compute (cloud_normals);

    //=======创建FPFH估计对象fpfh, 并将输入点云数据集cloud和法线normals传递给它=================
        //pcl::PFHEstimation<pcl::PointXYZ, pcl::Normal, pcl::PFHSignature125> pfh;// phf特征估计其器
      pcl::FPFHEstimation<pcl::PointXYZ,pcl::Normal,pcl::FPFHSignature33> fpfh;
        // pcl::FPFHEstimationOMP<pcl::PointXYZ,pcl::Normal,pcl::FPFHSignature33> fpfh;//多核加速
      fpfh.setInputCloud (cloud_ptr);
      fpfh.setInputNormals (cloud_normals_ptr);
       //如果点云是类型为PointNormal,则执行pfh.setInputNormals (cloud);
       //创建一个空的kd树表示法，并把它传递给PFH估计对象。
       //基于已给的输入数据集，建立kdtree
      pcl::search::KdTree<pcl::PointXYZ>::Ptr tree2 (new pcl::search::KdTree<pcl::PointXYZ> ());
        //pcl::KdTreeFLANN<pcl::PointXYZ>::Ptr tree2 (new pcl::KdTreeFLANN<pcl::PointXYZ> ()); //-- older call for PCL 1.5-
      fpfh.setSearchMethod (tree2);//设置近邻搜索算法 
        //输出数据集
        //pcl::PointCloud<pcl::PFHSignature125>::Ptr pfh_fe_ptr (new pcl::PointCloud<pcl::PFHSignature125> ());//phf特征
      pcl::PointCloud<pcl::FPFHSignature33>::Ptr fpfh_fe_ptr (new pcl::PointCloud<pcl::FPFHSignature33>());//fphf特征
        //使用半径在5厘米范围内的所有邻元素。
        //注意：此处使用的半径必须要大于估计表面法线时使用的半径!!!
      fpfh.setRadiusSearch (0.05);
        //计算pfh特征值
      fpfh.compute (*fpfh_fe_ptr);
    ---------------------------------------------------------------


## 【5】视点特征直方图VFH(Viewpoint Feature Histogram)描述子，
    它是一种新的特征表示形式，应用在点云聚类识别和六自由度位姿估计问题。

    视点特征直方图（或VFH）是源于FPFH描述子.
    由于它的获取速度和识别力，我们决定利用FPFH强大的识别力，
    但是为了使构造的特征保持缩放不变性的性质同时，
    还要区分不同的位姿，计算时需要考虑加入视点变量。

    我们做了以下两种计算来构造特征，以应用于目标识别问题和位姿估计：
    1.扩展FPFH，使其利用整个点云对象来进行计算估计（如2图所示），
    在计算FPFH时以物体中心点与物体表面其他所有点之间的点对作为计算单元。

    2.添加视点方向与每个点估计法线之间额外的统计信息，为了达到这个目的，
    我们的关键想法是在FPFH计算中将视点方向变量直接融入到相对法线角计算当中。

    因此新组合的特征被称为视点特征直方图（VFH）。
    下图表体现的就是新特征的想法，包含了以下两部分：

    1.一个视点方向相关的分量
    2.一个包含扩展FPFH的描述表面形状的分量

    对扩展的FPFH分量来说，默认的VFH的实现使用45个子区间进行统计，
    而对于视点分量要使用128个子区间进行统计，这样VFH就由一共308个浮点数组成阵列。
    在PCL中利用pcl::VFHSignature308的点类型来存储表示。P
    FH/FPFH描述子和VFH之间的主要区别是：

    对于一个已知的点云数据集，只一个单一的VFH描述子，
    而合成的PFH/FPFH特征的数目和点云中的点数目相同。

    头文件
    #include <pcl/features/vfh.h>
    #include <pcl/features/normal_3d.h>//法线特征

    ------------------------------------------------------
    // =====计算法线========创建法线估计类====================================
      pcl::NormalEstimation<pcl::PointXYZ, pcl::Normal> ne;
      ne.setInputCloud (cloud_ptr);

    // 添加搜索算法 kdtree search  最近的几个点 估计平面 协方差矩阵PCA分解 求解法线
      pcl::search::KdTree<pcl::PointXYZ>::Ptr tree (new pcl::search::KdTree<pcl::PointXYZ> ());
      ne.setSearchMethod (tree);//设置近邻搜索算法 
      // 输出点云 带有法线描述
      pcl::PointCloud<pcl::Normal>::Ptr cloud_normals_ptr (new pcl::PointCloud<pcl::Normal>);
      pcl::PointCloud<pcl::Normal>& cloud_normals = *cloud_normals_ptr;
      // Use all neighbors in a sphere of radius 3cm
      ne.setRadiusSearch (0.03);//半价内搜索临近点 3cm
      // 计算表面法线特征
      ne.compute (cloud_normals);


    //=======创建VFH估计对象vfh，并把输入数据集cloud和法线normal传递给它================
      pcl::VFHEstimation<pcl::PointXYZ, pcl::Normal, pcl::VFHSignature308> vfh;
      //pcl::PFHEstimation<pcl::PointXYZ, pcl::Normal, pcl::PFHSignature125> pfh;// phf特征估计其器
      //pcl::FPFHEstimation<pcl::PointXYZ,pcl::Normal,pcl::FPFHSignature33> fpfh;// fphf特征估计其器
      // pcl::FPFHEstimationOMP<pcl::PointXYZ,pcl::Normal,pcl::FPFHSignature33> fpfh;//多核加速
      vfh.setInputCloud (cloud_ptr);
      vfh.setInputNormals (cloud_normals_ptr);
      //如果点云是PointNormal类型，则执行vfh.setInputNormals (cloud);
      //创建一个空的kd树对象，并把它传递给FPFH估计对象。
      //基于已知的输入数据集，建立kdtree
      pcl::search::KdTree<pcl::PointXYZ>::Ptr tree2 (new pcl::search::KdTree<pcl::PointXYZ> ());
      //pcl::KdTreeFLANN<pcl::PointXYZ>::Ptr tree2 (new pcl::KdTreeFLANN<pcl::PointXYZ> ()); //-- older call for PCL 1.5-
      vfh.setSearchMethod (tree2);//设置近邻搜索算法 
      //输出数据集
      //pcl::PointCloud<pcl::PFHSignature125>::Ptr pfh_fe_ptr (new pcl::PointCloud<pcl::PFHSignature125> ());//phf特征
      //pcl::PointCloud<pcl::FPFHSignature33>::Ptr fpfh_fe_ptr (new pcl::PointCloud<pcl::FPFHSignature33>());//fphf特征
      pcl::PointCloud<pcl::VFHSignature308>::Ptr vfh_fe_ptr (new pcl::PointCloud<pcl::VFHSignature308> ());//vhf特征
      //使用半径在5厘米范围内的所有邻元素。
      //注意：此处使用的半径必须要大于估计表面法线时使用的半径!!!
      //fpfh.setRadiusSearch (0.05);
      //计算pfh特征值
      vfh.compute (*vfh_fe_ptr);

    ----------------------------------------------------------

## 【6】NARF　
    从深度图像(RangeImage)中提取NARF关键点  pcl::NarfKeypoint   
     然后计算narf特征 pcl::NarfDescriptor
    边缘提取
    直接把三维的点云投射成二维的图像不就好了。
    这种投射方法叫做range_image.

    #include <pcl/range_image/range_image.h>// RangeImage 深度图像
    #include <pcl/keypoints/narf_keypoint.h>// narf关键点检测
    #include <pcl/features/narf_descriptor.h>// narf特征

    ------------------------------------------------------

    // ======从点云数据，创建深度图像=====================
      // 直接把三维的点云投射成二维的图像
      float noise_level = 0.0;
    //noise level表示的是容差率，因为1°X1°的空间内很可能不止一个点，
    //noise level = 0则表示去最近点的距离作为像素值，如果=0.05则表示在最近点及其后5cm范围内求个平均距离
    //minRange表示深度最小值，如果=0则表示取1°X1°的空间内最远点，近的都忽略
      float min_range = 0.0f;
    //bordersieze表示图像周边点 
      int border_size = 1;
      boost::shared_ptr<pcl::RangeImage> range_image_ptr (new pcl::RangeImage);//创建RangeImage对象（智能指针）
      pcl::RangeImage& range_image = *range_image_ptr; //RangeImage的引用  
    //半圆扫一圈就是整个图像了
     range_image.createFromPointCloud (point_cloud, angular_resolution, pcl::deg2rad (360.0f), pcl::deg2rad (180.0f),
                                       scene_sensor_pose, coordinate_frame, noise_level, min_range, border_size);
      range_image.integrateFarRanges (far_ranges);//整合远距离点云
      if (setUnseenToMaxRange)
        range_image.setUnseenToMaxRange ();

     // ====================提取NARF关键点=======================
      pcl::RangeImageBorderExtractor range_image_border_extractor;//创建深度图像的边界提取器，用于提取NARF关键点
      pcl::NarfKeypoint narf_keypoint_detector (&range_image_border_extractor);//创建NARF对象
      narf_keypoint_detector.setRangeImage (&range_image);//设置点云对应的深度图
      narf_keypoint_detector.getParameters ().support_size = support_size;// 感兴趣点的尺寸（球面的直径）
      //narf_keypoint_detector.getParameters ().add_points_on_straight_edges = true;
      //narf_keypoint_detector.getParameters ().distance_for_additional_points = 0.5;

      pcl::PointCloud<int> keypoint_indices;//用于存储关键点的索引 PointCloud<int>
      narf_keypoint_detector.compute (keypoint_indices);//计算NARF关键

    //========================提取 NARF 特征 ====================
      // ------------------------------------------------------
      std::vector<int> keypoint_indices2;//用于存储关键点的索引 vector<int>  
      keypoint_indices2.resize (keypoint_indices.points.size ());
      for (unsigned int i=0; i<keypoint_indices.size (); ++i) // This step is necessary to get the right vector type
        keypoint_indices2[i] = keypoint_indices.points[i];//narf关键点 索引
      pcl::NarfDescriptor narf_descriptor (&range_image, &keypoint_indices2);//narf特征描述子
      narf_descriptor.getParameters().support_size = support_size;
      narf_descriptor.getParameters().rotation_invariant = rotation_invariant;
      pcl::PointCloud<pcl::Narf36> narf_descriptors;
      narf_descriptor.compute (narf_descriptors);

    ---------------------------------------------------------------
## 【7】RoPs特征(Rotational Projection Statistics) 描述子
    0.在关键点出建立局部坐标系。
    1.在一个给定的角度在当前坐标系下对关键点领域(局部表面) 进行旋转
    2.把 局部表面 投影到 xy，yz，xz三个2维平面上
    3.在每个投影平面上划分不同的盒子容器，把点分到不同的盒子里
    4.根据落入每个盒子的数量，来计算每个投影面上的一系列数据分布
    （熵值，低阶中心矩
    5.M11,M12,M21,M22，E。E是信息熵。4*2+1=9）进行描述
    计算值将会组成子特征。
    盒子数量 × 旋转次数×9 得到特征维度

    我们把上面这些步骤进行多次迭代。不同坐标轴的子特征将组成RoPS描述器
    我们首先要找到目标模型:
    points 包含点云
    indices 点的下标
    triangles包含了多边形
    -------------------------------------

    #include <pcl/features/rops_estimation.h>
    ------------------------------------------------
      float support_radius = 0.0285f;//局部表面裁剪支持的半径 (搜索半价)，
      unsigned int number_of_partition_bins = 5;//以及用于组成分布矩阵的容器的数量
      unsigned int number_of_rotations = 3;//和旋转的次数。最后的参数将影响描述器的长度。

    //搜索方法
      pcl::search::KdTree<pcl::PointXYZ>::Ptr search_method (new pcl::search::KdTree<pcl::PointXYZ>);
      search_method->setInputCloud (cloud_ptr);
    // rops 特征算法 对象 盒子数量 × 旋转次数×9 得到特征维度  3*5*9 =135
      pcl::ROPSEstimation <pcl::PointXYZ, pcl::Histogram <135> > feature_estimator;
      feature_estimator.setSearchMethod (search_method);//搜索算法
      feature_estimator.setSearchSurface (cloud_ptr);//搜索平面
      feature_estimator.setInputCloud (cloud_ptr);//输入点云
      feature_estimator.setIndices (indices);//关键点索引
      feature_estimator.setTriangles (triangles);//领域形状
      feature_estimator.setRadiusSearch (support_radius);//搜索半径
      feature_estimator.setNumberOfPartitionBins (number_of_partition_bins);//盒子数量
      feature_estimator.setNumberOfRotations (number_of_rotations);//旋转次数
      feature_estimator.setSupportRadius (support_radius);// 局部表面裁剪支持的半径 

      pcl::PointCloud<pcl::Histogram <135> >::Ptr histograms (new pcl::PointCloud <pcl::Histogram <135> > ());
      feature_estimator.compute (*histograms);


    -------------------------------------
## 【8】momentofinertiaestimation类获取基于惯性偏心矩 描述子。
    这个类还允许提取云的轴对齐和定向包围盒。
    但请记住，不可能的最小提取OBB包围盒。

    首先计算点云的协方差矩阵，提取其特征值和向量。
    你可以认为所得的特征向量是标准化的，
    总是形成右手坐标系（主要特征向量表示x轴，而较小的向量表示z轴）。
    在下一个步骤中，迭代过程发生。每次迭代时旋转主特征向量。
    旋转顺序总是相同的，并且在其他特征向量周围执行，
    这提供了点云旋转的不变性。
    此后，我们将把这个旋转主向量作为当前轴。

    然后在当前轴上计算转动惯量 和 将点云投影到以旋转向量为法线的平面上
    计算偏心距

    ------------------------
    #include <pcl/features/moment_of_inertia_estimation.h>
    -------------------------------------------------------------------------------------
     pcl::MomentOfInertiaEstimation <pcl::PointXYZ> feature_extractor;// 惯量 偏心距 特征提取
      feature_extractor.setInputCloud (cloud_ptr);//输入点云
      feature_extractor.compute ();//计算

      std::vector <float> moment_of_inertia;// 惯量
      std::vector <float> eccentricity;// 偏心距
      pcl::PointXYZ min_point_AABB;
      pcl::PointXYZ max_point_AABB;
      pcl::PointXYZ min_point_OBB;
      pcl::PointXYZ max_point_OBB;
      pcl::PointXYZ position_OBB;
      Eigen::Matrix3f rotational_matrix_OBB;
      float major_value, middle_value, minor_value;
      Eigen::Vector3f major_vector, middle_vector, minor_vector;
      Eigen::Vector3f mass_center;

      feature_extractor.getMomentOfInertia (moment_of_inertia);// 惯量
      feature_extractor.getEccentricity (eccentricity);// 偏心距
    // 以八个点的坐标 给出 包围盒
      feature_extractor.getAABB (min_point_AABB, max_point_AABB);// AABB 包围盒坐标 八个点的坐标
    // 以中心点 姿态 坐标轴范围 给出   包围盒
      feature_extractor.getOBB (min_point_OBB, max_point_OBB, position_OBB, rotational_matrix_OBB);// 位置和方向姿态
      feature_extractor.getEigenValues (major_value, middle_value, minor_value);//特征值
      feature_extractor.getEigenVectors (major_vector, middle_vector, minor_vector);//特征向量
      feature_extractor.getMassCenter (mass_center);//点云中心点

    --------------------------------------------------------------------
## 【9】全局一致的空间分布描述子特征
    Globally Aligned Spatial Distribution (GASD) descriptors
    可用于物体识别和姿态估计。
    是对可以描述整个点云的参考帧的估计，
    这是用来对准它的正则坐标系统
    之后，根据其三维点在空间上的分布，计算出点云的描述符。
    种描述符也可以扩展到整个点云的颜色分布。
    匹配点云(icp)的全局对齐变换用于计算物体姿态。

    使用主成分分析PCA来估计参考帧
    三维点云P
    计算其中心点位置P_
    计算协方差矩阵 1/n × sum((pi - P_)*(pi - P_)转置)
    奇异值分解得到 其 特征值eigen values  和特征向量eigen vectors
    基于参考帧计算一个变换 [R t]
    ------------------------------------
    #include <pcl/features/gasd.h>
    --------------------------------------------------
    // 创建 GASD 全局一致的空间分布描述子特征 传递 点云
     // pcl::GASDColorEstimation<pcl::PointXYZRGBA, pcl::GASDSignature984> gasd;//包含颜色
      pcl::GASDColorEstimation<pcl::PointXYZ, pcl::GASDSignature984> gasd;
      gasd.setInputCloud (cloud_ptr);
      // 输出描述子
      pcl::PointCloud<pcl::GASDSignature984> descriptor;
      // 计算描述子
      gasd.compute (descriptor);
      // 得到匹配 变换
      Eigen::Matrix4f trans = gasd.getTransform();

    ---------------------------------------------------------------
