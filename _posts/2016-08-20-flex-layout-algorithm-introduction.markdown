---
layout: post
title:  "CSS Flex Layout 算法解析"
date:   2016-08-20 16:32:32
categories: WebNative
comments: true
---
实现 Web-Native 混合开发已有很多知名开源框架，包括 Facebook 的 [React-Native](http://reactnative.cn/)， 阿里的 [Weex](http://alibaba.github.io/weex/)， GeekZoo 的 [Samurai-Native](https://github.com/hackers-painters/samurai-native) 和 [Bee-Framework](https://github.com/gavinkwoe/BeeFramework)。其实现各有差异和特色，React-Native 采用 JSX + JS + CSS 技能栈，为前端开发者向移动端迁移这一路径提供了解决方案。与 React-Native 侧重 JS 端不同的是，Samurai-Native 的原生端更重，采用 W3C 标准 HTML + CSS + JS，更倾向于为移动端开发者向前端开发迁移这一路径提供解决方案。无论何种框架，要实现通过用网页的规范或自定义的模板规范来达到动态控制原生 UI，都会包含以下过程：

  1. HTML 或模板解析，构建 DOM 树
  2. CSS 样式解析，并转换为原生系统属性
  3. 动态数据解析及注入
  4. 从 DOM 树构建渲染树
  5. 对渲染树各节点应用样式，并计算布局
  6. 对渲染树各节点绑定事件，实现 JS 和原生方法之间的互相调用
  7. 从渲染树生成视图，最终显示

布局是承上启下的中间环节，渲染树是从 DOM 树映射而来的可布局的层级关系，通过应用布局属性确定视图排版。React-Native 和 Weex 的核心布局算法都采用 Facebook 开源的 [CSSLayout](https://github.com/facebook/css-layout) 算法，CSSLayout 基于 [W3C 标准的 Flexbox 模型](https://www.w3.org/TR/css-flexbox-1/)对页面元素排版，同时也支持相对布局和绝对布局，iOS 和 Andriod 平台都适用。把布局和视图生成两部分从整个架构中抽离出来，也可成为客户端 UI 框架，比如 Facebook 的 [ComponentKit](http://componentkit.org/) 和 [AsyncDisplayKit](http://asyncdisplaykit.org/)，前者用 React 的思路通过描述性、可组合的组件实现视图层，后者极大程度的优化了布局、渲染等操作，大大提高了帧率。因此布局不仅是起承转合的环节，更是性能的瓶颈所在，需要非常扎实的功底，灵活运用缓存、线程切换等手段来优化性能。水很深，慢慢学习，先从布局算法开始。

CSSLayout 基于 Flexbox 模型，对容器可应用以下属性：

  - FlexDirection
  - FlexWrap
  - JustifiyContent
  - AlignItems
  - AlignContent

对元素可应用以下属性：

  - Flex
  - AlignSelf

除了 Flex 属性，还支持普通的 Position 和 Overflow 属性。

CSSLayout 按照 [CSS Flexbox 标准]((https://www.w3.org/TR/css-flexbox-1/))建议的流程计算布局，主要步骤：

  1. 对特殊节点和情况进行预处理：

  1.1 文本节点：采用 measure 方法，通过回调视图对文本实际计算得到的尺寸确定宽高

  1.2 叶子节点：直接求解，不进行递归计算

  1.3 不执行布局计算的节点：直接求解

  2. 确定节点内每个子节点的 FlexBasis

  3. 对节点内所有子节点遍历，对元素分行并计算主轴和交叉轴对齐

  3.1 将子节点分行

  3.2 计算当前行内元素在主轴上的尺寸，计算当前行剩余可分配空间

  3.3 计算当前行内元素在主轴上的位置，计算当前行内元素在交叉轴上的尺寸

  3.4 计算当前行内元素在交叉轴上的位置

  4. 计算节点的多行对齐，更新元素在交叉轴上的位置

  5. 计算节点的最终尺寸和位置

  6. 计算绝对定位子节点的尺寸和位置

  7. 设置子节点的 trailing 位置

### 准备

#### 样式和布局属性

一个对象的布局由位置和尺寸这两个要素唯一确定，但实际使用中我们很少用这种赋绝对值的思维来指定排版，而是通过指定相对位置、相对宽高、相互关系来间接实现，所以布局要做的就是从这些相对信息中推算出每个对象的绝对信息，通过多次从根节点开始向下遍历，以及从子节点向上回溯，不断估计、修正，计算出树上每个节点的唯一布局。

计算过程中用到的样式属性包括：

![style-properties]({{ site.url }}/images/style-properties.png)

  - 主轴 `mainAxis` 以及垂直于主轴方向的交叉轴 `crossAxis`
  - 外边距 `margin`、边框 `border`、内边距 `padding`，这三个边距都分别包括 `left`、`top`、`right`、`bottom` 四个方向的值，可以分别指定。需要注意的是，对象的实际尺寸 `width` 和 `height` 是除去 `margin` 后的部分，而在计算过程中，一个对象内部的计算尺寸是除去 `border` 和 `padding` 后的部分。
  - `leading` 和 `trailing` 是另一种访问上面三个边距的方式，根据 `FlexDirection` 属性分别对应不同边缘的边距值，`leading-left` 在行排列 `Row` 时对应 `left` 边缘，在列排列 `Column` 时对应 `top` 边缘，在逆向行排列 `RowReverse` 时对应 `right` 边缘，在逆向列排列 `Column-Reverse` 时对应 `bottom` 边缘。
  - 对象的边界可以通过 `minWidth`、`minHeight`、`maxWidth`、`maxHeight` 来指定，当基于 `FlexGrow` 扩展和 `FlexShrink` 压缩时作为边界的约束条件。

布局算法把外部传入的计算属性先转化为对应的数组，通过下标访问具体值，而下标又是通过主轴、交叉轴构造的映射关系表来获取。比如在四种 `FlexDirection` 模式下，`margin` 的 `left` 值可以统一用 `margin[leading[mainAxis]]` 来表示。

计算过程中用到的布局属性包括：

  - 位置 `position`，包括 `left`、`top`、`right`、`bottom` 四个定位值
  - 尺寸 `dimension`，包括 `width` 和 `height`
  - 估计尺寸 `measuredDimension`，包括 `width` 和 `height`，`measuredDimension` 是计算过程中的中间变量，几次迭代后得到最终的 `dimension`

布局模式：

  - 未定义 `MeasureModeUndified`
  - 精确 `MeasureModeExactly`
  - 至多 `MeasureModeAtMost`

#### 布局模板

文章中用下面的排版来解析布局算法：

![layout-result]({{ site.url }}/images/layout-result.png)

  1. 最外层是一个 Flex 容器对象，内部包含四个 Flex 元素对象和一个绝对布局对象，容器对象定宽 `600px`，高度由内部对象决定，同时规定了主轴、`margin` 和 `padding`，以及一些对齐方式：行排列、元素可换行、主轴起点对齐、交叉轴起点对齐、多行时沿交叉轴两端对齐
  2. 内部第一个元素定宽 `300px`，高度由内容决定，实际可能是一个 Label、TextField 或者 ImageView；第二个元素定高 `100px`，上下外边距各 `40px`，扩展比例系数为 `2`，压缩比例系数为 `1`；第三个元素定义最小宽度 `100px`，扩展和压缩比例系数都为 `1`，并规定自己沿交叉轴拉伸对齐；第四个元素定宽 `200px`，高度由内部子元素决定
  3. 最后一个绝对布局对象定义其距离父对象的右边距和下边距各 `10px`

### 预处理

算法首先对内容节点、叶子节点和非布局节点这三种情况进行预处理，提前返回，减少走完整个流程的次数，尽可能的减少计算量。

![xmind-precalculating]({{ site.url }}/images/xmind-precalculating.png)

#### 内容节点

对 `Label`、`TextField` 等文本节点和 `ImageView` 等由内容决定的节点直接通过外部传入的 measure 回调拿到尺寸。

`layoutNode` 的 `measure` 方法可以通过协议让具体的视图来实现：

```ruby
    - (CGSize (^)(CGFloat))measure
    {
      CGSize (^measure)(CGFloat w) = objc_getAssociatedObject(self, _cmd);
      if (measure == nil)
      {
        __weak typeof(self) wself = self;
        measure = ^CGSize(CGFloat width) {
            __strong typeof(wself) sself = wself;
            CGFloat w = isnan(width) ? 0 : width;
            return [sself sizeThatFits:CGSizeMake(w, MAXFLOAT)];
        };
        [self setMeasure:measure];
      }
      return measure;
    }
```


引用 `layoutNode` 的上下文可以拿到其绑定视图的 `measure` 方法：

内容节点宽高的取值由外部传入的布局模式决定，精确模式下内容节点的尺寸就是外部传入的宽高，未定义和至多模式下尺寸由 `measure` 回调的宽高确定，同时要保证内部尺寸非负。

为了比较清晰的阐明思路，只列出了宽度的计算表达式。

#### 叶子节点

对没有孩子的叶子节点，由于不需要递归计算内部子节点的布局，因此可以直接通过指定的模式算出估计尺寸，跳过接下来的流程。

#### 非布局节点

同样，在对布局树的多次递归过程中，对于只需知道子节点尺寸而不需要知道位置的情况，会把 `performLayout` 标志位置为 `NO` 来跳过计算量消耗较大的计算位置的流程。

可以看到，容器中第一个元素的宽高已经确定。

### 确定 flexBasis

这一步确定容器中每个子元素的在主轴上的 `flexBasis` 值。`flexBasis` 是每个元素的在未扩展和压缩前的基准尺寸，父容器用来计算主轴的剩余空间，然后根据扩展和压缩比例系数为每个 Flex 子元素调整尺寸。由于绝对定位子节点不参与 Flex 布局，因此不需要计算 `flexBasis` 值，在这一步中，会先把绝对定位子节点存储在链表中，在 Flex 布局完成后再单独计算所有绝对定位节点的布局。

![xmind-flexbasis]({{ site.url }}/images/xmind-flexbasis.png)

对于相对定位子节点：

  1. 如果样式中直接规定了主轴尺寸，则 `flexBasis` 直接被指定，这里还会将尺寸和 `padding + border` 的尺寸比较，限定 FlexBasis 值不能小于内边距和边框长度
  2. 如果 `flexBasis <= 0` 并且内部主轴尺寸未定义，则 `flexBasis = 0`
  3. 其他情况则通过估计子节点内部元素的尺寸来确定，这里首先会估算子节点的宽高并确定对应的估计模式：

    3.1 样式中定义了宽高：直接使用定义的值，且指定模式为 `MeasureModeExactly`

    3.2 交叉轴尺寸未定义：根据标准，在未定义交叉轴尺寸的情况下，默认等于由外部指定的最大可用尺寸，并且指定模式是 `MeasureModeAtMost`

    3.3 交叉轴拉伸对齐：交叉轴尺寸为最大可用尺寸，且指定模式为 `MeasureModeExactly`

   然后调用 layout 函数递归估算内部节点在主轴上所占用的尺寸，并赋值给 `flexBasis`
   layout 函数

这步中可以得到容器中第一个元素 `flexBasis = 300px`，第二个元素 `flexBasis = padding + border`，第三个元素 `flexBasis = 100px`，第四个元素 `flexBasis = 200px`。

### 单行计算

这一步骤是一个大循环体，首先遍历容器中所有子节点，按照上一步中计算出的 `flexBasis` 对超过容器可用主轴尺寸的节点分行，接着对每行分别计算各节点经过 flexible 后的尺寸并更新剩余可分配空间。之后根据主轴对齐方式调整各元素在主轴上的位置并计算交叉轴尺寸，最后根据交叉轴对齐方式计算元素的交叉轴位置。

#### 元素分行

这一步是一个内部循环体，对相对布局元素累加当前消耗的主轴尺寸和该元素的 `flexBasis`，如果超过可用主轴尺寸且容器支持换行，则表示占满一行并跳出内循环。循环体中：

  1. 累加当前行所消耗的主轴尺寸 `sizeConsumedOnCurrentLine += flexBasis + margin`
  2. 累加扩展系数 `flexGrowFactors += flexGrow`
  3. 累加压缩比例系数 `flexShrinkScaledFactors += flexShrink * flexBasis`，注意 Flex 中定义压缩比例是相对于子节点的主轴尺寸而言，因此压缩系数需要乘 `flexBasis` 后再累加
  4. 把节点保存到相对布局节点链表中，之后进行布局

到这里可以确定容器分为两行，第一行 `sizeConsumedOnCurrentLine = ` `flexGrowFactors = 3`，`flexGrowFactors = 3`。

#### 主轴尺寸计算

  1. 计算初始剩余可分配空间
  2. 初步计算子节点的主轴 flex 尺寸，主要处理触发边界限制的元素

  	2.1 经 flex 扩展和压缩后，对触发边界限制的元素使用边界条件，比如元素扩展后的尺寸大于 `maxWidth` 限制，则元素的主轴尺寸为 `maxWidth`

  	2.2 累加触发边界限制元素的修正空间 `deltaFreeSpace`、修正扩展系数 `deltaFlexGrowFactors` 和修正压缩比例系数 `deltaFlexShrinkScaledFactors`

  	2.3 剔除触发边界限制的元素，并更新剩余可分配空间和扩展压缩系数

  3. 重新计算子节点的主轴 flex 尺寸
  4. 确定子节点主轴尺寸及估计模式：主轴尺寸为子节点主轴 flex 尺寸加上外边距尺寸，并规定模式为 `MeasureModeExactly`
  5. 确定子节点交叉轴尺寸及估计模式：
  6. 第二次调用 layout 	方法，递归计算子节点内部节点的主轴 flex 尺寸
  7. 最后更新容器内的剩余空间

到这里四个元素的主轴尺寸已经计算完毕。

#### 主轴位置计算及交叉轴尺寸计算

  1. 对主轴估计模式为 `MeasureModeAtMost` 的情况，清空剩余空间
  2. 根据容器的 `JustifyContent` 属性计算 `leadingMainDim` 和 `betweenMainDim`
  3. 对本行内元素依次计算主轴位置 `posotion[mainAxis] = mainDim`，并更新 `mainDim` 和初步计算 `crossDim`：
  3.1 对非 layout 节点：`mainDim += betweenMainDim + margin + flexBasis`，`crossDim` 为外部设定的可用尺寸
  3.2 对 layout 节点： `mainDim += betweenMainDim + margin + measureDim`，`crossDim` 为本行元素最大交叉轴尺寸

  4. 确定本行交叉轴尺寸

#### 交叉轴位置计算


### 多行计算

### 确定尺寸和位置

### 绝对定位

### Trailing 赋值

### 缓存机制
