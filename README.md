
Sketch Plugins Cookbook
=======================

![学渣乱翻，仅为自用🙃](https://github.com/hbzs/FGP/raw/master/resource/transinfo.png) ![感觉会翻完耶😌](https://github.com/hbzs/FGP/raw/master/resource/trans2.png)

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

通常声音是令人厌烦且无用的，但有时小心使用的话还是非常有帮助的。

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
/* 译者注：fills().addNewStylePart()、borders().addNewStylePart() 类似方法在 Sketch 3.8 中已废弃，新方法：
layer.style().addStylePartOfType(0) // To add a new fill
layer.style().addStylePartOfType(1) // To add a new border
layer.style().addStylePartOfType(2) // To add a new shadow
layer.style().addStylePartOfType(3) // To add a new inner shadow
（吐槽：代码不改就算了，官方文档提都没提，起码写个已废弃，要改也成啊，还是在 issue 里提到的邮件列表找到的，有这时间，改个文档很难？我对 Sketch 新的收费规则下急需解决的新旧文档兼容问题深深担忧）
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

In order to create a line shape programmatically, you have to create an instance of [NSBezierPath](https://developer.apple.com/library/mac/Documentation/Cocoa/Reference/ApplicationKit/Classes/NSBezierPath_Class/index.html) class and add two points to it. Then create a shape group from it using `+(MSShapeGroup*)MSShapeGroup.shapeWithBezierPath:(NSBezierPath*)path` class method.

![Create Line Shape](./docs/create_line_shape.png)

To make Sketch recognize the provided path as a line shape, you have to add only two points using `moveToPoint` & `lineToPoint` methods of `NSBezierPath`.

The following example creates a simple line shape with two points:
```JavaScript
var doc = context.document;

var path = NSBezierPath.bezierPath();
path.moveToPoint(NSMakePoint(10,10));
path.lineToPoint(NSMakePoint(200,200));

var shape = MSShapeGroup.shapeWithBezierPath(path);
var border = shape.style().borders().addNewStylePart();
border.color = MSColor.colorWithSVGString("#dd0000");
border.thickness = 2;

doc.currentPage().addLayers([shape]);
```

The same way, you can easily create a multi segment line using methods provided by [NSBezierPath](https://developer.apple.com/library/mac/Documentation/Cocoa/Reference/ApplicationKit/Classes/NSBezierPath_Class/index.html) class. Whenever you add more than two points into the path, Sketch treats such shape as a vector path similar to what can be created using standard `V - Vector` tool.

The following example demonstrates how to create a curved path with four points:
```JavaScript
var doc = context.document;

var path = NSBezierPath.bezierPath();
path.moveToPoint(NSMakePoint(84.5,161));
[path curveToPoint:NSMakePoint(166,79.5) controlPoint1:NSMakePoint(129.5,161) controlPoint2:NSMakePoint(166,124.5)];
[path curveToPoint:NSMakePoint(84.5,-2) controlPoint1:NSMakePoint(166,34.5) controlPoint2:NSMakePoint(129.5,-2)];
[path curveToPoint:NSMakePoint(3,79.5) controlPoint1:NSMakePoint(39.5,-2) controlPoint2:NSMakePoint(3,34.5)];

var shape = MSShapeGroup.shapeWithBezierPath(path);
var border = shape.style().borders().addNewStylePart();
border.color = MSColor.colorWithSVGString("#dd0000");
border.thickness = 2;

doc.currentPage().addLayers([shape]);
```

Complete examples:
- [Create Line Shape.sketchplugin](./Samples/Create Line Shape.sketchplugin)
- [Create Curved Line Shape.sketchplugin](./Samples/Create Curved Line Shape.sketchplugin)

Works in:
- Sketch 3.2 +

## Set Border Radius for Specific Corners

Starting from version 3.2 Sketch allows to set custom border radius for specific corner of rectangle shape. It was possible prior to 3.2, but there was no direct API.

![Set Custom Border Radius](./docs/set_custom_border_radius_for_specific_corner.png)

In order to set custom radiuses you use `-MSRectangleShape.setCornerRadiusFromComponents:(NSString*)compoents` instance method, where `components` is a string that represents radius values for every corner separated by `/` sybmols. The sequence is following: `left-top/right-top/right-bottom/left-bottom`.

The following sample sets left-top and right-top corners of a selected rect shape to 15 points:
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

Complete examples:
- [Set Border Radius From Components.sketchplugin](./Samples/Set Border Radius From Components.sketchplugin)

Works in:
- Sketch 3.2 +

## Scaling Layers

You can scale any layer using `-MSLayer.multiplyBy:(double)scaleFactor` instance method, where `scaleFactor` is a floating-point value that is used to multiple all the layers' properties including position, size, and all the style attributes such as border thickness, shadow, etc. Here are some example scale factors: `1.0 = 100%`, `2.5 = 250%`, `0.5 = 50%`, etc.

This method produces the same result as a standard [Scale](http://bohemiancoding.com/sketch/support/documentation/03-layer-basics/4-resizing-layers.html) tool. Since all the layer type classes are inherited from `MSLayer` class, you can use this method to scale any type of layer including Pages and Artboards.

> Note: After the call of the method, `x` and `y` position values will also be multiplied. If you need the layer to remain in the same position after scaling, you'll have to change its position to the appropriate values.

![Finding Selection Bounds](./docs/scale_layers.png)


The following sample demonstrates how to scale first selected layer:
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

Works in:
- Sketch 3.1 +

## Finding Bounds For a Set of Layers

If you want to quickly find a bounding rectangle for selected layers or any set of layers, there is a very handy class method for that `+(CGRect)MSLayerGroup.groupBoundsForLayers:(NSArray*)layers`. It accepts a list of layers and returns CGRect structure.

![Scaling Layers](./docs/find_selection_bounds.png)

A quick sample that demonstrate how to use it:
```JavaScript
var selection = context.selection;
var bounds=MSLayerGroup.groupBoundsForLayers(selection);

print("x: "+bounds.origin.x);
print("y: "+bounds.origin.y);
print("width: "+bounds.size.width);
print("height: "+bounds.size.height);
```

Works in:
- Sketch 3.3 +

## Create Oval Shape

In order to create an oval shape programmatically, you have to create an instance of `MSOvalShape` class, set its frame and wrap with `MSShapeGroup` container.

![Create Oval Shape](./docs/create_oval_shape.png)

The following sample demonstrates how to do it:
```JavaScript
var doc = context.document;
var ovalShape = MSOvalShape.alloc().init();
ovalShape.frame = MSRect.rectWithRect(NSMakeRect(0,0,100,100));

var shapeGroup=MSShapeGroup.shapeWithPath(ovalShape);
var fill = shapeGroup.style().fills().addNewStylePart();
fill.color = MSColor.colorWithSVGString("#dd2020");

doc.currentPage().addLayers([shapeGroup]);
```

Complete examples:
- [Create Oval Shape.sketchplugin](./Samples/Create Oval Shape.sketchplugin)

Works in:
- Sketch 3.1 +

## Create Shared Style Programmatically

In order to create a shared style programmatically you use `-MSSharedLayerStyleContainer.addSharedStyleWithName:(NSString*)name firstInstance:(MSStyle*)style` method, where `name` is a name of shared style being created, `style` is a reference style used as a template for future shared style.

You can create a shared style from the existing style that is bound to some layer or create it from scratch with a custom `MSStyle` instance.

![Create Shared Style Programmically](./docs/create_shared_style_programmatically.png)

Create shared style from selected layers' style:
```JavaScript
var selection = context.selection;
var doc = context.document;
var layer=selection.firstObject();
if(layer) {
    var sharedStyles=doc.documentData().layerStyles();
    sharedStyles.addSharedStyleWithName_firstInstance("Custom Style",layer.style());
}
```

Create shared style from scratch:
```JavaScript
var doc = context.document;
var sharedStyles=doc.documentData().layerStyles();

var style=MSStyle.alloc().init();
var fill=style.fills().addNewStylePart();
fill.color = MSColor.colorWithSVGString("#B1C151");

sharedStyles.addSharedStyleWithName_firstInstance("Custom Style 2",style);

doc.reloadInspector();
```

Complete examples:
- [Create Shared Style From Selected Layer.sketchplugin](./Samples/Create Shared Style From Selected Layer.sketchplugin)
- [Create Shared Style Programmatically.sketchplugin](./Samples/Create Shared Style Programmatically.sketchplugin)

Works in:
- Sketch 3.1 +

## Missing 'MSColor.colorWithHex:alpha:'? :)

Prior to Sketch 3.2 there was a really nice and handy class method called `MSColor.colorWithHex:alpha:` that allowed to create instance of `MSColor` class with hex string, but unfortunately with the release of Sketch 3.2 version it was removed from the API.

Good news everyone! The replacement for this method does exist:
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

Works in:
- Sketch 3.0 +

## Flatten Vector Layer

If you want to flatten a complex vector layer that contains several sub paths combined using different boolean operations into single layer, you can use `+MSShapeGroup.flatten` method.

![Flatten Vector Shape](./docs/flatten_vector_shape.png)

This sample code flattens a first selected vector layer:
```JavaScript
var selection = context.selection;
var layer=selection.firstObject();
if(layer && layer.isKindOfClass(MSShapeGroup)) {
    layer.flatten();
}
```

Complete examples:
- [Flatten Vector Layer.sketchplugin](./Samples/Flatten Vector Layer.sketchplugin)

Works in:
- Sketch 3.2 +

## Flatten Layers to Bitmap

### Note: This example currently doesn't work in Sketch 3.3 

In order to flatten one or several layers of any type to a single `MSBitmapLayer`, use `-MSLayerFlattener.flattenLayers:` method. It accepts one arguments which is an array of layers to be flattened.

![Flatten Layers to Bitmap](./docs/flatten_layers_to_bitmap.png)

The following example flattens all the selected layers to a bitmap layer:
```JavaScript
var flattener = MSLayerFlattener.alloc().init();
flattener.flattenLayers(selection);
```
Complete examples:
- [Flatten Selection to Bitmap.sketchplugin](./Samples/Flatten Selection To Bitmap.sketchplugin)

Works in:
- Sketch 3.2 +

## Convert Text Layer to Outlines

In order to convert an existing `MSTextLayer` to `MSShapeGroup` layer, you have to get texts' `NSBezierPath` representation and then convert it to a `MSShapeGroup` layer.

![Convert Text Layer to Outlines](./docs/convert_text_layer_to_outlines.png)

The following source code demonstrates how to get text layers' vector outline and use it to create a vector shape from it:
```JavaScript
function convertToOutlines(layer) {
    if(!layer.isKindOfClass(MSTextLayer)) return;

    var parent=layer.parentGroup();
    var shape=MSShapeGroup.shapeWithBezierPath(layer.bezierPathWithTransforms());

    shape.style = layer.style();
    var style=shape.style();
    if(!style.fill()) {
        var fill=style.fills().addNewStylePart();
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
Complete examples:
- [Convert Text Layer to Outlines.sketchplugin](./Samples/Convert Text Layer to Outlines.sketchplugin)

Works in:
- Sketch 3.1 +

## Get Points Coords Along the Shape Path

If you want to distribute some shapes along a path there is a convenient method `-pointOnPathAtLength:` implemented in `NSBezierPath_Slopes` class extension.

This method accepts a `double` value that represents a position on path at which you want to get a point coordinate. It returns a `CGPoint` struct with coordinates of the point.

![Ge points coords along shape path](./docs/getting_points_along_path.png)

The following example divides shape path into 15 segments and prints out their points coordinates:
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
Complete examples:
- [Get Points Coords Along Path.sketchplugin](./Samples/Get Points Coords Along Path.sketchplugin)
- [Create Dots Along Path.sketchplugin](./Samples/Create Dots Along Path.sketchplugin)

Works in:
- Sketch 3.2 +
