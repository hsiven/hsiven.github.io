---
layout: post
title: 无人机中常用的坐标系介绍
---
最近使用opencv进行3D重建，获得了基于左摄像机坐标系的三维坐标，然后需要将这个三维坐标系转换到机体NED坐标系坐标和GPS坐标。这个过程比较绕，涉及到关于无人机的几个参考坐标系，在这里简单的总结一下。 本文主要参考《Unmanned Rotorcraft Systems》第二章，Coordinate Systems and Transformations，相当于一个简单的学习摘要。 


# 一、五种常见坐标系
1.大地测量坐标系  

大地测量坐标系主要应用于GPS导航中。它并不是笛卡尔坐标系，而是使用经度，纬度，以及高度来描述地球表面上的点，并分别使用λ, ϕ, 和 h   来表示。经度的范围是-180° 到 180°，纬度的范围是-90°到正90°。高度则是测量点和参考椭圆体的垂直高度。使用下标g标注。  
<img src="https://raw.githubusercontent.com/hsiven/MarkdownPhotos/master/blog_6/blog_6_2.1.png">

和大地测量坐标系相关的一些参数包括，长半轴，短半轴，平坦因子，第一偏心率，子午线曲率半径，以及主曲率半径等。  
<img src="https://raw.githubusercontent.com/hsiven/MarkdownPhotos/master/blog_6/blog_6_2.2.png">


2.地固地心坐标系  
原点是地球中心。z轴是地球的旋转轴，正方向为指向北极的方向。x轴是地心和经度为0，纬度为0点相连得到的直线。然后通过右手法则就可以得到Y轴。使用下标e表示。
<img src="https://raw.githubusercontent.com/hsiven/MarkdownPhotos/master/blog_6/blog_6_2.3.png">


3.本地NED坐标系  
本地NED坐标系也被称为导航坐标系，或者地表坐标系。原点固定在地球表面。X轴指向北，Y轴指向东，z轴和重力加速度方向相同。小型UAV的导航一般都用这个坐标系来导航。使用下标n表示。
<img src="https://raw.githubusercontent.com/hsiven/MarkdownPhotos/master/blog_6/blog_6_2.4.png">


4.机体NED坐标系  
原点是机体的质心。X轴指向北，Y轴指向东，Z轴垂直向下。可以近似认为机载NED坐标系的轴和本地NED坐标系的轴方向是相同的。用下标nv表示。
<img src="https://raw.githubusercontent.com/hsiven/MarkdownPhotos/master/blog_6/blog_6_2.5.png">


5.机体坐标系  
原点是机体质心。X轴指向机头方向，Y轴是机翼方向。z轴根据右手法则获取，垂直向下。使用下标b表示。

# 二、坐标系之间相互转换
各个坐标系之间可以相互转换，这里不涉及欧拉角、四元数、旋转矩阵等基础知识，只是在这里记录这五种坐标系之间的转换，作为记录。  
1.大地测量坐标系和地固地心坐标系之间的转换  
转换公式如下：
<img src="https://raw.githubusercontent.com/hsiven/MarkdownPhotos/master/blog_6/blog_6_2.6.png">


2.地固地心坐标系和本地NED坐标系的转换  
转换公式如下：
<img src="https://raw.githubusercontent.com/hsiven/MarkdownPhotos/master/blog_6/blog_6_2.7.png">

其中Rn/e表示从地固地心坐标系到本地NED坐标系的旋转矩阵，λref和φref表示本地NED坐标系原点的经纬度。

3.大地测量坐标系和机体NED坐标系之间的转换  
这里给出速度的转换关系。
<img src="https://raw.githubusercontent.com/hsiven/MarkdownPhotos/master/blog_6/blog_6_2.8.png">

4.机体NED坐标系和机体坐标系  
机体NED坐标系和机体坐标系之间的关系，可以通过姿态解算得到的旋转矩阵进行转换。

5.本地NED坐标系和机体NED坐标系  
可以近似因为本地NED坐标系和机体NED坐标系的坐标轴方向相同，所以在任一时刻他们的速度、加速度近似相等。他们之间的坐标关系只要得到机体质心（也就是机体NED坐标系的原点）在本地NED坐标系的坐标即可计算得到。