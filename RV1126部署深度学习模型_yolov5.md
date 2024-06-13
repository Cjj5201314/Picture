# RV1126部署yolov5模型

## 环境准备

#### 安装adb

[windows下载安装adb（极其简单）_adb工具下载windows-CSDN博客](https://blog.csdn.net/x2584179909/article/details/108319973)



#### 安装RKNN-Toolkit工具包

下载链接https://github.com/rockchip-linux/rknn-toolkit/releases

安装教程https://blog.csdn.net/sazass/article/details/127127091

下载依赖

进入`rknn-toolkit-master`项目包的`packages`目录，根据需求下载`requirements-gpu.txt`或者`requirements-cpu.txt`。

安装RKNN-Toolkit工具包遇到无法编译opencv-python时，降低版本

```
pip install opencv-python==3.4.9.31
```

安装完之后测试是否安装成功

```
python
from rknn.api import RKNN
```



#### 在电脑上运行仿真示例

下载整个rknn_toolkit工具包

```
git clone https://github.com/rockchip-linux/rknn-toolkit
```

下载完后，执行一下命令

```
cd /examples/tflite/mobilenet_v1

python3 test.py
```

结果如下：

![](https://cdn.jsdelivr.net/gh/Cjj5201314/Picture@main/Data/Pictures/3229836506167fc3430da9279ad8a1d.png)



#### 安装交叉编译工具链

1、官网下载https://developer.arm.com/downloads/-/gnu-a

RV1126是32位的

![](https://cdn.jsdelivr.net/gh/Cjj5201314/Picture@main/Data/Pictures/e4369c8704c4067dac0b356e046632e.png)

2、接着是解压

```
xz -d gcc-arm-8.3-2019.03-x86_64-arm-linux-guneabihf.tar.xz

chmod 777 gcc-arm-8.3-2019.03-x86_64-arm-linux-guneabihf.tar

tar -xavf gcc-arm-8.3-2019.03-x86_64-arm-linux-guneabihf.tar
```

3、然后添加环境变量

```
vim ~/.bashrc

export PATH=/data/cjj/rk1126_20211014/gcc-arm-8.9-2019.03-x86_64-arm-linux-guneabihf/bir:$PATH

source ~/.bashrc
```



## yolov5转成RKNN模型

#### 转换方法

将yolov5的pt模型转成RKNN模型方法，分为两步：

1. 使用yolov5工程中自带的export.py将pt模型转成onnx模型

2. 利用这个脚本生成rknn模型

   https://github.com/rockchip-linux/rknpu/blob/master/rknn/rknn_api/examples/rknn_yolov5_demo/convert_rknn_demo/yolov5/pytorch2rknn.py



yolov5模型下载链接https://github.com/ultralytics/yolov5?tab=readme-ov-file

输入下面命令即可转成onnx类型

```
python3 export.py --weights yolov5x.pt --img 640 --batch 1 --opset 12 --include onnx
```

![](https://cdn.jsdelivr.net/gh/Cjj5201314/Picture@main/Data/Pictures/e9fe9e77495c9aebf37f23870c86d49.png)



运行转rknn模型的脚本

```
python3 pytorch2rknn.py
```

成功之后会在运行脚本所在目录下生成yolov5s_relu_rv1126_out_opt.rknn，放到**netron.app**检查模型结构

![image-20240606132150094](../AppData/Roaming/Typora/typora-user-images/image-20240606132150094.png)

## 在RV1126上推理RKNN模型

参考文献https://blog.csdn.net/kxh123456/article/details/129370265

要求：这里需要在ubuntu虚拟机上的镜像换成官方的sdk

![](https://cdn.jsdelivr.net/gh/Cjj5201314/Picture@main/Data/Pictures/afcb7577e8cc68b3a677a914a20d046.png)

#### 编译源码

1、下载交叉编译器

下载地址：https://releases.linaro.org/components/toolchain/binaries/6.4-2017.08/arm-linux-gnueabihf/

![](https://cdn.jsdelivr.net/gh/Cjj5201314/Picture@main/Data/Pictures/0b97f752c1bd7b6e6528eea5f25b8de.png)



2、修改build.sh内容

在原【build.sh】的基础上，只需添加交叉编译器的路径即可【RV1126_TOOL_CHAIN】，即可编译程序。

```
#!/bin/bash

set -e

# for rk1808 aarch64
# GCC_COMPILER=${RK1808_TOOL_CHAIN}/bin/aarch64-linux-gnu

# for rk1806 armhf
# GCC_COMPILER=~/opts/gcc-linaro-6.3.1-2017.05-x86_64_arm-linux-gnueabihf/bin/arm-linux-gnueabihf

# for rv1109/rv1126 armhf
# 自己添加的编译器路径【RV1126_TOOL_CHAIN】
RV1126_TOOL_CHAIN='/home/Cjj/gcc-linaro-6.4.1-2017.08-i686_arm-linux-gnueabihf'
GCC_COMPILER=${RV1126_TOOL_CHAIN}/bin/arm-linux-gnueabihf

ROOT_PWD=$( cd "$( dirname $0 )" && cd -P "$( dirname "$SOURCE" )" && pwd )

# build rockx
BUILD_DIR=${ROOT_PWD}/build

if [[ ! -d "${BUILD_DIR}" ]]; then
  mkdir -p ${BUILD_DIR}
fi

cd ${BUILD_DIR}
cmake .. \
    -DCMAKE_C_COMPILER=${GCC_COMPILER}-gcc \
    -DCMAKE_CXX_COMPILER=${GCC_COMPILER}-g++
make -j4
make install
cd -

# cp run_rk180x.sh install/rknn_yolov5_demo/
cp run_rv1109_rv1126.sh install/rknn_yolov5_demo/
```



3、编译yolov5文件夹

![](https://cdn.jsdelivr.net/gh/Cjj5201314/Picture@main/Data/Pictures/1f2fe5e2e9df3c96dc6ab1d39b9eb3d.png)

```
cd ~/RV1126/rknpu-master/rknpu-master/rknn/rknn_api/examples/rknn_yolov5_demo
./build.sh
```



4、将编译好的文件夹传给RV1126开发板，并运行

```
在ubuntu传文件给RV1126
adb push rknn_yolov5_demo /

到指定目录下运行rknn_rv1109_rv1126.sh脚本
cd /home/rknn_yolov5_demo
./rknn_rv1109_rv1126.sh

把RV1126文件传回给ubuntu
adb pull /home/rknn_yolov5_demo/out.bmp ./
```

RV1126推理yolov5效果如下：

![](https://cdn.jsdelivr.net/gh/Cjj5201314/Picture@main/Data/Pictures/9e79df1cb448335831ccd508208d41b.png)



5、RV1126部署yolov5演示视频

<video src="[/videos/your-video-filename.mp4](https://github.com/Cjj5201314/Picture/blob/main/Data/Videoes/RV1126%E6%88%90%E6%9E%9C%E6%BC%94%E7%A4%BA.mp4)" autoplay="true" controls="controls" width="800" height="600">
</video>


