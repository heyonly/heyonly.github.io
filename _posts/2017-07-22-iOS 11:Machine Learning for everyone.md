---
layout: post
title: iOS 11:Machine Learning for everyone 
categories: translation
description: Machine Learning
keywords: Machine Learning,iOS Machine Learning
---

在刚刚过去的WWDC 2017上，一件很引人注目的事情：苹果全面进军机器学习。




并且苹果想尽办法尽可能的让更多的开发者加入。



去年苹果发布了[Metal CNN and BNNS frameworks](http://machinethink.net/blog/apple-deep-learning-bnns-versus-metal-cnn/)来创建基本卷积网络。今年，我们了解更多关于Metal，一个新的计算机视觉框架，另外还有Core ML:一个很容易将ML模型整合进你的应用中的工具包。


<div align="center">
<img src="/images/blog/CoreML.png" align="middle"/>
</div>


在这片博客中我将分享我对于机器学习在iOS 11 和macOS 10.13 上的一些想法和经验。



<h4>Core ML</h4>
很容易理解Core ML 在WWDC上为什么会成为焦点：这是一个大多数开发者都想将其加入到自己应用中。



它的API非常简单，你只需要做以下一些事情就可以：


1. 加载训练模型
2. 做出预测
3. 坐享其成


这听起来似乎很有限但是在实际操作中不，加载一个模型并且做出预测是你要将其加入你的应用中通常都要做的操作。



先前，加载一个训练模型是非常艰难的--事实上，我写了一个[静态库来减轻这些痛苦](https://github.com/hollance/Forge)。所以现在我非常高兴只需要简单的两部就可以达到目的。



这个模型包含在一个.mlmodel 文件中。这是一个在你的模型中描述层级的[可打开的文件格式](https://pypi.python.org/pypi/coremltools),输入和输出，类标签，和一些需要预处理的数据。这也包含所有的学习参数（权重和偏差）。




你需要使用这个文件中的模型来完成所有的事情。



你简单的将这个mlmodel 文件拖入到你的工程中，然后Xcode就会自动生成一个Swift 或者Objective-C 的封装类以便于你能真正的容易使用这个模型。



例如，如果你添加ResNet50.mlmodel 这个文件到你的工程中，你只需要这么写：



```
let model = ResNet50()
```

来实例化模型，然后接下来就是要做出预测：



```
let pixelBuffer: CVPixelBuffer = /* your image */

if let prediction = try? model.prediction(image: pixelBuffer) {
  print(prediction.classLabel)
}
```


干得非常漂亮。你不需要写太多代码来加载模型或者转换输出，这些都是Core ML 和Xcode关心的事情。



> 为了了解背后所发生的事情，你可以选中mlmodel文件然后点击导航栏中的按钮查看生成的swift 代码和帮助文档。



CoreML 会根据迷行来决定运行在CPU 上还是GPU上。它自己会根据可利用资源做出优化。CoreML 甚至能够将模型分开运行在GPU上（需要大量的运算的任务）和CPU上（需要大量的内存的任务）。



CoreML 能够运行在CPU上的一个好处是：你能够将其运行在iOS 模拟器中（这些事情使用Metal 是不可能完成的任务，也无法更好的进行单元测试）。



<h4>Core ML 支持什么模型</h4>
上面的ResNet50示例是一个图像分类器，除此之外Core ML可以处理几种不同类型的模型，例如：


* 支持向量机（SVM）
* 树组合例如随机数和增强树
* 线性回归和逻辑回归
* 神经网络：前馈神经网络，卷积神经网络，周期神经网络



所有这些都可以用于回归和分类。此外，您的模型可以包含典型的ML预处理步骤，如单热编码，功能缩放，缺失值插补等。



苹果提供了很多可以直接[下载使用](https://developer.apple.com/machine-learning/)的训练模型，例如Inception V3,ResNet50 和VGG16，但是你也可以使用[Core ML 工具](https://pypi.python.org/pypi/coremltools)转化你自己的模型。




现在有很多模型转化工具，例如：Keras，Caffeine，scikit-learn，XGBoost和libSVM。这些转化工具每一个版本都有一点小小的差别----例如Keras 1.2.2 能够正常运行但是2.0 却不能。很幸运的是这些工具都是开源的。毫无疑问将来会有更多的训练工具。



如果很不幸上面所有的工具都无法使用的话，你可以自己写一个转化工具☺。mlmodel 文件是可以打开和直接使用的（它的protobuf 格式和规格标准都是由苹果发布的）




<h4>局限性</h4>



Core ML 可以快速获取模型并且运行在你的应用中这一点来说是非常棒的。然而如此简单的API当然有一定的局限性


* 只支持监督机器学习类型的模型。不支持无监督学习算法或加强学习算法（尽管这也支持通用神经网络，但你可能不会使用于此）
* 在设备上无法进行训练。你需要将你的模型使用一些离线工具转化成CoreML 格式。
* 如果Core ML 不支持某种层类型，你将无法使用它。在这一点上，使用自己的计算内核不可能扩展Core ML，在使用TensorFlow这样的工具来构建通用计算图形时，mlmodel文件格式就不那么灵活了。
* Core ML转化工具仅仅支持有限的特定版本的训练工具。for example，加入你使用TensorFlow 训练 的一个模型，很不幸，你无法使用，这时候你就不得不自己写一个转换脚本。正如我刚才所提到：如果你使用TensorFlow 训练出来的模型是mlmodel 不支持的类型，你将无法使用与Core ML。
* 你无法观察中间层的输出，你只能获得最后的结果。
* 我不能百分百确定下载下来的更新版本的模型是有问题的。如果你经常需要重新训练模型并且向将模型集成到你的应用中时，你就不得不在每次更新模型后重新发布一个新版本的应用。
* Core ML 隐藏了运行在CPU上还是在GPU上-- 这是非常方便的--但是这也导致你无法确定你的应用正在做的事情是不是你想要的。你也无法强制Core ML运行在GPU上，即使你非常想要这么做。




如果您能够忍受这些限制，那么Core ML就是您应该选择的框架。



而如果您无法忍受这些限制或者说你想完全控制，那么您就要考虑是使用Metal Performance Shaders 还是Accelerate框架----或者是两者。


> 当然，在你的模型中，真正的私房菜不是Core ML，如果你不能正确使用Core ML，那么Core ML对于你而言，将是毫无用处的。设计和训练模型是机器学习中的非常艰难的一部分。



<h4>快速集成应用</h4>
我在GitHub上开源了一个集成Core ML 的demo工程[source code on GitHub](https://github.com/hollance/MobileNet-CoreML)


<div align="center">
<img src="/images/blog/Demo@2x.png" align="middle"/>
</div>



该demo工程使用 [MobileNet architecture](https://arxiv.org/abs/1704.04861v1)来分类猫的图片。



这个模型是使用 [Caffe 训练](https://github.com/shicai/MobileNet-Caffe)的.在如何将这个模型转换成mlmodel文件时让我花了一些功夫。一旦将这个模型转换后，集成到应用中确是非常容易的。（这个[转换脚本](https://github.com/hollance/MobileNet-CoreML/blob/master/Convert/coreml.py)我已经在GitHub开源）



这个应用并没有什么令人惊奇的地方----对于一张静态图片5个输出结果。但是却展示了Core ML是如此简单，仅仅需要几行代码。


> 这个demo 应用在模拟器中是完全OK的但是再真机上却崩溃。请继续往下阅读，找出为什么会发生这种情况。



当然，我想知道这一切的背后发生了什么。很容易可以知道mlmodel 文件被编译进了应用bundle中的mlmodelc 目录中。这个目录包含了一些不同文件，如一些二进制文件，json文件。所以有了一点眉目，在实际部署到您的应用程序之前，您可以看到Core ML如何转换mlmodel的。



例如，MobileNet Caffe 模型使用了所谓的批量标准化层（Batch Normalization layers ）并且我也证实了这些都会展现在转化后的mlmodel 文件中，然而，在编译后的mlmodelc 中显示这些标准化的layers 已经被移除了。这是一个好消息：Core ML 优化了模型。



尽管如此，它似乎可以更好地优化模型的结构，因为mlmodelc仍然包含不严格必要的缩放图层。



当然，我们现在仅仅只是iOS 11 beta 1，Core ML 还有待提升。这也就是说，在给你使用之前还有优化模型可能----例如，通过[“folding” the Batch Normalization layers](http://machinethink.net/blog/object-detection-with-yolo/#converting-to-metal)----你不得不测试和比较你的模型。




还有一件事需要注意：尽管你的模型同样是泡在CPU和GPU上。我注意到Core ML 会选择是运行在CPU（使用Accelerate 框架）还是GPU（使用Metal）。这有两种实现，两种工作方式----所以你都需要测试。



例如，MobileNet 使用所谓的深度卷积层。原始的模型是使用Caffeine训练的，该模型是支持深度卷积的，将一个输出通道数赋值给一个常规卷积的groups属性。其结果MobileNet.mlmodel 文件也是相同的。在iOS模拟器中工作的很好但是在真机上却崩溃。




是什么导致在模拟器中使用了Accelerate 框架而在设备中却使用Metal Performance Shaders。由于Metal序列化数据的方式，MPSCNNConvolution 内核有一个限制：你不能将输出通道数 属性赋值给groups属性。😔




我已经把这个bug 提交给苹果了，但是我的观点是：仅仅因为在模拟器中运行正常并不意味着在真机设备中也能正常运行。要经过测试！！




<h4>到底有多快</h4>
我无法测试Core ML 的速度，因为我的10.5寸的iPad Pro 要到下周才到（呵呵）。



我非常感兴趣使用我的[Forge library](https://github.com/hollance/Forge) 和使用CoreML （考虑到我们仍然在使用的早期的beta版）运行MobileNets 在速度上的差异。





敬请关注！ 当我有数据要分享时，我会更新此部分



<h4>Vision</h4>

下面我们要讨论的是Vision 框架。



正如其名，Vision 让你执行计算机视觉任务。在过去你可能使用[OpenCV](http://opencv.org),但是现在苹果有自己的API。


<div align="center">
<img src="/images/blog/Vision@2x.png" align="middle"/>
</div>




Vision 可以做很多工作：



* 在一张图片中识别脸部。以上给出每一张脸的轮廓。



* 识别脸部特征，例如眼睛和嘴巴的位置，头部的形状等等。



* 在一张图片查找有规则形状的东西，像路牌。



* 在视频中跟踪运动的物体。



* 确定地平线的角度。



* 转换两个图片使其内容对齐，这对于照片合成是非常有用的。



* 检测包含文本的图像中的区域。



* 检测和识别条形码。



以上所有的工作我们现在可能使用Core Image 和AVFoundation 但是现在所有的这些我们只需要集成一个框架。



如果你的应用需要做一个计算机视觉任务，你再也不需要自己实现或者使用别人的library----只需要使用Vision框架即可。您还可以将其与Core Image框架相结合，以获得更多的图像处理能力。



甚至您还可以使用Vision来驱动Core ML，这允许您使用计算机视觉技术来作为神经网络的预处理步骤。例如，您可以使用Vision 探测脸部位置和大小，视频剪辑，而这只是神经网络的一部分。



事实上，很多时候您将Core ML用于图像或者视频仅仅只是通过Vision，对于“raw” Core ML，你需要确定你的输入图像是可预测的，但是对于Vision框架更关注于调整图像大小等等，这节省了一些额外的努力。




以下是Vision 驱动CoreML的代码：



```
// the Core ML machine learning model
let modelCoreML = ResNet50()

// link the Core ML model to Vision
let visionModel = try? VNCoreMLModel(for: modelCoreML.model)

let classificationRequest = VNCoreMLRequest(model: visionModel) { 
  request, error in
  if let observations = request.results as? [VNClassificationObservation] {
    /* do something with the prediction */
  }
}

let handler = VNImageRequestHandler(cgImage: yourImage)
try? handler.perform([classificationRequest])

```


请注意，`VNImageRequestHandler` 需要请求一个对象数组，您可以这样讲多个计算机视觉任务连接到一起，像这样：



```
try? handler.perform([faceDetectionRequest, classificationRequest])
```











