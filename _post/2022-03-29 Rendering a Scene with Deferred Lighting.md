# 2022-03-29 Rendering a Scene with Deferre

Tile-based模式的GPU光照的计算是非常消耗计算资源的，为了减少光照的重复计算，通过实施延时光照渲染来进行优化。

# 概述

通过Apple例子来看一下延时光照渲染实施方法，这个例子应用shadow map（阴影贴图）实现阴影，并使用模版缓冲区剔除光量。

![](image/image.png "")

与 Forward lighting相比，Deferred lighting可以渲染大量的灯光数量。比如，在Forward lighting模式下，如果场景中有很多光源，无法对每个光源在每个片元上作用的计算量进行全量计算。需要应用到复杂的排序和像素合并算法来筛除对每个片元能产生作用的光源来限制计算量。使用Deferred lighting，可以容易地将多个光源应用到场景中。

# 重要概念

在开始解读示例代码的之前，回顾一下以下概念可以更好的理解关键细节。

## 传统延时光照渲染

传统延时光照渲染通常分为两个渲染通道：

- 第一个渲染通道：G-buffer 渲染。渲染器绘制和变换场景里的模型，片元函数将渲染结果输出到 "几何缓冲区" 和 "G-Buffer"中。G-Buffer包含了模型的材料颜色信息，以及每个片元的法线、阴影和深度信息。
- 第二个渲染通道：延时光照和构图。渲染器绘制每个光柱，使用G-Buffer中的数据重建每个片元的位置信息并应用光照计算。绘制光源时，该光源输出为混合前一个光源的输出后的结果。最后，渲染器将其他数据（如阴影和定向光照）合成到场景中。

![](image/image_1.png "")

一些macOS的GPU是"即时渲染"（IMR）架构。IMR GPU延时光照只能通过两个渲染通道实现。所以这个例子的macOS版本实现了两个通道的延时光照算法。

## Apple芯片GPU上的单通道延时光照

Apple芯片GPU使用了基于磁贴分片的延迟渲染（TBDR）架构，TBDR架构在GPU内部专门为第一块磁贴设计了内存来存储渲染数据。渲染结果存储到磁贴内存中，可以避免数据在GPU和系统内存中所花费的大量资源消耗，GPU和系统内存DDR是通过带宽受限的内存总线交换数据。GPU什么时候需要将磁贴内存写入到系统内存取决于以下配置情况：

- 应用程序主动执行了存储命令
- 应用程序纹理的存储模式
将 "MTLStoreAction.store"设置为存储操作时，渲染通道中的渲染目标的输出数据将从磁贴内存中写入系统内存。如果这些数据将用于后续的渲染通道，这些纹理将做为输入数据从系统内存中读入到GPU的纹理缓存中。因此，传统的延时光照渲染器要求第一次和第二次渲染通道之间将G-Buffer数据存储在系统内存中。
![](image/image_2.png "")
由于TBDR架构允许在任何时间从磁贴内存中读取数据。这允许片元着色器从磁贴内存中读取数据并对渲染目标进行计算，然后将数据再次写入磁贴内存。这个特性避免了在第一次和第二次渲染通道之间将G-Buffer数据存储到系统内存中。所以，TBDR架构下延迟光照渲染器可以通过单个渲染通道实现。
G- Buffer在单个渲染通道由GPU（而不是CPU）生成和使用。因此，在渲染通道开始之前，不会从系统内存中加载数据，也不会在渲染通道完成后将结果存储在系统内存中。光照片元函数不是从系统内存中的纹理读取G-Buffer数据，而是从G-Buffer读取数据，同时它仍然作为渲染目标附加到渲染通道。因此，不需要为G-Buffer纹理分配系统内存，可以使用MTLStorageMode.memoryless存储模式声明这些纹理。
![](image/image_3.png "")

允许TBDR GPU从片元函数中附加的渲染目标读取数据的特性称为"可编程混合"
## 具有栅格顺序组的延时光照
默认情况下，当片元着色器将数据写入像素时，GPU会等到着色程序完全完成对该像素的写入后，再开始为同一个像素执行另一个片元着色程序。
![](image/image_4.png "")

栅格顺序组允许应用程序增加GPU片元着色器的并行化。使用栅格顺序组，片元函数可以将渲染目标分隔到不同的执行组中。这种分离执行允许GPU在片元着色器的上一个实例完成时将数据写入另一个组的像素之前，读取一个组中的渲染目标并对其执行计算。

![](image/image_5.png "")
在示例程序中，一些光照片元函数使用了栅格顺序组：
- Raster order group 0. "AAPLLightingROG" 用于包含光照计算结果的渲染目标
- Raster order group 1. "AAPLGBufferROG" 用于光照函数中G-Buffer数据。

这些栅格顺序组允许GPU在上一个片元着色器完成光照计算并写入输出数据之前读取片元着色器中的G-Buffer并执行光照计算。

## 渲染延时光照帧

这个示例通过以下阶段来显示完整帧：

1. 阴影贴图
2. G-Buffer
3. 定向光
4. 光蒙板
5. 点光源
6. 天空盒
7. 彩色小灯

示例的单通道延时渲染器生成G-Buffer并在单个渲染通道中执行所有后续阶段。由于IOS和tvOS的GPU是TBDR架构，单通道实现允许设备从磁贴内存中的渲染目标读取G-Buffer数据。

```Swift
encodePass(into: commandBuffer, 
           using: gBufferAndLightingPassDescriptor, 
           label: "GBuffer & Lighting Pass") { renderEncoder in
    
    encodeGBufferStage(using: renderEncoder)
    encodeDirectionalLightingStage(using: renderEncoder)
    encodeLightMaskStage(using: renderEncoder)
    encodePointLightStage(using: renderEncoder)
    encodeSkyboxStage(using: renderEncoder)
    encodeFairyBillboardStage(using: renderEncoder)
}
```


示例程序的传统延时渲染在一个渲染通道中生成G-Buffer，然后在另一个渲染通道中执行后续

```Swift
encodePass(into: commandBuffer,
           using: gBufferPassDescriptor,
           label: "GBuffer Generation Pass") { renderEncoder in

    encodeGBufferStage(using: renderEncoder)
}
```


```Swift
encodePass(into: commandBuffer,
           using: lightingPassDescriptor,
           label: "Lighting Pass") { (renderEncoder) in
            
    encodeDirectionalLightingStage(using: renderEncoder)
    encodeLightMaskStage(using: renderEncoder)
    encodePointLightStage(using: renderEncoder)
    encodeSkyboxStage(using: renderEncoder)
    encodeFairyBillboardStage(using: renderEncoder)
}
```


## 渲染阴影贴图

示例程序通过从光源的透视渲染模型，为场景中的单个定向光源（太阳）渲染阴影贴图。











