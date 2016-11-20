---
layout: post
title:  Samurai-Native 渲染及布局原理
date:   2016-11-28 17:46:34
categories: WebNative
comments: true
---

接着上一篇博客 [Samurai-Native 模板及样式解析原理](http://code.liqingyao.com/samurai-native-parse-workflow/)

## Render Workflow

### 分配 StyleSheet

这里的分配 styleSheet 是为 domTree 上的每个节点选择应用于自身的样式。在开始渲染 workflow 之前，先来回顾一下剥离了 Samurai CSS 解析内部继承关系之后的关键类图，同时弄明白 CSS 解析部分关键类之间的相互协作关系，也就明白了为什么需要分配 styleSheet 这一个步骤。

<img src="{{ site.url }}/images/samurai-css-class-structure.png"/>

类图中间的 `SamuraiCSSStyleSheet` 是与 domTree 解析、renderTree 渲染等模块相连接的类。在解析样式时，他会调用 `SamuraiCSSRuleParser` 单例的解析方法，把解析得到的 `KatanaOutput` 数据结构中的规则表添加到 `SamuraiCSSRuleSet` 对象中

{% highlight ruby %}
- (BOOL)parse
{
    _output = [[SamuraiCSSParser sharedInstance] parseStylesheet:self.resContent];
    [self.ruleSet addStyleRules:&_output->stylesheet->rules];
}
{% endhighlight %}

添加方法中，先创建一个 `SamuraiCSSRule` 规则对象，然后按照规则的匹配属性选择是 idRules、tagRules、classRules，还是其他类型的规则，放到相应的集合中。

{% highlight ruby %}
- (void)addStyleRule:(KatanaStyleRule *)rule
{
    for ( unsigned int i = 0; i < rule->selectors->length; i++ )
    {
        KatanaSelector * selector = rule->selectors->data[i];
        SamuraiCSSRule * data = [[SamuraiCSSRule alloc] initWithRule:rule selector:selector position:_ruleSeed++];
        BOOL found = [self findBestRuleSetAndAddWithSelector:selector ruleData:data];
        if ( NO == found )
        {
            [self.privateUniversalRules addObject:data];
        }
    }
}
{% endhighlight %}

domTree 在为每个节点选择样式时，调用 `SamuraiCSSRuleCollector` 的方法在集合中为指定节点选择规则

{% highlight ruby %}
- (NSDictionary *)collectFromRuleSet:(SamuraiCSSRuleSet *)ruleSet forElement:(id<SamuraiCSSProtocol>)element;
{% endhighlight %}

在过程中可以发现，Katana Parser 只是扮演解析的工作，即把样式表转换为键值对词典，保存的内容是字符串形式的样式，那么接下来需要做的两件事就是样式选择器和转化器。转化器的实现在建立 renderTree 的过程中完成。

接下来看一下样式选择器的工作流程

<img src="{{ site.url }}/images/samurai-apply-style-timeline.png"/>

分配 styleSheet 同样是从 worklet 的工厂方法发起，实际调用的 `SamuraiHtmlDocumentWorklet_60ApplyStyleTree` 方法

{% highlight ruby %}
- (BOOL)processWithContext:(SamuraiHtmlDocument *)document
{% endhighlight %}

内部对 `document.domTree` 递归，为每个 `domNode` 和 `domNode.shadowRoot` 调用

{% highlight ruby %}
- (NSMutableDictionary *)computeStyleForDomNode:(SamuraiHtmlDomNode *)domNode
{% endhighlight %}

选择样式集合，存储到 `domNode.computedStyle` 里，为下一步渲染时的样式转化器使用。

选择器主要的工作是处理继承关系和多种属性的集成，在上面方法的内部，实际上依次处理了以下样式：

- 默认样式值 `[SamuraiHtmlUserAgent sharedInstance].defaultCSSValue`
- 默认继承的样式：在 `[SamuraiHtmlUserAgent sharedInstance].defaultCSSInherition` 中选择父类有的样式
- 默认属性样式：在 `[SamuraiHtmlUserAgent sharedInstance].defaultDOMAttributedStyle` 中选择节点属性中存在的样式
- 内联样式：直接调用 CSS 解析器解析样式属性的内容

{% highlight ruby %}
NSDictionary * attrStyle = [[SamuraiCSSParser sharedInstance] parseDictionary:domNode.attrStyle];
{% endhighlight %}

- 匹配样式：之前 styleTree 解析出的匹配当前节点的样式

{% highlight ruby %}
NSDictionary * matchedStyle = [domNode.document.styleTree queryForObject:domNode];
{% endhighlight %}

- 继承样式：继承的样式享有最高的优先级，在父节点的 `computedStyle` 中选出当前节点继承的样式

{% highlight ruby %}
NSObject * inheritedValue = [domNode.parent.computedStyle objectForKey:key];
{% endhighlight %}

另一款 NetSurf 浏览器 的 LibCSS 解析器具备解析和选择的能力，并且内部采用的数据结构占内存更小、选择器速度更快，但缺点是把上层对象做的事放到底层做，而且依赖外部库，代码的扩展性稍弱。两个解析器各有利弊，看如何取舍。

### 生成 RenderTree

renderTree 是渲染过程中的核心，在这一步中会逐步构建出一棵完整的 renderTree。首先看一下相关类的继承和调用关系，把握一个宏观方向。
 
<img src="{{ site.url }}/images/samurai-render-class-structure.png"/>

类图中祖先类 `SamuraiTreeNode` 的两路分支，一路 `SamuraiDomNode` 掌管着 domTree，可以看到类属性 document 持有解析的源头文档对象，通过 `- (void)attach:(SamuraiDocument *)document` 和 `- (void)detach()` 绑定和解绑文档，另一个 `computesStyle` 属性保存节点的样式词典，同时一个 domNode 还关联着 `shadowRoot`，如果自己是 shadow 节点那么还有一个 `shadowHost` 属性。

`SamuraiTreeNode` 的另一路分支 `SamuraiRenderObject` 掌管着 renderTree，与 domTree 不同的是 renderTree 作用与渲染、样式计算、视图创建和布局计算，这点从数据结构中就能体现，`SamuraiRenderObejct` 弱引用生成自己的 domNode，持有一个用于计算渲染样式的 `SamuraiRenderStyle` 对象、一个实际的原生控件视图、和一个用于计算视图布局的 `SamuraiLayoutObject` 对象，相应的有绑定 domNode、style 和计算样式、视图尺寸的一些方法。

了解了 `SamuraiRenderObject` 的横向关系网之后，来看一下纵向继承关系。`SamuraiRenderObject` 的子类 `SamuraiHtmlRenderObject` 在内部重写了父类的大多数方法，他的四个子类是 renderTree 构建时的实体类，根据 domNode 的类型和在 domTree 中的层级进行路由选择，

- `SamuraiHtmlRenderViewPort`：document 类型的节点，一般是唯一的，指代浏览器或屏幕对象
- `SamuraiHtmlRenderContainer`：domTree 中的非叶子节点和 shadowTree 的根节点
- `SamuraiHtmlRenderElement`：domTree 中所有视图层级非隐藏的叶子节点
- `SamuraiHtmlRenderText`：domTree 中所有 text 类型的节点

之所以需要上面的路由规则来对每个节点分类，一是在构建 renderTree 时可以做一些视图层级上的优化，二是便于接下来计算布局时的分类讨论，有点类似于 stackView 的概念，这一点放在后面分析。

宏观把握之后，再来看关键路径的方法调用就容易理解了

<img src="{{ site.url }}/images/samurai-build-render-timeline.png"/>

作为渲染流程中的最后一步，同样是通过触发 `SamuraiHtmlDocumentWorklet_70BuildRenderTree` 的入口方法

{% highlight ruby %}
- (BOOL)processWithContext:(SamuraiHtmlDocument *)document
{% endhighlight %}

先根据 domNode 的类型创建对应的 renderObject，对 document 类型节点创建 `SamuraiHtmlRenderViewPort` 对象，对 text 类型节点创建 `SamuraiHtmlRenderText` 对象。容器和元素对象的创建条件稍微复杂一些，

{% highlight ruby %}
- (SamuraiHtmlRenderObject *)renderDomNodeElement:(SamuraiHtmlDomNode *)domNode forContainer:(SamuraiHtmlRenderObject *)container inDocument:(SamuraiHtmlDocument *)document
{
    SamuraiHtmlRenderStyle * thisStyle = [SamuraiHtmlRenderStyle renderStyle:domNode.computedStyle];
    SamuraiHtmlRenderObject * thisObject = nil;

    CSSViewHierarchy viewHierarchy = [thisStyle computeViewHierarchy:CSSViewHierarchy_Inherit];
    switch ( viewHierarchy )
    {
        case CSSViewHierarchy_Hidden:
        break;
        case CSSViewHierarchy_Branch:
        {
            for ( SamuraiHtmlDomNode * childDom in domNode.childs )
            {
                [self renderDomNode:childDom forContainer:container inDocument:document];
            }
        }
        break;
        case CSSViewHierarchy_Leaf:
        {
            thisObject = [SamuraiHtmlRenderElement renderObjectWithDom:domNode andStyle:thisStyle];
        }
        break;
        case CSSViewHierarchy_Tree:
        {
            thisObject = [SamuraiHtmlRenderContainer renderObjectWithDom:domNode andStyle:thisStyle];
            if ( thisObject )
            {
                for ( SamuraiHtmlDomNode * childDom in domNode.childs )
                {
                    [self renderDomNode:childDom forContainer:thisObject inDocument:document];
                }
            }
        }
        break;
        default:
        break;
    }
    return thisObject;
}
{% endhighlight %}

通过样式对象计算自己的视图层级，处理继承关系，之后根据视图层级选择创建 `SamuraiHtmlRenderContainer` 对象还是 `SamuraiHtmlRenderElement` 对象。

接下调用 `SamuraiHtmlRenderStyle` 的接口方法计算 renderNode 的每个样式属性，完成后根据 `SamuraiHtmlRenderObejct` 的具体类型创建相应的 `SamuraiHtmlLayoutObject`，并绑定到 `renderNode.layout` 属性上，为下面的布局计算做准备。
