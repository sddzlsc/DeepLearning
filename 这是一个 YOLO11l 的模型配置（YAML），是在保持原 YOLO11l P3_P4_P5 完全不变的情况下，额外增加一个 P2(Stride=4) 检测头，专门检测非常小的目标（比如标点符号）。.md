# 这是一个 YOLO11l 的模型配置（YAML），是在保持原 YOLO11l P3/P4/P5 完全不变的情况下，额外增加一个 P2\(Stride=4\) 检测头，专门检测非常小的目标（比如标点符号）。

下面按照每一行解释。

---

# 注释

```Plain Text
# Punctuation ablation B: preserve the original P3/P4/P5 path and add an
# auxiliary high-resolution P2/4 detection branch.
```

意思：

这是一个消融实验\(B\)。

设计目标：

> 保留 YOLO11 原来的 P3、P4、P5 检测结构，
>  再额外增加一个 P2 分支。
> 
> 

为什么？

因为：

假设图片大小 1280

一个 8 pixel 的标点

到了

P3 \(stride=8\)

就只有

```Plain Text
8 / 8 = 1 pixel
```

几乎没法检测。

而 P2

```Plain Text
8 /4 =2 pixel
```

还能保留一点信息。

所以增加 P2。

---

```Plain Text
Detect inputs deliberately
keep P3/P4/P5 first so their pretrained head widths remain compatible.
```

解释：

Detect输入顺序故意写成

```Plain Text
P3
P4
P5
P2
```

不要写

```Plain Text
P2
P3
P4
P5
```

为什么？

因为：

YOLO11官方Detect层

预训练权重

默认前三个Detect Head就是

```Plain Text
P3
P4
P5
```

如果顺序改了：

很多Detect Head权重不能直接加载。

所以：

新增P2放最后。

---

# 基本参数

```Plain Text
nc: 3
```

类别数

例如：

```Plain Text
0 汉字
1 标点
2 格线
```

---

```Plain Text
scale: l
```

使用Large模型。

---

```Plain Text
scales:
  l: [1.00, 1.00, 512]
```

意思：

```Plain Text
depth =1.0
width =1.0
max_channels=512
```

YOLO会自动根据scale缩放网络。

---

# Backbone

第一层

```Plain Text
- [-1, 1, Conv, [64, 3, 2]]
```

格式：

```Plain Text
[from,
 repeat,
 module,
 args]
```

这里

```Plain Text
from=-1
```

表示

输入来自上一层。

---

```Plain Text
1
```

表示重复一次。

---

```Plain Text
Conv
```

普通卷积。

---

```Plain Text
64
```

输出64通道。

---

```Plain Text
3
```

卷积核3×3。

---

```Plain Text
2
```

stride=2

图片尺寸：

```Plain Text
1280

↓

640
```

---

第二层

```Plain Text
- [-1, 1, Conv, [128, 3, 2]]
```

再次下采样

```Plain Text
640

↓

320
```

通道：

```Plain Text
64

↓

128
```

---

第三层

```Plain Text
- [-1, 2, C3k2, [256, False, 0.25]]
```

这一层非常重要。

表示：

重复2个

C3k2模块。

输出

256通道。

False表示：

不是深层版本。

0\.25表示：

内部BottleNeck比例。

注释：

```Plain Text
#2-P2/4
```

说明：

这一层就是

P2。

因为：

输入

1280

经过两次stride2：

```Plain Text
1280

↓

640

↓

320
```

stride

```Plain Text
4
```

所以叫

```Plain Text
P2
```

---

第四层

```Plain Text
- [-1,1,Conv,[256,3,2]]
```

继续下采样：

```Plain Text
320

↓

160
```

stride

```Plain Text
8
```

---

第五层

```Plain Text
- [-1,2,C3k2,[512,False,0.25]]
```

输出：

P3

```Plain Text
160×160
```

对应

stride8

---

第六层

```Plain Text
- [-1,1,Conv,[512,3,2]]
```

下采样

```Plain Text
160

↓

80
```

---

第七层

```Plain Text
- [-1,2,C3k2,[512,True]]
```

得到：

P4

stride16

True说明：

这是深层C3结构。

---

第八层

```Plain Text
- [-1,1,Conv,[1024,3,2]]
```

继续：

```Plain Text
80

↓

40
```

---

第九层

```Plain Text
- [-1,2,C3k2,[1024,True]]
```

深层特征。

---

第十层

```Plain Text
- [-1,1,SPPF,[1024,5]]
```

SPPF

Spatial Pyramid Pooling Fast

作用：

扩大感受野。

几乎不增加计算。

---

第十一层

```Plain Text
- [-1,2,C2PSA,[1024]]
```

YOLO11新增注意力。

PSA

Partial Self Attention

作用：

增强全局上下文。

最后得到

P5。

---

# Head

开始FPN\+PAN

---

```Plain Text
- [-1,1,Upsample]
```

把P5

```Plain Text
40

↓

80
```

---

```Plain Text
- [[-1,6],1,Concat,[1]]
```

把

```Plain Text
P5↑

+

Backbone P4
```

拼接。

Concat

表示：

通道拼接。

---

```Plain Text
- [-1,2,C3k2,[512,False]]
```

融合。

得到新的P4。

---

再次

```Plain Text
Upsample
```

```Plain Text
80

↓

160
```

---

```Plain Text
Concat

P3
```

融合。

---

```Plain Text
C3k2
```

得到

P3输出。

对应

layer16。

---

随后：

```Plain Text
Conv stride2
```

把P3

重新下采样

```Plain Text
160

↓

80
```

---

Concat

```Plain Text
13
```

就是：

和刚刚P4融合。

PAN路径。

得到

P4输出。

---

再次：

Conv

```Plain Text
80

↓

40
```

Concat

P5。

得到：

P5输出。

到这里：

**和官方YOLO11完全一样。**

所以：

官方权重都能加载。

---

# 新增P2

这一部分才是你的改动。

---

```Plain Text
- [16,1,Upsample]
```

注意：

不是22。

而是16。

为什么？

16就是：

P3输出。

把P3

```Plain Text
160

↓

320
```

恢复。

---

```Plain Text
- [[-1,2],1,Concat,[1]]
```

和Backbone里的P2

融合。

为什么不用最原始P2？

因为：

P3里面已经融合了高层语义。

直接Upsample后：

```Plain Text
语义

+

细节
```

一起进入P2。

这是FPN思想。

---

```Plain Text
- [-1,2,C3k2,[128,False]]
```

融合后

输出128通道。

得到

```Plain Text
P2输出
```

尺寸：

```Plain Text
320×320
```

stride

```Plain Text
4
```

---

# Detect

最后：

```Plain Text
- [[16,19,22,25],1,Detect,[nc]]
```

Detect输入：

```Plain Text
16

↓

P3

stride8
```

```Plain Text
19

↓

P4

stride16
```

```Plain Text
22

↓

P5

stride32
```

```Plain Text
25

↓

P2

stride4
```

Detect会自动计算对应的步长（stride）为：

需要注意的是，虽然 `Detect` 中的顺序写成 `[16, 19, 22, 25]`（P3、P4、P5、P2），YOLO 内部会根据各特征图的实际尺寸自动推导它们对应的 stride，因此第四个输入仍然会被识别为 stride=4 的检测层，而不是固定认为它是最大的 stride。这样做的主要目的就是保持前 3 个检测头与官方 YOLO11 的预训练权重布局兼容，同时新增一个独立的 P2 检测头用于极小目标。

