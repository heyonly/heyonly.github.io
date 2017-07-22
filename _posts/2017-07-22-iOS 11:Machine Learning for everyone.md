---
layout: post
title: iOS 11:Machine Learning for everyone 
categories: translation
description: Machine Learning
keywords: Machine Learning,iOS Machine Learning
---

在刚刚过去的WWDC 2017上，一件很引人注目的事情：苹果全面进军机器学习。




并且苹果想尽办法尽可能的让更多的开发者加入。



去年苹果发布了\:[Metal CNN and BNNS frameworks](http://machinethink.net/blog/apple-deep-learning-bnns-versus-metal-cnn/)来创建基本卷积网络。今年，我们了解更多关于Metal，一个新的计算机视觉框架，另外还有Core ML:一个很容易将ML模型整合进你的应用中的工具包。


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



这个模型包含在一个.mlmodel 文件中。这是一个在你的模型中描述层级的!:[可打开的文件格式](https://pypi.python.org/pypi/coremltools),输入和输出，类标签，和一些需要预处理的数据。这也包含所有的学习参数（权重和偏差）。




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








