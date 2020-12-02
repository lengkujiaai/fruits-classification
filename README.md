# fruits-classification
在jetson nano上根据水果照片训练神经网络，得出网络模型后对水果照片进行分类

作者：lengkujiaai@126.com

在jetson nano上通过tensorflow利用迁移学习对水果进行分类。

已经在jetsonnano上进行了测试，在其他平台上也可以运行，个别时候可能需要改动。

如果你是一个初学者，在继续之前，你应该先学一下Hello AI World的例子（ https://github.com/dusty-nv/jetson-inference ）。这样会自动安装很多依赖，从而避免了很多不便，在后面使用时也不用再担心。如果你要制作自己的数据集，你也需要这样。

在继续之前，你需要先make并mount一个交换分区，最少4GB。这样会让你的旅程更顺利，并且在这里也是必须的。更多细节参考 https://docs.rackspace.com/support/how-to/create-a-linux-swap-file/


sudo fallocate -l 4G /mnt/4GB.swap
sudo mkswap /mnt/4GB.swap
sudo swapon /mnt/4GB.swap


如果想让上述改变永久生效，你需要在/etc/fstab文件结尾增加：

/mnt/4GB.swap none swap sw 0 0

可以通过命令sudo tedgrastats来确定是否成功建立交换分区。


可以通过下面的命令克隆本项目代码：

git clone https://github.com/--------.git

对任何物体分类都需要一个数据集。本代码库的数据集共6类水果600张图片，每类100张。这里用的是Hello AI World 示例中的camera-capture工具获取图片的。等有时间，我专门增加一个获取图片的文件放到本代码库中。

你可以在fruit-dataset文件夹中发现分类的图片；你可以用这个数据集，也可以用网络上其他的数据集或者自己制作的数据集。我后面写一个文件教你制作自己的数据集。

# 本数据集中有如下类：
Apple/

Beetroot/

Dates/

Mango/

Orange/

Pomegranate/

# 制作你自己的数据集

使用tensorflow再次训练自己的模型：

Python3 retrain/retrain.py –image_dir fruits-dataset/

原作者的代码是不能运行的，我已经做了修改，可以放心运行，如果第一遍失败，再运行一下试试

# 推理：
你需要一些图片测试模型，在test-images文件夹下面。你可以用事先训练好的或者你自己训练的模型，在目录model下面。假定你在目录jetson-fruits-classification下面，只需要把你要测试的模型复制到retrain目录下面并运行下面的命令就行：

python3 retrain/label_image.py --graph=retrain/output_graph.pb --labels=retrain/output_labels.txt --input_layer=Placeholder --output_layer=final_result --image=test-images/apple.jpeg

如果想用存放于其他位置的模型，你需要把路径填对。稍等一下，你应该能看到类似下面的输出：

apple 0.9856818

orange 0.005912187

dates 0.002886865

pomegranate 0.0026501939

mango 0.0014653547

准确识别出图片是苹果。现在你可以试试其他的图片，来自test-images文件夹或网络的图片或其他位置的图片。

# 进一步提升：
为了降低推理的时间，可以用TensorRT来优化一下。

再训练脚本默认使用的是Inception_v3，也可以用更轻量级的模型结构来提升在jetson nano上推理的速度。


# 另：

1、2020-11-12，现供职于北京中电科卫星导航系统有限公司，本部门为研发中心。

2、公司在淘宝销售nvidia jetson 系列的产品，包括jetson nano，     TX1,     TX2,    AGX XAVIER,        XAVIER NX产品

3、我们属于提供技术支持的，本项目就是一老师要求的功能。

4、复制链接：   

    2.0fυィ直信息₰gyi7clU3sNj₤回t~bao或點几url链 https://m.tb.cn/h.4WAPC9j?sm=19844c 至浏览er【北京中电科卫星导航公司】
    
后打开淘宝即可

技术支持：

![image](https://github.com/lengkujiaai/wearing_mask_or_not_jetsonNano/blob/main/images/%E5%85%AC%E5%8F%B8%E4%BA%A7%E5%93%81.png)
