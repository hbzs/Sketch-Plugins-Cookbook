
Sketch Plugins Cookbook
=======================

![学渣乱翻，仅为自用🙃](https://github.com/hbzs/FGP/raw/master/resource/transinfo.png) ![译完打卡😎](https://github.com/hbzs/FGP/raw/master/resource/trans3.png)

Sketch 应用插件开发者的一系列技巧。

我将在 twitter 上发布日常更新，敬请关注 [@turbobabr](https://twitter.com/turbobabr) （作者）。

## CocoaScript: 不要使用 '===' 操作

首先概览 CocoaScript 像是带些语法糖的取了个花哨名字的 JavaScript，然而事实上许多方面是不一样的，要更好的了解它们。

![Don't use strict equal operator](./docs/cocoascript_do_not_use_scrict_equal_operator.png)

警告之一是 `===` 和 `!==` 操作符最为人所知的是 `严格相等` 和 `严格不相等` 。对于这个简单的建议是： 

- **在 SKETCH 插件中尽可能避免使用！**

要理解这个问题，尝试运行下面的脚本：

```JavaScript
var strA = "hello!";
var strB = @"hello!";

if(strA == strB) {
    print("They are EQUAL!");
} else {
    print("NOT EQUAL!")
}
// -> "They are EQUAL!"

if(strA === strB) {
    print("They are EQUAL!");
} else {
    print("NOT EQUAL!")
}
// -> "NOT EQUAL!"
```

这里 `==` 为 `true` ， `===` 为 `false` 。字符串的值是相等的，并且都被赋值 `"hello!"` 字符串，然而它们的类型是不同的。现在运行这个脚本检查它们的类型：

```JavaScript
function typeOf(obj) {
    print(toString.call(obj));
}

var strA = "hello!";
var strB = @"hello!";

typeOf(strA);
// -> [object String]

typeOf(strB);
// -> [object MOBoxedObject]
```

如你所见，变量 `strA` 和 `strB` 是不同类型。 `strA` 是 JavaScript 字符串类型，而 `strB` 是神秘的 `MOBoxedObject` 类型。问题在于 `strB` 的定义 - `@"hello!"`  相当于 `NSString.stringWithString("hello!")` ，它生成 NSString 类的装箱实例而不是 JS 字符串。

开发 Sketch 插件的时候，你经常处理在 `Sketch 运行时` 的数据。getter 属性和类方法的大部分返回的是装箱的 Objective-C 类，而不是原生 JS 对象。

要演示一个实际问题你能简单地操作：（1）创建一个矩形形状，（2）选中它，（3）运行下面的脚本：

```JavaScript
var layer=selection.firstObject();
if(layer) {
    print(layer.name());
    // -> Rectangle 1

    var isNameEqual = layer.name() === "Rectangle 1";
    print(isNameEqual);
    // -> false
}
```

> 注意： `===` 和 `!==` 的使用不是禁止的，你能在你想要的任何时候使用它们，但要注意你所比较的类型。 当你尝试移植一个已经存在的 JavaScript 库或框架到 CocoaScript 时这件事尤为重要。无论如何，我坚持忘记严格相等/不相等操作符，使用 '=='和 '!=' + 有必要的话手动类型检查。

## 播放声音

通常声音（sounds bound to commands）是令人厌烦且无用的，但有时小心使用的话还是非常有帮助的。

![Play Sound](./docs/play_sound.png)

因为 Sketch 插件可以使用所有 [AppKit Framework](https://developer.apple.com/library/mac/Documentation/Cocoa/Reference/ApplicationKit/ObjC_classic/index.html#//apple_ref/doc/uid/20001093) 的 API，我们能用插件做足够疯狂且酷的事……例如当插件使用 `-MSDocument.displayMessage:` 方法显示错误信息时播放 `bump!` 声音使得信息能吸引用户更多注意。

要播放声音我们能使用一个简单的 [NSSound](https://developer.apple.com/library/mac/Documentation/Cocoa/Reference/ApplicationKit/Classes/NSSound_Class/index.html) 类的接口。这是如何使用的例子：

```JavaScript
var filePath = sketch.scriptPath.stringByDeletingLastPathComponent()+"/assets/glass.aiff";

var sound = NSSound.alloc().initWithContentsOfFile_byReference(filePath,true);
doc.displayMessage("I'm Mr Meeseeks LOOK AT ME! :)")
sound.play();
```

> 重要说明：如果你想在 Sketch 应用的 MAS 版本之外的插件文件夹播放声音文件，你必须使用 [sketch-sandbox](https://github.com/bomberstudios/sketch-sandbox) 库来验证文件许可，因为 Sketch 版本是沙盒化的且禁止获取沙盒以外的文件权限。（译者注：该库已停止更新，因为Sketch 已放弃 MAS 版本，所以安装完插件后仅需将 assets 文件夹复制到插件目录即可播放声音）

完整例子：
- [Play Sound.sketchplugin](./Samples/Play Sound.sketchplugin)


可工作在：
- Sketch 3.0 +

## 在画布上居中矩形

要在确定的点或区域居中画布，你能使用一个有用的 `-(void)MSContentDrawView.centerRect:(GKRect*)rect animated:(BOOL)animated` 实例方法，参数 `rect` 是一个被居中的矩形，`animated` 是一个在滚动处理期间打开/关闭动画的标识。

![Create Custom Shape](./docs/center_rect.png)

你提供给这个方法的矩形原点（origin）和尺寸（size）理应是绝对坐标。

下面的例子是在 `x:200,y:200` 点居中视图：

```JavaScript
var view=doc.currentView();
var rect=GKRect.rectWithRect(NSMakeRect(200,200,1,1));
view.centerRect_animated(rect,true);
```
接下来的例子显示了如何使用相同的方法居中首个被选中层：

```JavaScript
var layer = selection.firstObject();
if(layer) {
    var view=doc.currentView();
    view.centerRect_animated(layer.absoluteRect(),true);
}
```

可工作在：
- Sketch 3.1 +

## 创建自定义形状

要用编程的方式创建一个自定义的矢量图形，你需要创建一个 [NSBezierPath](https://developer.apple.com/library/mac/Documentation/Cocoa/Reference/ApplicationKit/Classes/NSBezierPath_Class/index.html) 类的实例，和画你想要的任何形状或者混合的形状。然后从 `+(MSShapeGroup*)MSShapeGroup.shapeWithBezierPath:(NSBezierPath*)path` 类方法创建形状组。

![Create Custom Shape](./docs/create_custom_shape.png)

这个技巧非常类似于之前的自定义路径创建的技巧，唯一的不同就是在把它转化成形状组前你必须关闭路径。

下面的例子创建了一个简单地箭头形状：

```JavaScript
var doc = context.document;
var path = NSBezierPath.bezierPath();
path.moveToPoint(NSMakePoint(10,10));
path.lineToPoint(NSMakePoint(100,10));
path.lineToPoint(NSMakePoint(100,0));
path.lineToPoint(NSMakePoint(120,15));
path.lineToPoint(NSMakePoint(100,30));
path.lineToPoint(NSMakePoint(100,20));
path.lineToPoint(NSMakePoint(10,20));
path.closePath();

var shape = MSShapeGroup.shapeWithBezierPath(path);
//var fill = shape.style().fills().addNewStylePart();
/* 译者注：原文档上行代码已不再有效，fills().addNewStylePart()、borders().addNewStylePart() 类似方法在 Sketch 3.8 中已废弃，新方法：
layer.style().addStylePartOfType(0) // To add a new fill
layer.style().addStylePartOfType(1) // To add a new border
layer.style().addStylePartOfType(2) // To add a new shadow
layer.style().addStylePartOfType(3) // To add a new inner shadow
资料来自邮件列表：http://mail.sketchplugins.com/pipermail/dev_sketchplugins.com/2016-May/003781.html
*/
var fill = shape.style().addStylePartOfType(0);
fill.color = MSColor.colorWithSVGString("#dd0000");

doc.currentPage().addLayers([shape]);
```

完整例子：
- [Create Custom Shape.sketchplugin](./Samples/Create Custom Shape.sketchplugin)


可工作在：
- Sketch 3.2 +

## 创建线形

为了以编程方式创建一个线形，你要创建一个 [NSBezierPath](https://developer.apple.com/library/mac/Documentation/Cocoa/Reference/ApplicationKit/Classes/NSBezierPath_Class/index.html) 类的实例并添加两个点，然后使用 `+(MSShapeGroup*)MSShapeGroup.shapeWithBezierPath:(NSBezierPath*)path` 类方法创建一个形状组。

![Create Line Shape](./docs/create_line_shape.png)

要使 Sketch 认出作为线形提供的路径，你仅仅只要使用 `NSBezierPath` 的 `moveToPoint` & `lineToPoint` 方法添加两个点。

接下来是用两个点创建一个简单的线形的例子：

```JavaScript
var doc = context.document;

var path = NSBezierPath.bezierPath();
path.moveToPoint(NSMakePoint(10,10));
path.lineToPoint(NSMakePoint(200,200));

var shape = MSShapeGroup.shapeWithBezierPath(path);
// var border = shape.style().borders().addNewStylePart();
// 译者注：废弃 API 修改
var border = shape.style().addStylePartOfType(1);
border.color = MSColor.colorWithSVGString("#dd0000");
border.thickness = 2;

doc.currentPage().addLayers([shape]);
```

同样地方法，你能使用 [NSBezierPath](https://developer.apple.com/library/mac/Documentation/Cocoa/Reference/ApplicationKit/Classes/NSBezierPath_Class/index.html) 类提供的方法创建一个多段线。无论何时你添加超过两个点到路径，Sketch 对待矢量路径这样的形状类似于使用标准的 `V - Vector` 工具创建的东西。

接下来的例子演示如何使用四个点创建一条曲线：

```JavaScript
var doc = context.document;

var path = NSBezierPath.bezierPath();
path.moveToPoint(NSMakePoint(84.5,161));
[path curveToPoint:NSMakePoint(166,79.5) controlPoint1:NSMakePoint(129.5,161) controlPoint2:NSMakePoint(166,124.5)];
[path curveToPoint:NSMakePoint(84.5,-2) controlPoint1:NSMakePoint(166,34.5) controlPoint2:NSMakePoint(129.5,-2)];
[path curveToPoint:NSMakePoint(3,79.5) controlPoint1:NSMakePoint(39.5,-2) controlPoint2:NSMakePoint(3,34.5)];

var shape = MSShapeGroup.shapeWithBezierPath(path);
// var border = shape.style().borders().addNewStylePart();
// 译者注：废弃 API 修改
var border = shape.style().addStylePartOfType(1);
border.color = MSColor.colorWithSVGString("#dd0000");
border.thickness = 2;

doc.currentPage().addLayers([shape]);
```

完整例子：
- [Create Line Shape.sketchplugin](./Samples/Create Line Shape.sketchplugin)
- [Create Curved Line Shape.sketchplugin](./Samples/Create Curved Line Shape.sketchplugin)

工作在：
- Sketch 3.2 +

## 指定角设置圆角半径

从版本 3.2 开始，Sketch 允许为矩形指定角设置自定义圆角半径，3.2 之前也是可以的，只是没有直接的 API。

![Set Custom Border Radius](./docs/set_custom_border_radius_for_specific_corner.png)

你能使用 `-MSRectangleShape.setCornerRadiusFromComponents:(NSString*)compoents` 实例方法设置自定义半径， `components` 参数表示用 `/` 符号分隔的各个角半径值的字符串，字符串的顺序是：`左上/右上/右下/左下` 。

接下来的例子设置选中矩形的左上和右上角为 15 点（points）：

```JavaScript
var selection = context.selection;
var layer = selection.firstObject();
if(layer && layer.isKindOfClass(MSShapeGroup)) {
    var shape=layer.layers().firstObject();
    if(shape && shape.isKindOfClass(MSRectangleShape)) {
        shape.setCornerRadiusFromComponents("15/15/0/0");
    }
}
```

完整例子：
- [Set Border Radius From Components.sketchplugin](./Samples/Set Border Radius From Components.sketchplugin)

工作在：
- Sketch 3.2 +

## 缩放层

你能使用 `-MSLayer.multiplyBy:(double)scaleFactor` 实例方法缩放任意层， `scaleFactor` 参数是被用于所有层属性（包含 position，size 等）和样式属性（例如 border thickness，shadow等）的倍数。一些缩放因子的例子：`1.0 = 100%`， `2.5 = 250%`， `0.5 = 50%` 等。

这个方法和用标准的 [Scale](http://bohemiancoding.com/sketch/support/documentation/03-layer-basics/4-resizing-layers.html) 工具产生相同的结果，因此所有层类型类都继承自 `MSLayer` 类，你能使用这个方法缩放包括页面和画板在内的任意类型的层。

> 注意：在方法被调用后，`x` 和 `y` 位置值也将被倍乘。如果你要层在缩放后仍然在同样的位置，你要改变它的位置用合适的值。

![Finding Selection Bounds](./docs/scale_layers.png)


接下来的例子演示如何缩放首个被选中的层：
```JavaScript
var selection = context.selection;
var layer = selection.firstObject();
if(layer) {
    // Preserve layer center point.
    var midX=layer.frame().midX();
    var midY=layer.frame().midY();

    // Scale layer by 200%
    layer.multiplyBy(2.0);

    // Translate frame to the original center point.
    layer.frame().midX = midX;
    layer.frame().midY = midY;
}
```

工作在：
- Sketch 3.1 +

## 找到一组层（layer）的边界（bounds）

如果你想快速找到选中层或者任意层集合的边界矩形，有个非常便利的方法：`+(CGRect)MSLayerGroup.groupBoundsForLayers:(NSArray*)layers`。它接收一列层且返回 CGRect 结构。

![Scaling Layers](./docs/find_selection_bounds.png)

一个快速的例子来演示如何使用它：
```JavaScript
var selection = context.selection;
var bounds=MSLayerGroup.groupBoundsForLayers(selection);

print("x: "+bounds.origin.x);
print("y: "+bounds.origin.y);
print("width: "+bounds.size.width);
print("height: "+bounds.size.height);
```

工作在：
- Sketch 3.3 +

## 创建椭圆形

为了以编程方式创建一个椭圆形，你要创建一个 `MSOvalShape` 类的实例，设置它的 frame 且用 `MSShapeGroup`容器包裹起来。

![Create Oval Shape](./docs/create_oval_shape.png)

接下来的例子来演示如何使用它：
```JavaScript
var doc = context.document;
var ovalShape = MSOvalShape.alloc().init();
ovalShape.frame = MSRect.rectWithRect(NSMakeRect(0,0,100,100));

var shapeGroup=MSShapeGroup.shapeWithPath(ovalShape);
// var fill = shapeGroup.style().fills().addNewStylePart();
// 译者注：废弃 API 修改
var fill = shapeGroup.style().addStylePartOfType(0);
fill.color = MSColor.colorWithSVGString("#dd2020");

doc.currentPage().addLayers([shapeGroup]);
```

完整例子：
- [Create Oval Shape.sketchplugin](./Samples/Create Oval Shape.sketchplugin)

工作在：
- Sketch 3.1 +

## 编程方式创建共享样式

为了以编程方式创建一个共享样式，你使用 `-MSSharedLayerStyleContainer.addSharedStyleWithName:(NSString*)name firstInstance:(MSStyle*)style` 方法， `name` 是创建共享样式的名字，`style` 是用于要添加的共享样式模板的参考样式。

你能从绑定到一些层的已存在样式中创建一个共享样式，也能用自定义的 `MSStyle` 实例从头开始创建。

![Create Shared Style Programmically](./docs/create_shared_style_programmatically.png)

从选中层的样式中创建共享样式：
```JavaScript
var selection = context.selection;
var doc = context.document;
var layer=selection.firstObject();
if(layer) {
    var sharedStyles=doc.documentData().layerStyles();
    sharedStyles.addSharedStyleWithName_firstInstance("Custom Style",layer.style());
}
```

从头创建共享样式：
```JavaScript
var doc = context.document;
var sharedStyles=doc.documentData().layerStyles();

var style=MSStyle.alloc().init();
// var fill=style.fills().addNewStylePart();
// 译者注：废弃 API 修改
var fill = style.addStylePartOfType(0);
fill.color = MSColor.colorWithSVGString("#B1C151");

sharedStyles.addSharedStyleWithName_firstInstance("Custom Style 2",style);

doc.reloadInspector();
```

完整例子：
- [Create Shared Style From Selected Layer.sketchplugin](./Samples/Create Shared Style From Selected Layer.sketchplugin)
- [Create Shared Style Programmatically.sketchplugin](./Samples/Create Shared Style Programmatically.sketchplugin)

工作在：
- Sketch 3.1 +

## 'MSColor.colorWithHex:alpha:' 方法不见了？ :)

在 Sketch 3.2 之前有个真的漂亮且便利的 `MSColor.colorWithHex:alpha:` 类方法允许用16进制字符串创建 `MSColor` 类的实例，然而不幸的是在 Sketch 3.2 的正式版中该 API 被移除了。

好消息！存在该方法的替代：

```JavaScript
// Create color without alpha.
var color = MSColor.colorWithSVGString("#FF0000");
print(color);
// -> (r:1.000000 g:0.000000 b:0.000000 a:1.000000)

// Create color with alpha.
var color = MSColor.colorWithSVGString("#FF0000");
color.alpha = 0.2;
print(color);
// -> (r:1.000000 g:0.000000 b:0.000000 a:0.200000)
```

工作在：
- Sketch 3.0 +

## 弄平（Flatten）矢量层

如果你想弄平包含几个使用不同布尔操作到单层的子路径的复杂矢量层，你能用 `+MSShapeGroup.flatten` 方法。

![Flatten Vector Shape](./docs/flatten_vector_shape.png)

样例代码弄平首个被选中矢量层：
```JavaScript
var selection = context.selection;
var layer=selection.firstObject();
if(layer && layer.isKindOfClass(MSShapeGroup)) {
    layer.flatten();
}
```

完整例子：
- [Flatten Vector Layer.sketchplugin](./Samples/Flatten Vector Layer.sketchplugin)

工作在：
- Sketch 3.2 +

## Flatten Layers to Bitmap

### 注意：在 Sketch 3.3 中此样例已不工作

> 译者注：本段已不工作，故未翻

## 转换文本层为轮廓（Outlines）

为了转换已存在的 `MSTextLayer` 到 `MSShapeGroup` 层，你要得到文本的 `NSBezierPath` 描述，然后转换为 `MSShapeGroup` 层。

![Convert Text Layer to Outlines](./docs/convert_text_layer_to_outlines.png)

接下来的例子演示如何得到文本层的矢量轮廓并使用它从中创建矢量图形：

```JavaScript
function convertToOutlines(layer) {
    if(!layer.isKindOfClass(MSTextLayer)) return;

    var parent=layer.parentGroup();
    var shape=MSShapeGroup.shapeWithBezierPath(layer.bezierPathWithTransforms());

    shape.style = layer.style();
    var style=shape.style();
    if(!style.fill()) {
        // var fill=style.fills().addNewStylePart();
        // 译者注：废弃 API 修改
        var fill = style.addStylePartOfType(0);
        fill.color = MSColor.colorWithNSColor(layer.style().textStyle().attributes().NSColor);
    }

    var isSelected=layer.isSelected();
    shape.name = layer.name();
    shape.setIsSelected(isSelected);

    parent.removeLayer(layer);
    parent.addLayers([shape]);

    return shape;
}

var selection = context.selection;
var layer=selection.firstObject();
if(layer) {
    var vectorizedTextLayer=convertToOutlines(layer);
    print(vectorizedTextLayer);
}
```
完整例子：
- [Convert Text Layer to Outlines.sketchplugin](./Samples/Convert Text Layer to Outlines.sketchplugin)

工作在：
- Sketch 3.1 +

## 沿着形状路径获取点坐标

如果你要沿着路径分布一些形状，有个实现在 `NSBezierPath_Slopes` 类扩展中的便利方法 `-pointOnPathAtLength:` 。

该方法接收一个表示你想要得到点坐标路径的位置的 `double` 值，它返回一个点坐标的 `CGPoint` 结构。

![Ge points coords along shape path](./docs/getting_points_along_path.png)

下面的例子把形状路径分割成 15 段，且打印出它们的点坐标：

```JavaScript
var selection = context.selection;
var layer=selection.firstObject();
if(layer && layer.isKindOfClass(MSShapeGroup)) {

    var count=15;
    var path=layer.bezierPathWithTransforms();

    var step=path.length()/count;
    for(var i=0;i<=count;i++) {
        var point=path.pointOnPathAtLength(step*i);
        print(point);
    }
}
```
完整例子：
- [Get Points Coords Along Path.sketchplugin](./Samples/Get Points Coords Along Path.sketchplugin)
- [Create Dots Along Path.sketchplugin](./Samples/Create Dots Along Path.sketchplugin)

工作在：
- Sketch 3.2 +
