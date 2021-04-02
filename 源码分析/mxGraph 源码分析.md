

# `mxGraph` 源码分析

> 本文基于 [`mxGraph`](https://github.com/jgraph/mxgraph) 的4.2.2版本展开。由于mxGraph体系庞大，本文主要是梳理核心架构与流程，不会面面俱到，深入到所有细节。mxGraph有Java、.net、JavaScript三个版本，但是实现方式是大致一样的，本文主要是基于JavaScript解读，熟悉Java、.net的同学也可以作为参考。

## 什么是`mxGraph`？

先看官方文档解释：

> mxGraph is a JavaScript diagramming library that enables` interactive graph` and `charting applications` to be quickly created that run natively in any major browser that is supported by its vendor.

另一种解释是：

> mxGraph is a `JavaScript component` that provides features aimed at applications that display `interactive diagrams` and `graphs`.Note by graphs we mean [`mathematical graphs`](http://en.wikipedia.org/wiki/Graph_(mathematics)), not necessarily [`charts`](http://en.wikipedia.org/wiki/Charts) (although some charts are graphs).

简单的说，`mxGraph`是一个用于构建可交互的，绘制基于图的数据结构的图例的应用程序的JavaScript库。

为了明晰`mxGraph`的定位，此处需要区分一下graph与chart的[区别](https://en.wikipedia.org/wiki/Wikipedia:Graphs_and_charts)：

**graph**

- Graph visualization is based on the mathematical theory of networks, graph theory. 
- Graphs must be accurate and convey information efficiently.
- A graph consists of vertices, also called nodes, and of edges (the connecting lines between the nodes).

![img](https://jgraph.github.io/mxgraph/docs/images/mx_man_simple_graph.png)

**chart**

- A **chart** is a [graphical representation](https://en.wikipedia.org/wiki/Graphics) for [data visualization](https://en.wikipedia.org/wiki/Data_visualization), in which "the data is represented by symbols, such as bars in a [bar chart](https://en.wikipedia.org/wiki/Bar_chart), lines in a [line chart](https://en.wikipedia.org/wiki/Line_chart), or slices in a [pie chart](https://en.wikipedia.org/wiki/Pie_chart)"。



## 项目结构

### 整体目录结构

```
.
├── docs
│   ├── css
│   ├── images
│   ├── js
│   └── js-api
│       ├── files
│       ├── images
│       ├── index
│       ├── javascript
│       ├── search
│       └── styles
├── dotnet
│   ├── aspnet
│   │   └── Properties
│   ├── docs
│   │   └── html
│   ├── examples
│   │   ├── Properties
│   │   └── src
│   ├── src
│   │   ├── canvas
│   │   ├── io
│   │   ├── layout
│   │   ├── model
│   │   ├── reader
│   │   ├── utils
│   │   └── view
│   └── test
├── etc
│   └── build
├── java
│   ├── docs
│   │   └── com
│   ├── examples
│   │   └── com
│   ├── jars
│   ├── lib
│   ├── src
│   │   └── com
│   └── test
│       └── com
└── javascript
    ├── examples
    │   ├── editors
    │   ├── grapheditor
    │   ├── images
    │   ├── map-background
    │   └── webpack
    └── src
        ├── css
        ├── images
        ├── js
        └── resources
```

- [docs](https://github.com/jgraph/mxgraph/tree/master/docs): mxGraph官方文档

- [dotnet](https://github.com/jgraph/mxgraph/tree/master/dotnet): 基于 .net 的实现，暂时无需关注

- [etc/build](https://github.com/jgraph/mxgraph/tree/master/etc/build): 这个目录下只有一份 `Gruntfile.js` 文件，用于构建生成环境版本

- [java](https://github.com/jgraph/mxgraph/tree/master/java): 基于 java 的实现，暂时无需关注

- [javascript](https://github.com/jgraph/mxgraph/tree/master/javascript): 基于JavaScript的实现，其中 `javascript/src` 为框架源码；`javascript/examples` 为示例代码。



### 核心目录结构

```
./javascript/src/js/
├── editor
│   ├── mxDefaultKeyHandler.js
│   ├── mxDefaultPopupMenu.js
│   ├── mxDefaultToolbar.js
│   └── mxEditor.js
├── handler
│   ├── mxCellHighlight.js
│   ├── mxCellMarker.js
│   ├── mxCellTracker.js
│   ├── mxConnectionHandler.js
│   ├── mxConstraintHandler.js
│   ├── mxEdgeHandler.js
│   ├── mxEdgeSegmentHandler.js
│   ├── mxElbowEdgeHandler.js
│   ├── mxGraphHandler.js
│   ├── mxHandle.js
│   ├── mxKeyHandler.js
│   ├── mxPanningHandler.js
│   ├── mxPopupMenuHandler.js
│   ├── mxRubberband.js
│   ├── mxSelectionCellsHandler.js
│   ├── mxTooltipHandler.js
│   └── mxVertexHandler.js
├── index.txt
├── io
│   ├── mxCellCodec.js
│   ├── mxChildChangeCodec.js
│   ├── mxCodec.js
│   ├── mxCodecRegistry.js
│   ├── mxDefaultKeyHandlerCodec.js
│   ├── mxDefaultPopupMenuCodec.js
│   ├── mxDefaultToolbarCodec.js
│   ├── mxEditorCodec.js
│   ├── mxGenericChangeCodec.js
│   ├── mxGraphCodec.js
│   ├── mxGraphViewCodec.js
│   ├── mxModelCodec.js
│   ├── mxObjectCodec.js
│   ├── mxRootChangeCodec.js
│   ├── mxStylesheetCodec.js
│   └── mxTerminalChangeCodec.js
├── layout
│   ├── hierarchical
│   │   ├── model
│   │   │   ├── mxGraphAbstractHierarchyCell.js
│   │   │   ├── mxGraphHierarchyEdge.js
│   │   │   ├── mxGraphHierarchyModel.js
│   │   │   ├── mxGraphHierarchyNode.js
│   │   │   └── mxSwimlaneModel.js
│   │   ├── mxHierarchicalLayout.js
│   │   ├── mxSwimlaneLayout.js
│   │   └── stage
│   │       ├── mxCoordinateAssignment.js
│   │       ├── mxHierarchicalLayoutStage.js
│   │       ├── mxMedianHybridCrossingReduction.js
│   │       ├── mxMinimumCycleRemover.js
│   │       └── mxSwimlaneOrdering.js
│   ├── mxCircleLayout.js
│   ├── mxCompactTreeLayout.js
│   ├── mxCompositeLayout.js
│   ├── mxEdgeLabelLayout.js
│   ├── mxFastOrganicLayout.js
│   ├── mxGraphLayout.js
│   ├── mxParallelEdgeLayout.js
│   ├── mxPartitionLayout.js
│   ├── mxRadialTreeLayout.js
│   └── mxStackLayout.js
├── model
│   ├── mxCell.js
│   ├── mxCellPath.js
│   ├── mxGeometry.js
│   └── mxGraphModel.js
├── mxClient.js
├── shape
│   ├── mxActor.js
│   ├── mxArrow.js
│   ├── mxArrowConnector.js
│   ├── mxCloud.js
│   ├── mxConnector.js
│   ├── mxCylinder.js
│   ├── mxDoubleEllipse.js
│   ├── mxEllipse.js
│   ├── mxHexagon.js
│   ├── mxImageShape.js
│   ├── mxLabel.js
│   ├── mxLine.js
│   ├── mxMarker.js
│   ├── mxPolyline.js
│   ├── mxRectangleShape.js
│   ├── mxRhombus.js
│   ├── mxShape.js
│   ├── mxStencil.js
│   ├── mxStencilRegistry.js
│   ├── mxSwimlane.js
│   ├── mxText.js
│   └── mxTriangle.js
├── util
│   ├── mxAbstractCanvas2D.js
│   ├── mxAnimation.js
│   ├── mxAutoSaveManager.js
│   ├── mxClipboard.js
│   ├── mxConstants.js
│   ├── mxDictionary.js
│   ├── mxDivResizer.js
│   ├── mxDragSource.js
│   ├── mxEffects.js
│   ├── mxEvent.js
│   ├── mxEventObject.js
│   ├── mxEventSource.js
│   ├── mxForm.js
│   ├── mxGuide.js
│   ├── mxImage.js
│   ├── mxImageBundle.js
│   ├── mxImageExport.js
│   ├── mxLog.js
│   ├── mxMorphing.js
│   ├── mxMouseEvent.js
│   ├── mxObjectIdentity.js
│   ├── mxPanningManager.js
│   ├── mxPoint.js
│   ├── mxPopupMenu.js
│   ├── mxRectangle.js
│   ├── mxResources.js
│   ├── mxSvgCanvas2D.js
│   ├── mxToolbar.js
│   ├── mxUndoManager.js
│   ├── mxUndoableEdit.js
│   ├── mxUrlConverter.js
│   ├── mxUtils.js
│   ├── mxVmlCanvas2D.js
│   ├── mxWindow.js
│   ├── mxXmlCanvas2D.js
│   └── mxXmlRequest.js
└── view
    ├── mxCellEditor.js
    ├── mxCellOverlay.js
    ├── mxCellRenderer.js
    ├── mxCellState.js
    ├── mxCellStatePreview.js
    ├── mxConnectionConstraint.js
    ├── mxEdgeStyle.js
    ├── mxGraph.js
    ├── mxGraphSelectionModel.js
    ├── mxGraphView.js
    ├── mxLayoutManager.js
    ├── mxMultiplicity.js
    ├── mxOutline.js
    ├── mxPerimeter.js
    ├── mxPrintPreview.js
    ├── mxStyleRegistry.js
    ├── mxStylesheet.js
    ├── mxSwimlaneManager.js
    └── mxTemporaryCellStates.js
```

- editor：基础编辑器的实现。
- handler：各种基于事件的功能插件。
- io：各种编译码器的实现，实现mxGraph实例的持久化。
- layout：布局类，实现各种布局逻辑。
- model：各种数据模型抽象。
- shape：各种图元实现。
- util：各种工具类。
- view：各种交互接口类。



## 抽象接口

整个项目主要是采取基于事件的编程模型，优势是松耦合，易扩展，劣势则是逻辑较分散，debug不方便。

### mxEventSource

[`mxEventSource`](https://jgraph.github.io/mxgraph/docs/js-api/files/util/mxEventSource-js.html) 作为一个基础类，实现了自定义事件的发布监听机制。实现逻辑比较简单，就是常规的订阅发布模式。

**addListener**

```javascript
mxEventSource.prototype.addListener = function(name, funct)
{
	if (this.eventListeners == null)
	{
		this.eventListeners = [];
	}	
	this.eventListeners.push(name);
	this.eventListeners.push(funct);
};
```

**fireEvent**

```javascript
mxEventSource.prototype.fireEvent = function(evt, sender)
{
	if (this.eventListeners != null && this.isEventsEnabled())
	{
		if (evt == null)
		{
			evt = new mxEventObject();
		}	
		if (sender == null)
		{
			sender = this.getEventSource();
		}
		if (sender == null)
		{
			sender = this;
		}
		var args = [sender, evt];
		for (var i = 0; i < this.eventListeners.length; i += 2)
		{
			var listen = this.eventListeners[i];
			if (listen == null || listen == evt.getName())
			{
				this.eventListeners[i+1].apply(this, args);
			}
		}
	}
};
```

**removeListener**

```javascript
mxEventSource.prototype.removeListener = function(funct)
{
	if (this.eventListeners != null)
	{
		var i = 0;		
		while (i < this.eventListeners.length)
		{
			if (this.eventListeners[i+1] == funct)
			{
				this.eventListeners.splice(i, 2);
			}
			else
			{
				i += 2;
			}
		}
	}
};
```

**mxEventObject**

自定义事件的数据结构：

```javascript
function mxEventObject(name)
{
	this.name = name;
	this.properties = [];
	
	for (var i = 1; i < arguments.length; i += 2)
	{
		if (arguments[i + 1] != null)
		{
			this.properties[arguments[i]] = arguments[i + 1];
		}
	}
};
```

其中 [mxGraphModel](https://jgraph.github.io/mxgraph/docs/js-api/files/model/mxGraphModel-js.html#mxGraphModel), [mxGraph](https://jgraph.github.io/mxgraph/docs/js-api/files/view/mxGraph-js.html#mxGraph), [mxGraphView](https://jgraph.github.io/mxgraph/docs/js-api/files/view/mxGraphView-js.html#mxGraphView), [mxEditor](https://jgraph.github.io/mxgraph/docs/js-api/files/editor/mxEditor-js.html#mxEditor), [mxCellOverlay](https://jgraph.github.io/mxgraph/docs/js-api/files/view/mxCellOverlay-js.html#mxCellOverlay), [mxToolbar](https://jgraph.github.io/mxgraph/docs/js-api/files/util/mxToolbar-js.html#mxToolbar), [mxWindow](https://jgraph.github.io/mxgraph/docs/js-api/files/util/mxWindow-js.html#mxWindow) 等核心类都继承了`mxEventSource`。

### mxCell

在mxGraph中，所有的节点可以分成Vertex（顶点）、Edge（连线）两类。[`mxCell`](https://jgraph.github.io/mxgraph/docs/js-api/files/model/mxCell-js.html) 是这两种节点的抽象模型，用于维护id、value、style、geometry等节点信息。

```javascript
function mxCell(value, geometry, style)
{
	this.value = value;
	this.setGeometry(geometry);
	this.setStyle(style);
	
	if (this.onInit != null)
	{
		this.onInit();
	}
};
```



### mxGeometry

`mxGeometry` 用于描述一个节点的位置、大小、偏移、旋转等几何信息。

`mxGeometry`的继承链： `mxGeometry` --> `mxRectangle` --> `mxPoint`。

```javascript
function mxGeometry(x, y, width, height)
{
	mxRectangle.call(this, x, y, width, height);
};
```

```javascript
function mxRectangle(x, y, width, height)
{
	mxPoint.call(this, x, y);
	this.width = (width != null) ? width : 0;
	this.height = (height != null) ? height : 0;
};
```

```javascript
function mxPoint(x, y)
{
	this.x = (x != null) ? x : 0;
	this.y = (y != null) ? y : 0;
};
```



### mxShape、mxStencil

`mxCell` 实例是节点的抽象类，用于维护id、value、style、geometry等节点信息，这些信息都是底层的辅助信息，我们实际看到的是这个节点“长”的怎么样。`mxShape` 就是描述节点形状的抽象类。

> `mxCell` 与 `mxShape` 的关系：类比各式各样的泥人与模具的关系。

`mxShape` 是所有shape类的基类，包括以XML形式描述shape的`mxStencil`：

```javascript
function mxShape(stencil)
{
	this.stencil = stencil;
	this.initStyles();
};
```

```javascript
// desc - XML node that contains the stencil description.
function mxStencil(desc)
{
	this.desc = desc;
	this.parseDescription();
	this.parseConstraints();
};
// Extends mxShape.
mxUtils.extend(mxStencil, mxShape);
```

下面以一个椭圆形的shape为例，看看一个具体的shape是如何实现的：

```javascript
function mxEllipse(bounds, fill, stroke, strokewidth)
{
	mxShape.call(this);
	this.bounds = bounds;
	this.fill = fill;
	this.stroke = stroke;
	this.strokewidth = (strokewidth != null) ? strokewidth : 1;
};

/**
 * Extends mxShape.
 */
mxUtils.extend(mxEllipse, mxShape);

/**
 * Function: paintVertexShape
 * 
 * Paints the ellipse shape.
 */
mxEllipse.prototype.paintVertexShape = function(c, x, y, w, h)
{
	c.ellipse(x, y, w, h);
	c.fillAndStroke();
};
```

### mxAbstractCanvas2D

所有画布的基类，实现了原生canvas的所有接口。

```javascript
function mxAbstractCanvas2D()
{
	this.converter = this.createUrlConverter();	
	this.reset();
};
mxAbstractCanvas2D.prototype.save = function()
{
	// ...
};
mxAbstractCanvas2D.prototype.moveTo = function(x, y)
{
	// ...
};
// ...
```

#### mxSvgCanvas2D

继承 `mxAbstractCanvas2D` ，实现调用canvas的API输出svg节点。

```javascript
function mxSvgCanvas2D(root, styleEnabled)
{
	mxAbstractCanvas2D.call(this);
	this.root = root;
  // ...
}
mxSvgCanvas2D.prototype.begin = function()
{
	mxAbstractCanvas2D.prototype.begin.apply(this, arguments);
	this.node = this.createElement('path');
};
mxSvgCanvas2D.prototype.stroke = function()
{
	this.addNode(false, true);
};
// ...
```





## 如何画一个“图”？

### 图的三要素

先看如何用mxGraph画出一个简单的[helloworld](https://jgraph.github.io/mxgraph/javascript/examples/helloworld.html)：

```javascript
// Creates the graph inside the given container
var graph = new mxGraph(container);
// Gets the default parent for inserting new cells. This
// is normally the first child of the root (ie. layer 0).
var parent = graph.getDefaultParent();
// Adds cells to the model in a single step
graph.getModel().beginUpdate();
try
{
  var v1 = graph.insertVertex(parent, null, 'Hello,', 20, 20, 80, 30);
  var v2 = graph.insertVertex(parent, null, 'World!', 200, 150, 80, 30);
  var e1 = graph.insertEdge(parent, null, '', v1, v2);
}
finally
{
  // Updates the display
  graph.getModel().endUpdate();
}
```

从上例子可以看出，mxGraph是一个核心类，函数接收一个dom节点，可以猜测出是起着提供“画布”的作用。`insertVertex` 跟 `insertEdge` 则分别是用于插入 `Vertex` 节点跟 `Edge` 连线。

这样，一个“图”的三要素就显而易见了，分别就是“画布(graph)”、“点(vertex)”、“线(edge)”。



### “画布”初始化

下面看mxGraph初始化的具体过程：

```javascript
function mxGraph(container, model, renderHint, stylesheet)
{
	// ...
	
	// Initializes the main members that do not require a container
	this.model = (model != null) ? model : new mxGraphModel();
	// ...
	this.cellRenderer = this.createCellRenderer();
	this.setSelectionModel(this.createSelectionModel());
	this.setStylesheet((stylesheet != null) ? stylesheet : this.createStylesheet());
	this.view = this.createGraphView();
	
	// Adds a graph model listener to update the view
	this.graphModelChangeListener = mxUtils.bind(this, function(sender, evt)
	{
		this.graphModelChanged(evt.getProperty('edit').changes);
	});
	this.model.addListener(mxEvent.CHANGE, this.graphModelChangeListener);

	// Installs basic event handlers with disabled default settings.
	this.createHandlers();
	
	// Initializes the display if a container was specified
	if (container != null)
	{
		this.init(container);
	}
	
	this.view.revalidate();
};
```

下面逐一分析各个变量的含义。

#### this.model

```javascript
this.model = (model != null) ? model : new mxGraphModel();
```

每一个mxGraph实例都会有一个对应的mxGraphModel实例，用于维护、存储实例graph的数据结构。通过model实例，可以实现对graph实例的持久化：

```javascript
var encoder = new mxCodec();
var result = encoder.encode(graph.getModel());
var xml = mxUtils.getXml(result);
```

当然，也可以逆操作，把一个xml解码为一个model实例：

```javascript
var doc = mxUtils.parseXml(xmlString);
var codec = new mxCodec(doc);
codec.decode(doc.documentElement, graph.getModel());
```

由于画布graph是可渲染、可编辑的，model实例又是维护跟存储graph实例的数据结构的，自然而然就可以推测出，graph的所有渲染、编辑等操作都需要通过model来实现：

```javascript
// Adds a graph model listener to update the view
this.graphModelChangeListener = mxUtils.bind(this, function(sender, evt)
{
this.graphModelChanged(evt.getProperty('edit').changes);
});

this.model.addListener(mxEvent.CHANGE, this.graphModelChangeListener);
```

每当model实例有所变化时，执行更新逻辑`mxGraph.prototype.graphModelChanged`：

```javascript
mxGraph.prototype.graphModelChanged = function(changes)
{
	for (var i = 0; i < changes.length; i++)
	{
		this.processChange(changes[i]);
	}

	this.updateSelection();
	this.view.validate();
	this.sizeDidChange();
};
```



#### this.cellRenderer

顾名思义，cellRenderer实例是后续“绘制”cell时候所需要的。

```javascript
this.cellRenderer = this.createCellRenderer();
```

该函数创建一个mxCellRenderer实例：

```javascript
mxGraph.prototype.createCellRenderer = function()
{
	return new mxCellRenderer();
};
```

而且该类构造器函数是空的：

```javascript
function mxCellRenderer() { };
```

默认注册的shape（图元）：

```javascript
mxCellRenderer.registerShape = function(key, shape)
{
	mxCellRenderer.defaultShapes[key] = shape;
};

// Adds default shapes into the default shapes array
mxCellRenderer.registerShape(mxConstants.SHAPE_RECTANGLE, mxRectangleShape);
mxCellRenderer.registerShape(mxConstants.SHAPE_ELLIPSE, mxEllipse);
mxCellRenderer.registerShape(mxConstants.SHAPE_RHOMBUS, mxRhombus);
mxCellRenderer.registerShape(mxConstants.SHAPE_CYLINDER, mxCylinder);
mxCellRenderer.registerShape(mxConstants.SHAPE_CONNECTOR, mxConnector);
mxCellRenderer.registerShape(mxConstants.SHAPE_ACTOR, mxActor);
mxCellRenderer.registerShape(mxConstants.SHAPE_TRIANGLE, mxTriangle);
mxCellRenderer.registerShape(mxConstants.SHAPE_HEXAGON, mxHexagon);
mxCellRenderer.registerShape(mxConstants.SHAPE_CLOUD, mxCloud);
mxCellRenderer.registerShape(mxConstants.SHAPE_LINE, mxLine);
mxCellRenderer.registerShape(mxConstants.SHAPE_ARROW, mxArrow);
mxCellRenderer.registerShape(mxConstants.SHAPE_ARROW_CONNECTOR, mxArrowConnector);
mxCellRenderer.registerShape(mxConstants.SHAPE_DOUBLE_ELLIPSE, mxDoubleEllipse);
mxCellRenderer.registerShape(mxConstants.SHAPE_SWIMLANE, mxSwimlane);
mxCellRenderer.registerShape(mxConstants.SHAPE_IMAGE, mxImageShape);
mxCellRenderer.registerShape(mxConstants.SHAPE_LABEL, mxLabel);
```



#### this.view

```javascript
this.view = this.createGraphView();
```

该函数创建一个mxGraphView实例，同样， 一个mxGraph实例对应一个view实例：

```javascript
mxGraph.prototype.createGraphView = function()
{
	return new mxGraphView(this);
};
```

继续看构造函数的内容：

```javascript
function mxGraphView(graph)
{
	this.graph = graph;
	this.translate = new mxPoint();
	this.graphBounds = new mxRectangle();
	this.states = new mxDictionary();
};
```

其中`this.states`维护着所有节点cell的状态实例，先看实例内容：

```javascript
function mxCellState(view, cell, style)
{
	this.view = view;
	this.cell = cell;
	this.style = (style != null) ? style : {};
	
	this.origin = new mxPoint();
	this.absoluteOffset = new mxPoint();
};
```

view实例负责维护graph实例中各个节点的绝对坐标跟样式，并将它们缓存在`mxCellStates`中以进行更快的检索。每当model或cellState（平移，缩放）发生变化时，状态就会更新。



#### createHandlers

mxGraph提供了基于事件的handler接口，在此基础上可以实现功能代码的可插拔式配置。

```javascript
// Installs basic event handlers with disabled default settings.
this.createHandlers();
```

其中内置了几个默认的handler，并且默认禁用：

```javascript
mxGraph.prototype.createHandlers = function()
{
	this.tooltipHandler = this.createTooltipHandler();
	this.tooltipHandler.setEnabled(false);
	this.selectionCellsHandler = this.createSelectionCellsHandler();
	this.connectionHandler = this.createConnectionHandler();
	this.connectionHandler.setEnabled(false);
	this.graphHandler = this.createGraphHandler();
	this.panningHandler = this.createPanningHandler();
	this.panningHandler.panningEnabled = false;
	this.popupMenuHandler = this.createPopupMenuHandler();
};
```



#### init

经过前面的铺垫，终于来到真正初始化的环节：

```javascript
// Initializes the display if a container was specified
if (container != null)
{
	this.init(container);
}
```

需要传入一个dom节点：

```javascript
mxGraph.prototype.init = function(container)
{
	this.container = container;
	
	// Initializes the in-place editor
	this.cellEditor = this.createCellEditor();	

	// Initializes the container using the view
	this.view.init();
	
	// Updates the size of the container for the current graph
	this.sizeDidChange();
	
	// ...
};
```

##### this.cellEditor

该对象指向一个mxCellEditor实例：

```javascript
mxGraph.prototype.createCellEditor = function()
{
	return new mxCellEditor(this);
};
```

该实例实现内置的编辑功能，主要用于编辑cell的label。下面看其构造函数：

```javascript
function mxCellEditor(graph)
{
	this.graph = graph;
	
	// Stops editing after zoom changes
	this.zoomHandler = mxUtils.bind(this, function()
	{
		if (this.graph.isEditing())
		{
			this.resize();
		}
	});
	
	this.graph.view.addListener(mxEvent.SCALE, this.zoomHandler);
	this.graph.view.addListener(mxEvent.SCALE_AND_TRANSLATE, this.zoomHandler);
	
	// Adds handling of deleted cells while editing
	this.changeHandler = mxUtils.bind(this, function(sender)
	{
		if (this.editingCell != null && this.graph.getView().getState(this.editingCell) == null)
		{
			this.stopEditing(true);
		}
	});

	this.graph.getModel().addListener(mxEvent.CHANGE, this.changeHandler);
};
```

构造器内容比较简单，分别监听view实例跟model实例的相关变化事件，然后实现相应的操作。

##### this.view.init

```javascript
mxGraphView.prototype.init = function()
{
  // Installs the required listeners in the container.
	this.installListeners();
	
	// Creates the DOM nodes for the respective display dialect
	var graph = this.graph;
	
	if (graph.dialect == mxConstants.DIALECT_SVG)
	{
		this.createSvg();
	}
	else if (graph.dialect == mxConstants.DIALECT_VML)
	{
		this.createVml();
	}
	else
	{
		this.createHtml();
	}
};
```

默认是svg方言：

```javascript
mxGraphView.prototype.createSvg = function()
{
	var container = this.graph.container;
	this.canvas = document.createElementNS(mxConstants.NS_SVG, 'g');
	
	// For background image
	this.backgroundPane = document.createElementNS(mxConstants.NS_SVG, 'g');
	this.canvas.appendChild(this.backgroundPane);

	// Adds two layers (background is early feature)
	this.drawPane = document.createElementNS(mxConstants.NS_SVG, 'g');
	this.canvas.appendChild(this.drawPane);

	this.overlayPane = document.createElementNS(mxConstants.NS_SVG, 'g');
	this.canvas.appendChild(this.overlayPane);
	
	this.decoratorPane = document.createElementNS(mxConstants.NS_SVG, 'g');
	this.canvas.appendChild(this.decoratorPane);
	
	var root = document.createElementNS(mxConstants.NS_SVG, 'svg');
	// set root style 
  // ...
	if (container != null)
	{
		container.appendChild(root);
		this.updateContainerStyle(container);
	}
};
```

svg节点分成四层g节点，其中`backgroundPane`负责展示背景，`drawPane`负责展示真正内容，`overlayPane` 与 `decoratorPane` 节点负责高亮、预览等辅助内容等展示。



#### revalidate

初始化的最后一步，全量校验所有cell的cellState：

```javascript
// revalidates the complete view with all cell states.
this.view.revalidate();
```

其中包含两个方法：

```javascript
mxGraphView.prototype.revalidate = function()
{
	this.invalidate();
	this.validate();
};
```

##### invalidate

先看invalidate，用于重置校验状态：

```javascript
mxGraphView.prototype.invalidate = function(cell, recurse, includeEdges)
{
	// ...
	
	var state = this.getState(cell);
	
	if (state != null)
	{
		state.invalid = true;
	}
	
	// Avoids infinite loops for invalid graphs
	if (!cell.invalidating)
	{
		cell.invalidating = true;
		
		// Recursively invalidates all descendants
		if (recurse)
		{
			var childCount = model.getChildCount(cell);
			for (var i = 0; i < childCount; i++)
			{
				var child = model.getChildAt(cell, i);
				this.invalidate(child, recurse, includeEdges);
			}
		}
		
		// Propagates invalidation to all connected edges
		if (includeEdges)
		{
			var edgeCount = model.getEdgeCount(cell);
			for (var i = 0; i < edgeCount; i++)
			{
				this.invalidate(model.getEdgeAt(cell, i), recurse, includeEdges);
			}
		}
		
		delete cell.invalidating;
	}
};
```

核心逻辑就是重置cellState的状态，`state.invalid = true;`，并且默认递归处理子节点跟相关edge节点。`cell.invalidating` 属性主要是为了避免递归陷入死循环。



##### validate

在invalidate方法重置cellState状态之后，validate方法用于维护有效的cellState:

```javascript
mxGraphView.prototype.validate = function(cell)
{
	
	// ...
	
	var graphBounds = this.getBoundingBox(this.validateCellState(
		this.validateCell(cell || ((this.currentRoot != null) ?
			this.currentRoot : this.graph.getModel().getRoot()))));
	this.setGraphBounds((graphBounds != null) ? graphBounds : this.getEmptyBounds());
	this.validateBackground();
	
	// ...
};
```

其中核心逻辑是调用 `validateCell` 跟 `validateCellState` 方法。

###### validateCell

此方法递归创建`visible === true`的cell的cellState，反之则通过`removeState`从view.states缓存中移除：

```javascript
mxGraphView.prototype.validateCell = function(cell, visible)
{
	visible = (visible != null) ? visible : true;
	if (cell != null)
	{
		visible = visible && this.graph.isCellVisible(cell);
		var state = this.getState(cell, visible);
		
		if (state != null && !visible)
		{
			this.removeState(cell);
		}
		else
		{
			var model = this.graph.getModel();
			var childCount = model.getChildCount(cell);
			for (var i = 0; i < childCount; i++)
			{
				this.validateCell(model.getChildAt(cell, i), visible &&
					(!this.isCellCollapsed(cell) || cell == this.currentRoot));
			}
		}
	}	
	return cell;
};
```

###### validateCellState

此方法用来递归重绘`state.invalid === true`的节点：

```javascript
mxGraphView.prototype.validateCellState = function(cell, recurse)
{
	recurse = (recurse != null) ? recurse : true;
	var state = null;
	if (cell != null)
	{
		state = this.getState(cell);
		
		if (state != null)
		{
			var model = this.graph.getModel();
			// state.invalid === true 意味着state已被重置，需要重新渲染
			if (state.invalid)
			{
				state.invalid = false;		
				
        // ...
				
        // 更新cellState
				this.updateCellState(state);
				
				// Repaint happens immediately after the cell is validated
				if (cell != this.currentRoot && !state.invalid)
				{ 
          // 重绘节点
					this.graph.cellRenderer.redraw(state, false, this.isRendering());

					// Handles changes to invertex paintbounds after update of rendering shape
					state.updateCachedBounds();
				}
			}
			// 递归处理
			if (recurse && !state.invalid)
			{
				// Updates order in DOM if recursively traversing
				if (state.shape != null)
				{
					this.stateValidated(state);
				}
			
				var childCount = model.getChildCount(cell);
				
				for (var i = 0; i < childCount; i++)
				{
					this.validateCellState(model.getChildAt(cell, i));
				}
			}
		}
	}
	
	return state;
};
```

重绘语句是`this.graph.cellRenderer.redraw(state, false, this.isRendering());` 具体逻辑在图元绘制环节进一步探讨。

到这里，画布初始化完成。



### 插入一个cell

“画布”准备就绪，自然就到了挥毫泼墨到环节了，下面看看如何插入一个cell，以插入一个vertex为例。

#### insertVertex

```javascript
mxGraph.prototype.insertVertex = function(parent, id, value,
	x, y, width, height, style, relative)
{
	var vertex = this.createVertex(parent, id, value, x, y, width, height, style, relative);

	return this.addCell(vertex, parent);
};
```

先创建一个vertex实例，然后调用 `addCell` 方法，实际上是调用`addCells` :

```javascript
mxGraph.prototype.addCell = function(cell, parent, index, source, target)
{
	return this.addCells([cell], parent, index, source, target)[0];
};
```

此方法可以批量插入多个子cell：

```javascript
mxGraph.prototype.addCells = function(cells, parent, index, source, target, absolute)
{
	// ...
	this.model.beginUpdate();
	try
	{
		this.cellsAdded(cells, parent, index, source, target, (absolute != null) ? absolute : false, true);
		this.fireEvent(new mxEventObject(mxEvent.ADD_CELLS, 'cells', cells,
				'parent', parent, 'index', index, 'source', source, 'target', target));
	}
	finally
	{
		this.model.endUpdate();
	}
	return cells;
};
```

此时通过`this.model.beginUpdate()` 开启事务。继续看`cellsAdded` 方法：

```javascript
mxGraph.prototype.cellsAdded = function(cells, parent, index, source, target, absolute, constrain, extend)
{
	if (cells != null && parent != null && index != null)
	{
		this.model.beginUpdate();
		try
		{
      // ...
			for (var i = 0; i < cells.length; i++)
			{
				  // ...      
					this.model.add(parent, cells[i], index + i);			
					// ...
			}	
		}
		finally
		{
			this.model.endUpdate();
		}
	}
};
```

#### model.add

事务嵌套，最终是循环调用 `model.add` 函数：

```javascript
mxGraphModel.prototype.add = function(parent, child, index)
{
	if (child != parent && parent != null && child != null)
	{	
		// ...
		this.execute(new mxChildChange(this, parent, child, index));
		// ...
	}
	return child;
};
```

到这里，就涉及到model的事务机制了，在理解这段代码之前，需要理解整个事务流程。



## 事务管理机制

> 在很多情况下，我们需要执行一个复杂的操作，这个复杂操作可能由一个或者多个原子性的操作组成，这个复杂的操作称为一个事务。

> 为什么需要事务呢？虽然一个事务是由很多原子操作组成，但是宏观层面来看，只是一个复杂的操作而已。如果这个操作成功，意味着所有的子操作都是成功的；如果某个子操作失败，这个复杂的操作也应该是失败的，此时所有的子操作应该全部回滚，恢复到操作前的状态。

下面继续看前面提到helloworld的例子：

```javascript
graph.getModel().beginUpdate();
try
{
  var v1 = graph.insertVertex(parent, null, 'Hello,', 20, 20, 80, 30);
  var v2 = graph.insertVertex(parent, null, 'World!', 200, 150, 80, 30);
  var e1 = graph.insertEdge(parent, null, '', v1, v2);
}
finally
{
  // Updates the display
  graph.getModel().endUpdate();
}
```

通过`beginUpdate` 开启一个事务，中间有三个原子性的插入节点操作，最后`endUpdate`结束事务。

model实例上使用了事务的接口：

- add(parent, child, index)
- remove(cell)
- setCollapsed(cell, collapsed)
- setGeometry(cell, geometry)
- setRoot(root)
- setStyle(cell, style)
- setTerminal(cell, terminal, isSource)
- setTerminals(edge,source,target)
- setValue(cell, value)
- setVisible(cell, visible)

### 事务嵌套

既然一个事务可以包含多个子操作，事务嵌套是必须要支持的，下面看具体实现：

**beginUpdate**

```javascript
mxGraphModel.prototype.updateLevel = 0;
mxGraphModel.prototype.beginUpdate = function()
{
	this.updateLevel++;
	this.fireEvent(new mxEventObject(mxEvent.BEGIN_UPDATE));
	
	if (this.updateLevel == 1)
	{
		this.fireEvent(new mxEventObject(mxEvent.START_EDIT));
	}
};
```

`updateLevel`默认为0，每开启一个事务，就自增一次。**嵌套的事务是一个事务**。

**endUpdate**

```javascript
mxGraphModel.prototype.endingUpdate = false;
mxGraphModel.prototype.endUpdate = function()
{
	this.updateLevel--;
	if (this.updateLevel == 0)
	{
		this.fireEvent(new mxEventObject(mxEvent.END_EDIT));
	}
	
	if (!this.endingUpdate)
	{
		this.endingUpdate = this.updateLevel == 0;
		this.fireEvent(new mxEventObject(mxEvent.END_UPDATE, 'edit', this.currentEdit));

		try
		{	
      // 当 updateLevel === 0 时才会执行
			if (this.endingUpdate && !this.currentEdit.isEmpty())
			{
				this.fireEvent(new mxEventObject(mxEvent.BEFORE_UNDO, 'edit', this.currentEdit));
				var tmp = this.currentEdit;
				this.currentEdit = this.createUndoableEdit();
				tmp.notify();
				this.fireEvent(new mxEventObject(mxEvent.UNDO, 'edit', tmp));
			}
		}
		finally
		{
			this.endingUpdate = false;
		}
	}
};
```

`endingUpdate` 默认为false，只有 `this.updateLevel == 0` 时，也即是嵌套的事务中最后一个`endUpdate`执行时，事务的逻辑才会执行。



### 事务执行

下面具体看事务执行逻辑：

```javascript
var tmp = this.currentEdit;
// 创建新的实例，收集下一次事务的各类change
this.currentEdit = this.createUndoableEdit();
// 触发更新
tmp.notify();
// 发布此次更新实例到undoManager中心
this.fireEvent(new mxEventObject(mxEvent.UNDO, 'edit', tmp));
```

#### createUndoableEdit

每一次事务都创建一个新的mxUndoableEdit实例`currentEdit`：

```javascript
mxGraphModel.prototype.createUndoableEdit = function(significant)
{
	var edit = new mxUndoableEdit(this, (significant != null) ? significant : true);
	edit.notify = function()
	{
		// LATER: Remove changes property (deprecated)
		edit.source.fireEvent(new mxEventObject(mxEvent.CHANGE,
			'edit', edit, 'changes', edit.changes));
		edit.source.fireEvent(new mxEventObject(mxEvent.NOTIFY,
			'edit', edit, 'changes', edit.changes));
	};
	return edit;
};
```

通过 `tmp.notify()` 触发一个`mxEvent.CHANGE`事件，传过去`currentEdit` 实例。

#### execute

在触发notify执行事务之前，需要收集各个原子操作的封装对象，也就是各类change实例（下文有描述）。先看实现：

```javascript
mxGraphModel.prototype.execute = function(change)
{
	change.execute();
	this.beginUpdate();
	this.currentEdit.add(change);
	this.fireEvent(new mxEventObject(mxEvent.EXECUTE, 'change', change));
	// New global executed event
	this.fireEvent(new mxEventObject(mxEvent.EXECUTED, 'change', change));
	this.endUpdate();
};
```

`change.execute()` 用来执行change的更新逻辑，此时所有的更新只在model层面，还没有真正更新到页面上，等到notify之后才会更新到页面中。`currentEdit.add(change)`  用来收集此change实例到`currentEdit`实例中。

#### graphModelChanged

前面在 `mxGraph` 初始化的时候，已经监听了 `mxEvent.CHANGE` 事件，代码再贴一次：

```javascript
// Adds a graph model listener to update the view
this.graphModelChangeListener = mxUtils.bind(this, function(sender, evt)
{
this.graphModelChanged(evt.getProperty('edit').changes);
});
this.model.addListener(mxEvent.CHANGE, this.graphModelChangeListener);
```

`graphModelChanged` 接收change实例数组：

```javascript
mxGraph.prototype.graphModelChanged = function(changes)
{
	for (var i = 0; i < changes.length; i++)
	{
		this.processChange(changes[i]);
	}
	this.updateSelection();
	this.view.validate();
	this.sizeDidChange();
};
```

`processChange` 方法用于处理各类change实例，并且重置（invalidate）view实例中缓存cellState的状态：

```javascript
mxGraph.prototype.processChange = function(change)
{
	// Resets the view settings, removes all cells and clears
	// the selection if the root changes.
	if (change instanceof mxRootChange)
	{
		this.clearSelection();
		this.setDefaultParent(null);
		this.removeStateForCell(change.previous);
		// ...
	}
	
	// Adds or removes a child to the view by online invaliding
	// the minimal required portions of the cache, namely, the
	// old and new parent and the child.
	else if (change instanceof mxChildChange)
	{
		var newParent = this.model.getParent(change.child);
		this.view.invalidate(change.child, true, true);
		// ...
	}

	// Handles two special cases where the shape does not need to be
	// recreated from scratch, it only needs to be invalidated.
	else if (change instanceof mxTerminalChange || change instanceof mxGeometryChange)
	{
		// Checks if the geometry has changed to avoid unnessecary revalidation
		if (change instanceof mxTerminalChange || ((change.previous == null && change.geometry != null) ||
			(change.previous != null && !change.previous.equals(change.geometry))))
		{
			this.view.invalidate(change.cell);
		}
	}

	// ...
};
```

主要逻辑就是先对各个change中的cell进行 `view.invalidate(cell)`，然后整体 `view.validate()` 进行重绘，从而实现局部更新。



### 事件回滚（撤销、重做）

#### mxUndoableEdit

`mxUndoableEdit` 类是组合式的、可撤销的修改的一种实现。在一次事务中，它收集各个原子操作的change，基于此实现了可撤销，可重做的功能。下面看具体实现：

```javascript
function mxUndoableEdit(source, significant)
{
	this.source = source;
	this.changes = [];
	this.significant = (significant != null) ? significant : true;
};
```

此时的source就是model实例。

```javascript
mxUndoableEdit.prototype.add = function(change)
{
	this.changes.push(change);
};
```

通过add方法收集各类change实例。

**undo**

代码很简单，撤销操作的时候，其实就是执行各个change实例的execute或者undo方法。

```javascript
mxUndoableEdit.prototype.undone = false;
mxUndoableEdit.prototype.undo = function()
{
	if (!this.undone)
	{
		this.source.fireEvent(new mxEventObject(mxEvent.START_EDIT));
		var count = this.changes.length
		for (var i = count - 1; i >= 0; i--)
		{
			var change = this.changes[i];
			if (change.execute != null)
			{
				change.execute();
			}
			else if (change.undo != null)
			{
				change.undo();
			}	
			// New global executed event
			this.source.fireEvent(new mxEventObject(mxEvent.EXECUTED, 'change', change));
		}
		this.undone = true;
		this.redone = false;
		this.source.fireEvent(new mxEventObject(mxEvent.END_EDIT));
	}
	this.notify();
};
```

**redo**

与undo同理，重做操作的时候，其实就是执行各个change实例的execute或者redo方法。

```javascript
mxUndoableEdit.prototype.redone = false;
mxUndoableEdit.prototype.redo = function()
{
	if (!this.redone)
	{
		this.source.fireEvent(new mxEventObject(mxEvent.START_EDIT));
		var count = this.changes.length;
		for (var i = 0; i < count; i++)
		{
			var change = this.changes[i];	
			if (change.execute != null)
			{
				change.execute();
			}
			else if (change.redo != null)
			{
				change.redo();
			}		
			// New global executed event
			this.source.fireEvent(new mxEventObject(mxEvent.EXECUTED, 'change', change));
		}	
		this.undone = false;
		this.redone = true;
		this.source.fireEvent(new mxEventObject(mxEvent.END_EDIT));
	}
	this.notify();
};
```

#### 各类change

由上可知，在执行撤销或者重做操作时，实际上是执行各个change实例的`execute`方法，没有`execute`方法时，则分别执行其`undo`、`redo` 方法。下面具体看两个change类。

##### mxGeometryChange

移动cell几何位置时，用此类封装此类操作：

```javascript
function mxGeometryChange(model, cell, geometry)
{
	this.model = model;
	this.cell = cell;
	this.geometry = geometry;
	this.previous = geometry;
};
```

通过previous保存上一次操作的状态。

```javascript
mxGeometryChange.prototype.execute = function()
{
	if (this.cell != null)
	{
		this.geometry = this.previous;
		this.previous = this.model.geometryForCellChanged(
			this.cell, this.previous);
	}
};
```

`execute` 执行时，对调`geometry`、 `previous`，从而实现undo、redo交替时位置状态的切换。

##### mxChildChange

新增或者删除cell时，用此类封装此类操作：

```javascript
function mxChildChange(model, parent, child, index)
{
	this.model = model;
	this.parent = parent;
	this.previous = parent;
	this.child = child;
	this.index = index;
	this.previousIndex = index;
};
```

通过previous，previousIndex保存上一次操作的状态。

```javascript
mxChildChange.prototype.execute = function()
{
	if (this.child != null)
	{
		var tmp = this.model.getParent(this.child);
		var tmp2 = (tmp != null) ? tmp.getIndex(this.child) : 0;
		if (this.previous == null)
		{
			this.connect(this.child, false);
		}
    // 通过此处在model中插入cell或者删除cell
		tmp = this.model.parentForCellChanged(
			this.child, this.previous, this.previousIndex);	
		if (this.previous != null)
		{
			this.connect(this.child, true);
		}
		this.parent = this.previous;
		this.previous = tmp;
		this.index = this.previousIndex;
		this.previousIndex = tmp2;
	}
};
```

`execute` 执行时，分别对调`parent`、 `previous` 和 `index`、`previousIndex`，从而实现undo、redo交替时节点新增删除状态的切换。



#### mxUndoManager

各类`change` 实现了各类原子操作的封装， `mxUndoableEdit` 实现了事务的封装，要实现撤销、重做，自然还需要一个记录、管理事务的管理器，`mxUndoManager` 正是由此而来。

前面介绍事务执行逻辑代码时已经提到，每执行一个事务都会发布到管理中心：

```javascript
// 发布此次更新实例到undoManager中心
this.fireEvent(new mxEventObject(mxEvent.UNDO, 'edit', tmp));
```

管理中心记录事务：

```javascript
this.undoManager = new mxUndoManager();
mxEditor.prototype.installUndoHandler = function (graph)
{
	var listener = mxUtils.bind(this, function(sender, evt)
	{
		var edit = evt.getProperty('edit');
		this.undoManager.undoableEditHappened(edit);
	});
	// 记录model实例发布过来的事务
	graph.getModel().addListener(mxEvent.UNDO, listener);
	graph.getView().addListener(mxEvent.UNDO, listener);

	// Keeps the selection state in sync
	var undoHandler = function(sender, evt)
	{
		var changes = evt.getProperty('edit').changes;
		graph.setSelectionCells(graph.getSelectionCellsForChanges(changes));
	};

	this.undoManager.addListener(mxEvent.UNDO, undoHandler);
	this.undoManager.addListener(mxEvent.REDO, undoHandler);
};
```

**构造函数**

```javascript
function mxUndoManager(size)
{
	this.size = (size != null) ? size : 100;
	this.clear();
};
mxUndoManager.prototype = new mxEventSource();
mxUndoManager.prototype.constructor = mxUndoManager;
```

**trim**

新增一个事务时，移除当前指针位置到栈顶的事务。

```javascript
mxUndoManager.prototype.trim = function()
{
	if (this.history.length > this.indexOfNextAdd)
	{
		var edits = this.history.splice(this.indexOfNextAdd,
			this.history.length - this.indexOfNextAdd);
			
		for (var i = 0; i < edits.length; i++)
		{
			edits[i].die();
		}
	}
};
```

**undoableEditHappened**

记录新事务，入栈，指针指向栈顶。

```javascript
mxUndoManager.prototype.undoableEditHappened = function(undoableEdit)
{
	this.trim();
	if (this.size > 0 &&
		this.size == this.history.length)
	{
		this.history.shift();
	}
	this.history.push(undoableEdit);
	this.indexOfNextAdd = this.history.length;
	this.fireEvent(new mxEventObject(mxEvent.ADD, 'edit', undoableEdit));
};
```

**undo**

撤销操作，指针偏移栈顶位置+1。

```javascript
mxUndoManager.prototype.undo = function()
{
    while (this.indexOfNextAdd > 0)
    {
        var edit = this.history[--this.indexOfNextAdd];
        edit.undo();
		if (edit.isSignificant())
        {
        	this.fireEvent(new mxEventObject(mxEvent.UNDO, 'edit', edit));
            break;
        }
    }
};
```

**redo**

重做操作，指针偏移栈顶位置-1。

```javascript
mxUndoManager.prototype.redo = function()
{
    var n = this.history.length;
    while (this.indexOfNextAdd < n)
    {
        var edit =  this.history[this.indexOfNextAdd++];
        edit.redo();
        
        if (edit.isSignificant())
        {
        	this.fireEvent(new mxEventObject(mxEvent.REDO, 'edit', edit));
            break;
        }
    }
};
```



## 解码与持久化

`mxGraph` 支持导入（解码）与导出（持久化）xml。前面也提到相关的例子，下面重新摆出来：

```javascript
// 持久化
var encoder = new mxCodec();
var result = encoder.encode(graph.getModel());
var xml = mxUtils.getXml(result);

// 解码
var doc = mxUtils.parseXml(xmlString);
var codec = new mxCodec(doc);
codec.decode(doc.documentElement, graph.getModel());
```



### mxObjectCodec

通用对象编解码器，通过对象属性与节点属性的相互转换，实现了JavaScript对象与XML节点之间的相互映射。

#### 对象的映射

```javascript
var obj = new Object();
obj.foo = "Foo";
obj.bar = "Bar";

var enc = new mxCodec();
var node = enc.encode(obj);
// <Object foo="Foo" bar="Bar"/>
```

#### 数组的映射

```javascript
var obj = ["Bar", {bar: "Bar"}];
obj["bar"] = "Bar";
obj["foo"] = {bar: "Bar"};

/**
<Array bar="Bar">
  <add value="Bar"/>
  <Object bar="Bar"/>
  <Object bar="Bar" as="foo"/>
</Array>
*/
```

`as`是保留字，其值是父节点的key，所在节点是该key对应的值。

#### 相互引用

在JavaScript对象中，会出现相互引用的现象。譬如在一个`mxCell`实例中，为了易用性，会同时持有 `parent` 、 `children` 、 `edges` 、 `source` 、 `target`  等属性，如果直接解析，会陷入无限递归的死循环中。

`mxObjectCodec` 实例通过 `isExcluded` 方法判断该属性是否需要解析：

```javascript
// mxCellCodec:  ['children', 'edges', 'overlays', 'mxTransient'],
mxObjectCodec.prototype.isExcluded = function(obj, attr, value, write)
{
	return attr == mxObjectIdentity.FIELD_NAME ||
		mxUtils.indexOf(this.exclude, attr) >= 0;
};
```

通过`isReference` 方法判断是否需要转换该属性id值为对象:

```javascript
// mxCellCodec:  ['parent', 'source', 'target']
mxObjectCodec.prototype.isReference = function(obj, attr, value, write)
{
	return mxUtils.indexOf(this.idrefs, attr) >= 0;
};
```

下面看具体解析过程：

```javascript
mxObjectCodec.prototype.decodeAttribute = function(dec, attr, obj)
{
	if (!this.isIgnoredAttribute(dec, attr, obj))
	{
		var name = attr.nodeName;
		// Converts the string true and false to their boolean values.
		// This may require an additional check on the obj to see if
		// the existing field is a boolean value or uninitialized, in
		// which case we may want to convert true and false to a string.
		var value = this.convertAttributeFromXml(dec, attr, obj);
		var fieldname = this.getFieldName(name);
		if (this.isReference(obj, fieldname, value, false))
		{
			var tmp = dec.getObject(value);
			if (tmp == null)
			{
		    	mxLog.warn('mxObjectCodec.decode: No object for ' +
		    		this.getName() + '.' + name + '=' + value);
		    	return; // exit
		    }
		    value = tmp;
		}
		if (!this.isExcluded(obj, name, value, false))
		{
			//mxLog.debug(mxUtils.getFunctionName(obj.constructor)+'.'+name+'='+value);
			obj[name] = value;
		}
	}
};
```

`isExcluded` 返回false的时候，才会映射该值。`isReference` 返回true的时候，说明value是个引用的id，调用`getObject`查找该对象：

```javascript
mxCodec.prototype.getObject = function(id)
{
	var obj = null;
	if (id != null)
	{
		obj = this.objects[id];
		if (obj == null)
		{
			obj = this.lookup(id);
			if (obj == null)
			{
				var node = this.getElementById(id);	
				if (node != null)
				{
					obj = this.decode(node);
				}
			}
		}
	}
	return obj;
};
```

先从缓存`objects`中查找，如果没有，则通过id找到该节点，调用`this.decode(node);`解析得到。

#### 表达式

```xml
<Object>
  <add as="foo">mxConstants.ALIGN_LEFT</add>
</Object>
```

与下面等效：

```xml
<Object foo="left"/>
```

#### 各类Codec

`mxObjectCodec`是各类Codec的构造器函数，其实现了解编码器的核心解析逻辑。各子类则选择性复写了对应标签的解析功能。

```javascript
function mxObjectCodec(template, exclude, idrefs, mapping)
{
	this.template = template;
	
	this.exclude = (exclude != null) ? exclude : [];
	this.idrefs = (idrefs != null) ? idrefs : [];
	this.mapping = (mapping != null) ? mapping : [];
	
	this.reverse = new Object();
	for (var i in this.mapping)
	{
		this.reverse[this.mapping[i]] = i;
	}
};
```

**mxCellCodec**

```javascript
var codec = new mxObjectCodec(new mxCell(),
		['children', 'edges', 'overlays', 'mxTransient'],
		['parent', 'source', 'target']);

codec.afterEncode = function(enc, obj, node)
{
		// ...
		return node;
};
// ...
```

**mxStylesheetCodec**

```javascript
var codec = new mxObjectCodec(new mxStylesheet());

codec.encode = function(enc, obj)
{
  var node = enc.document.createElement(this.getName());
  // ...
  return node;
};
// ...
```

各个子类通过复写方法的方式，实现核心解析逻辑的复用。

### mxCodecRegistry

`mxCodecRegistry`是一个单例类，用于全局注册各类编解码器。

```javascript
register: function(codec)
	{
		if (codec != null)
		{
			var name = codec.getName();
			mxCodecRegistry.codecs[name] = codec;	
			var classname = mxUtils.getFunctionName(codec.template.constructor);
			if (classname != name)
			{
				mxCodecRegistry.addAlias(classname, name);
			}
		}
		return codec;
	},
```

其中注册的name为各个子类构造器名称：

```javascript
mxObjectCodec.prototype.getName = function()
{
	return mxUtils.getFunctionName(this.template.constructor);
};
```



### mxCodec

XML编解码器。通过此类使用注册在 `mxCodecRegistry` 上的各类`mxObjectCodec`的实例来解码节点或者编码对象。

#### parseXml

**`DOMParser`** 是原生DOM解析器，可以将 [XML](https://developer.mozilla.org/en-US/docs/Glossary/XML) 或 [HTML](https://developer.mozilla.org/en-US/docs/Glossary/HTML) 字符串解析为一个 DOM [`Document`](https://developer.mozilla.org/zh-CN/docs/Web/API/Document)。

```javascript
parseXml: function()
	{
		if (window.DOMParser)
		{
			return function(xml)
			{
				var parser = new DOMParser();
				
				return parser.parseFromString(xml, 'text/xml');
			};
		}
		else // IE<=9
		{
			return function(xml)
			{
				var doc = mxUtils.createMsXmlDocument();
				doc.loadXML(xml);
				
				return doc;
			};
		}
	}(),
```

#### mxCodec.prototype.decode

```javascript
mxCodec.prototype.decode = function(node, into)
{
	this.updateElements();
	var obj = null;
	if (node != null && node.nodeType == mxConstants.NODETYPE_ELEMENT)
	{
		var ctor = null;	
		try
		{
      // 通过标签名拿到全局构造器函数
			ctor = window[node.nodeName];
		}
		catch (err)
		{
			// ignore
		}
    // 通过构造器函数名称从全局注册中心拿到对应标签的codec实例
		var dec = mxCodecRegistry.getCodec(ctor);
		if (dec != null)
		{
      // 编解码器对node进行解码
			obj = dec.decode(this, node, into);
		}
		else
		{
			obj = node.cloneNode(true);
			obj.removeAttribute('as');
		}
	}
	return obj;
};
```

#### mxObjectCodec.prototype.decode

```javascript
mxObjectCodec.prototype.decode = function(dec, node, into)
{
	var id = node.getAttribute('id');
	var obj = dec.objects[id];
	// obj为对应标签构造器实例，没有缓存则创建一个
	if (obj == null)
	{
		obj = into || this.cloneTemplate();
		if (id != null)
		{
			dec.putObject(id, obj);
		}
	}
  // 解码前置处理接口
	node = this.beforeDecode(dec, node, obj);
  // 真正的解码逻辑
	this.decodeNode(dec, node, obj);
	// 解码后置处理接口，返回obj
  return this.afterDecode(dec, node, obj);
};	
```

#### mxObjectCodec.prototype.decodeNode

```javascript
mxObjectCodec.prototype.decodeNode = function(dec, node, obj)
{
	if (node != null)
	{
		this.decodeAttributes(dec, node, obj);
		this.decodeChildren(dec, node, obj);
	}
};
```

##### **decodeAttributes**

`decodeAttributes` 用于遍历解析节点的属性：

```javascript
mxObjectCodec.prototype.decodeAttributes = function(dec, node, obj)
{
	var attrs = node.attributes;
	if (attrs != null)
	{
		for (var i = 0; i < attrs.length; i++)
		{
			this.decodeAttribute(dec, attrs[i], obj);
		}
	}
};
```

**decodeAttribute**

前面分析互相引用小结时，已经贴过这段代码。 读取属性的值然后赋值到对应的obj中：

```javascript
mxObjectCodec.prototype.decodeAttribute = function(dec, attr, obj)
{
	if (!this.isIgnoredAttribute(dec, attr, obj))
	{
		var name = attr.nodeName;
		var value = this.convertAttributeFromXml(dec, attr, obj);
		var fieldname = this.getFieldName(name);
    // 如果是source，target，parent等引用字段，则value为id值，通过id查找对应的对象赋值给value。
		if (this.isReference(obj, fieldname, value, false))
		{
			var tmp = dec.getObject(value);
			if (tmp == null)
			{
		    	mxLog.warn('mxObjectCodec.decode: No object for ' +
		    		this.getName() + '.' + name + '=' + value);
		    	return; // exit
		    }
		    value = tmp;
		}
    // 如果不是'children', 'edges', 'overlays', 'mxTransient'等字段，则解析该值。
		if (!this.isExcluded(obj, name, value, false))
		{
			//mxLog.debug(mxUtils.getFunctionName(obj.constructor)+'.'+name+'='+value);
			obj[name] = value;
		}
	}
};
```

##### decodeChildren

遍历所有子节点，调用`decodeChild`：

```javascript
mxObjectCodec.prototype.decodeChildren = function(dec, node, obj)
{
	var child = node.firstChild;
	while (child != null)
	{
		var tmp = child.nextSibling;	
		if (child.nodeType == mxConstants.NODETYPE_ELEMENT &&
			!this.processInclude(dec, child, obj))
		{
			this.decodeChild(dec, child, obj);
		}
		child = tmp;
	}
};
```

##### **decodeChild**

以下面这段XML为例，探究解码过程：

```xml
<mxGraphModel>
  <root>
    <mxCell id="0"/>
    <mxCell edge="1" id="ff808081683c2d4e01683c41b87b15f0" parent="1"  value="华北.寨朔二线">
      <mxGeometry as="geometry" height="40.0" width="40.0">
        <Array as="points">
          <mxPoint x="13283.75" y="525.3828125"/>
          <mxPoint x="12381.8" y="599.3828125"/>
        </Array>
      </mxGeometry>
    </mxCell>
  </root>
</mxGraphModel>
```

###### **mxModelCodec#decodeChild**

先看特殊子节点的解码。当子节点是`root`节点时，model实例保存节点`model.setRoot(rootCell);`

```javascript
codec.decodeChild = function(dec, child, obj)
	{
		if (child.nodeName == 'root')
		{
			this.decodeRoot(dec, child, obj);
		}
		else
		{
			mxObjectCodec.prototype.decodeChild.apply(this, arguments);
		}
	};
```

并且通过`dec.decodeCell(tmp)` 解码root节点下所有的子节点，并通过 `insertIntoGraph`建立节点之间的关系：

```javascript
/**
	 * Function: decodeRoot
	 *
	 * Reads the cells into the graph model. All cells
	 * are children of the root element in the node.
	 */
codec.decodeRoot = function(dec, root, model)
{
  var rootCell = null;
  var tmp = root.firstChild;
  while (tmp != null)
  {
    var cell = dec.decodeCell(tmp);
    if (cell != null && cell.getParent() == null)
    {
      rootCell = cell;
    }
    tmp = tmp.nextSibling;
  }

  // Sets the root on the model if one has been decoded
  if (rootCell != null)
  {
    model.setRoot(rootCell);
  }
};
```

###### **insertIntoGraph**

```javascript
/**
 * Function: insertIntoGraph
 *
 * Inserts the given cell into its parent and terminal cells.
 */
mxCodec.prototype.insertIntoGraph = function(cell)
{
	var parent = cell.parent;
	var source = cell.getTerminal(true);
	var target = cell.getTerminal(false);

	// ...
			parent.insert(cell);
  // ...
};
```

###### model.setRoot(rootCell)

`setRoot` 发起一个事务，同步所有解码后的cell的数据到model实例中：

```javascript
mxGraphModel.prototype.setRoot = function(root)
{
	this.execute(new mxRootChange(this, root));
	return root;
};
```

**rootChanged**

```javascript
mxGraphModel.prototype.rootChanged = function(root)
{
	var oldRoot = this.root;
	this.root = root;
	
	// Resets counters and datastructures
	this.nextId = 0;
	this.cells = null;
	this.cellAdded(root);
	return oldRoot;
};
```

**cellAdded**

最终通过 `this.cells[cell.getId()] = cell;` 保存到model实例的`cells` 对象中：

```javascript
mxGraphModel.prototype.cellAdded = function(cell)
{
	if (cell != null)
	{
		// Creates an Id for the cell if not Id exists
		if (cell.getId() == null && this.createIds)
		{
			cell.setId(this.createId(cell));
		}
				// ...
				this.cells[cell.getId()] = cell;
		
		// ...
		
		for (var i=0; i<childCount; i++)
		{
			this.cellAdded(this.getChildAt(cell, i));
		}
	}
};
```



###### mxObjectCodec#decodeChild

接下来看常规子节点的解码：

```javascript
mxObjectCodec.prototype.decodeChild = function(dec, child, obj)
{
  /**
  <mxGeometry as="geometry" height="40.0" width="40.0">
    <Array as="points">
      <mxPoint x="13283.75" y="525.3828125"/>
      <mxPoint x="12381.8" y="599.3828125"/>
    </Array>
  </mxGeometry>
  */
  // 当子节点没有as属性时，意味着被包裹在一个数组中，反之则是一个对象的值
	var fieldname = this.getFieldName(child.getAttribute('as'));
	if (fieldname == null || !this.isExcluded(obj, fieldname, child, false))
	{
		var template = this.getFieldTemplate(obj, fieldname, child);
		var value = null;
		// 当子节点是 <add as="foo">mxConstants.ALIGN_LEFT</add>时
		if (child.nodeName == 'add')
		{
			value = child.getAttribute('value');
			
			if (value == null && mxObjectCodec.allowEval)
			{
				value = mxUtils.eval(mxUtils.getTextContent(child));
			}
		}
		else
		{
      // 递归解码节点
			value = dec.decode(child, template);
		}
		try
		{
      // 把子节点解码后的值赋值给父节点
			this.addObjectValue(obj, fieldname, value, template);
		}
		catch (e)
		{
			throw new Error(e.message + ' for ' + child.nodeName);
		}
	}
};
```

如注释所示，设置解码的子节点为父节点的值。如果父节点是个map对象，则设置为对应fieldname的值；如果父节点是数组，则添加到数组中：

```javascript
/**
 * Function: addObjectValue
 * 
 * Sets the decoded child node as a value of the given object. If the
 * object is a map, then the value is added with the given fieldname as a
 * key. If the fieldname is not empty, then setFieldValue is called or
 * else, if the object is a collection, the value is added to the
 * collection. For strongly typed languages it may be required to
 * override this with the correct code to add an entry to an object.
 */	
mxObjectCodec.prototype.addObjectValue = function(obj, fieldname, value, template)
{
	if (value != null && value != template)
	{
		if (fieldname != null && fieldname.length > 0)
		{
			obj[fieldname] = value;
		}
		else
		{
			obj.push(value);
		}
	}
};
```

到此为止，XML的解码流程结束。



## 图元注册与绘制

### mxCellRenderer**.**registerShape

前面已经提到过各类shape的注册，继续摆出来看一下：

```javascript
mxCellRenderer.registerShape = function(key, shape)
{
	mxCellRenderer.defaultShapes[key] = shape;
};
// Adds default shapes into the default shapes array
mxCellRenderer.registerShape(mxConstants.SHAPE_RECTANGLE, mxRectangleShape);
mxCellRenderer.registerShape(mxConstants.SHAPE_ELLIPSE, mxEllipse);
// ...
```

`defaultShapes` 是挂载在 `mxCellRenderer` 构造器上的静态map容器，注册维护了全局的所有shape构造器。

### mxStencilRegistry

shape不仅可以用函数来描述，还能用XML语言来进行描述。通过xml描述的shape是注册在`mxStencilRegistry` 构造器上的。

下面看一个真实的XML：

```xml
<shapes name="mxgraph.basic">
	<shape name="4 Point Star" h="92" w="92" aspect="variable" strokewidth="inherit">
    <connections>
      <constraint x="0.5" y="0" perimeter="0" name="N"/>
      <constraint x="0.5" y="1" perimeter="0" name="S"/>
      <constraint x="0" y="0.5" perimeter="0" name="W"/>
      <constraint x="1" y="0.5" perimeter="0" name="E"/>
    </connections>
    <background>
      <path>
        <move x="46" y="0"/>
        <line x="56" y="36"/>
        <line x="92" y="46"/>
        <line x="56" y="56"/>
        <line x="46" y="92"/>
        <line x="36" y="56"/>
        <line x="0" y="46"/>
        <line x="36" y="36"/>
        <close/>
      </path>
    </background>
    <foreground>
      <fillstroke/>
    </foreground>
  </shape>
  <shape>...</shape>
</shapes>
```

这里面有几个标签值得关注：

- shape：定义一个shape，shapes标签里面可包含多个shape。
- connections：定义相关连接点。
- background：负责图元的轮廓描边，只能包含`path`, `rect`, `roundrect` , `ellipse`等标签中的一个 ，`fill`、`fillstroke` 标签不能在`background`标签中使用。
- foreground：如果background不为空，`fill`、`fillstroke` 标签必须是`foreground`中第一个元素。

注册到`mxStencilRegistry`上：

```javascript
 var req = mxUtils.load('test/stencils.xml');
 var root = req.getDocumentElement();
 var shape = root.firstChild;
 while (shape != null)
 {
 	 if (shape.nodeType == mxConstants.NODETYPE_ELEMENT)
   {
     mxStencilRegistry.addStencil(shape.getAttribute('name'), new mxStencil(shape));
   }
   shape = shape.nextSibling;
 }
```

其中shape为dom节点。



### cellRenderer.redraw

前面在讲 `validate` 方法小节时，对于`state.invalid === true` 的节点，最后都会执行 `this.graph.cellRenderer.redraw(state, false, this.isRendering());` ，进行节点的重绘：

```javascript
mxCellRenderer.prototype.redraw = function(state, force, rendering)
{
	var shapeChanged = this.redrawShape(state, force, rendering);
	if (state.shape != null && (rendering == null || rendering))
	{
		this.redrawLabel(state, shapeChanged);
		this.redrawCellOverlays(state, shapeChanged);
		this.redrawControl(state, shapeChanged);
	}
};
```

### redrawShape

很显然，核心逻辑在`redrawShape`方法中：

```javascript
mxCellRenderer.prototype.redrawShape = function(state, force, rendering)
{
	var model = state.view.graph.model;
	var shapeChanged = false;
  
  // 判断state.shape实例是否需要重新创建
	// Forces creation of new shape if shape style has changed
	// ...
  {
  
		state.shape = this.createShape(state);
		
		if (state.shape != null)
		{
			state.shape.minSvgStrokeWidth = this.minSvgStrokeWidth;
			state.shape.antiAlias = this.antiAlias;
	
			this.createIndicatorShape(state);
			this.initializeShape(state);
			this.createCellOverlays(state);
			this.installListeners(state);
			
			// Forces a refresh of the handler if one exists
			state.view.graph.selectionCellsHandler.updateHandler(state);
		}
	}
	
  // ...
	
	// Updates indicator shape
	// ...

	if (state.shape != null)
	{
		// Handles changes of the collapse icon
		this.createControl(state);
		
		// Redraws the cell if required, ignores changes to bounds if points are
		// defined as the bounds are updated for the given points inside the shape
		if (force || this.isShapeInvalid(state, state.shape))
		{
			// ...
			
			if (rendering == null || rendering)
			{
				this.doRedrawShape(state);
			}
			else
			{
				state.shape.updateBoundingBox();
			}	
			shapeChanged = true;
		}
	}
	return shapeChanged;
};
```

#### createShape

优先通过`mxStencilRegistry` 查找注册的shape，然后通过`mxCellRenderer.defaultShapes` 查找：

```javascript
mxCellRenderer.prototype.createShape = function(state)
{
	var shape = null;
	if (state.style != null)
	{
		// Checks if there is a stencil for the name and creates
		// a shape instance for the stencil if one exists
		var stencil = mxStencilRegistry.getStencil(state.style[mxConstants.STYLE_SHAPE]);
		if (stencil != null)
		{
			shape = new mxShape(stencil);
		}
		else
		{
			var ctor = this.getShapeConstructor(state);
			shape = new ctor();
		}
	}
	return shape;
};
```

#### initializeShape

初始化节点，并挂载到容器`drawPane`上：

```javascript
mxCellRenderer.prototype.initializeShape = function(state)
{
	state.shape.dialect = state.view.graph.dialect;
	this.configureShape(state);
	state.shape.init(state.view.getDrawPane());
};
```

##### **init**

```javascript
mxShape.prototype.init = function(container)
{
	if (this.node == null)
	{
		this.node = this.create(container);
		if (container != null)
		{
			container.appendChild(this.node);
		}
	}
};
```

`create` 方法实际上是调用`createSvg`，创建一个`g` 容器标签：

```javascript
mxShape.prototype.createSvg = function()
{
	return document.createElementNS(mxConstants.NS_SVG, 'g');
};
```

#### doRedrawShape

到这一步，shape已经实例化，节点的容器也已经初始化，该进行真正的绘制环节了：

```javascript
mxCellRenderer.prototype.doRedrawShape = function(state)
{
	state.shape.redraw();
};
```

进入看`redraw`方法，此方法的作用就是创建绘制shape的所有svg节点：

```javascript
mxShape.prototype.redraw = function()
{
	this.updateBoundsFromPoints();
	if (this.visible && this.checkBounds())
	{
		// ...
			this.redrawShape();
  }
};
```

##### redrawShape

```javascript
mxShape.prototype.redrawShape = function()
{
  // 创建“类canvas”画布
	var canvas = this.createCanvas();
	if (canvas != null)
	{
		// ...
    // 真正绘制
		this.paint(canvas);
	
		// ...
		this.destroyCanvas(canvas);
	}
};
```

##### createCanvas

```javascript
mxShape.prototype.createCanvas = function()
{
	var canvas = null;
	
	// LATER: Check if reusing existing DOM nodes improves performance
	if (this.node.ownerSVGElement != null)
	{
		canvas = this.createSvgCanvas();
	}
	else if (mxClient.IS_VML)
	{
		this.updateVmlContainer();
		canvas = this.createVmlCanvas();
	}
	
	// ...

	return canvas;
};
```

**createSvgCanvas**

传入前面初始化的节点容器`this.node`，接收通过类canvas实例绘制生成的svg节点：

```javascript
mxShape.prototype.createSvgCanvas = function()
{
	var canvas = new mxSvgCanvas2D(this.node, false);
	canvas.strokeTolerance = (this.pointerEvents) ? this.svgStrokeTolerance : 0;
	canvas.pointerEventsValue = this.svgPointerEvents;
	
  // ...
	
	return canvas;
};
```

##### paint

“类canvas”画布实例已经初始化，是时候执行真正的绘制逻辑了：

```javascript
mxShape.prototype.paint = function(c)
{
	// ...
	
  // 设置transform属性
	this.updateTransform(c, x, y, w, h);
  // 配置样式等属性
	this.configureCanvas(c, x, y, w, h);

	// Adds background rectangle to capture events
	// ...
  
  // 如果shape是XML描述的
	if (this.stencil != null)
	{
		this.stencil.drawShape(c, this, x, y, w, h);
	}
	else
	{
		// Stencils have separate strokewidth
		c.setStrokeWidth(this.strokewidth);
		
		if (this.points != null)
		{
			// Paints edge shape
			// ...
			this.paintEdgeShape(c, pts);
		}
		else
		{
			// Paints vertex shape
			this.paintVertexShape(c, x, y, w, h);
		}
	}
	
  // ...
	
	// Draws highlight rectangle if no stroke was used
	if (c != null && this.outline && !strokeDrawn)
	{
		c.rect(x, y, w, h);
		c.stroke();
	}
};
```

`paintEdgeShape` 跟 `paintVertexShape`分别是用来绘制edge、vertex类的shape，需要各个具体的shape类自行实现绘制逻辑。

当shape是XML描述的，则走`this.stencil.drawShape(c, this, x, y, w, h);`，通过读取解析XML标签的绘制逻辑进行绘制：

```javascript
mxStencil.prototype.drawShape = function(canvas, shape, x, y, w, h)
{
	// ...

	this.drawChildren(canvas, shape, x, y, w, h, this.bgNode, aspect, false, true);
	this.drawChildren(canvas, shape, x, y, w, h, this.fgNode, aspect, true,
		!shape.outline || shape.style == null || mxUtils.getValue(
		shape.style, mxConstants.STYLE_BACKGROUND_OUTLINE, 0) == 0);
	
	// ...
};
```

到此为止，图元的注册与绘制流程结束。



## handler机制

`mxGraph` 的handler机制是通过基于事件的编程模型来实现，通过事件注册与移除，实现特定功能的可插拔。

### Listener

`mouseListeners` 中持有的listener必须实现`mouseDown`、`mouseMove`、`mouseUp` 三个方法。

```javascript
graph.addMouseListener(
 {
   mouseDown: function(sender, evt)
   {
     mxLog.debug('mouseDown');
   },
   mouseMove: function(sender, evt)
   {
     mxLog.debug('mouseMove');
   },
   mouseUp: function(sender, evt)
   {
     mxLog.debug('mouseUp');
   }
 });
```

### addMouseListener

所有的listener保存在一个数组中：

```javascript
mxGraph.prototype.addMouseListener = function(listener)
{
	if (this.mouseListeners == null)
	{
		this.mouseListeners = [];
	}	
	this.mouseListeners.push(listener);
};
```

### removeMouseListener

移除listener：

```javascript
mxGraph.prototype.removeMouseListener = function(listener)
{
	if (this.mouseListeners != null)
	{
		for (var i = 0; i < this.mouseListeners.length; i++)
		{
			if (this.mouseListeners[i] == listener)
			{
				this.mouseListeners.splice(i, 1);
				break;
			}
		}
	}
};
```

### fireMouseEvent

当事件触发时，遍历所有的listener，执行相应的事件：

```javascript
mxGraph.prototype.fireMouseEvent = function(evtName, me, sender)
{
  // ...
  
  if (this.mouseListeners != null)
  {
    var args = [sender, me];
    // Does not change returnValue in Opera
    if (!me.getEvent().preventDefault)
    {
      me.getEvent().returnValue = true;
    }
    for (var i = 0; i < this.mouseListeners.length; i++)
    {
      var l = this.mouseListeners[i];

      if (evtName == mxEvent.MOUSE_DOWN)
      {
        l.mouseDown.apply(l, args);
      }
      else if (evtName == mxEvent.MOUSE_MOVE)
      {
        l.mouseMove.apply(l, args);
      }
      else if (evtName == mxEvent.MOUSE_UP)
      {
        l.mouseUp.apply(l, args);
      }
    }
  }
  
  // ... 
}
```

### installListeners

事件的触发流程已经很清晰了，现在看一下默认监听的listener有哪些：

#### redrawShape

节点重绘时，绑定事件：

```javascript
mxCellRenderer.prototype.redrawShape = function(state, force, rendering)
{
   // ...
  
		if (state.shape != null)
		{
			// ...
			this.installListeners(state);
			
		}
	}
  // ...
	return shapeChanged;
};
```

在每个`node`上绑定事件，间接出发graph上的`mouseListeners`：

```javascript
mxCellRenderer.prototype.installListeners = function(state)
{
	// ...
	mxEvent.addGestureListeners(state.shape.node,
		mxUtils.bind(this, function(evt)
		{
			if (this.isShapeEvent(state, evt))
			{
				graph.fireMouseEvent(mxEvent.MOUSE_DOWN, new mxMouseEvent(evt, state));
			}
		}),
		mxUtils.bind(this, function(evt)
		{
			if (this.isShapeEvent(state, evt))
			{
				graph.fireMouseEvent(mxEvent.MOUSE_MOVE, new mxMouseEvent(evt, getState(evt)));
			}
		}),
		mxUtils.bind(this, function(evt)
		{
			if (this.isShapeEvent(state, evt))
			{
				graph.fireMouseEvent(mxEvent.MOUSE_UP, new mxMouseEvent(evt, getState(evt)));
			}
		})
	);
	
	// Uses double click timeout in mxGraph for quirks mode
	// ...
};
```

#### mxGraphView.prototype.init

`view` 实例初始化时，监听事件：

```javascript
mxGraphView.prototype.init = function()
{
	this.installListeners();
	
	// Creates the DOM nodes for the respective display dialect
	// ...
};
```

在`container`上绑定事件：

```javascript
mxGraphView.prototype.installListeners = function()
{
	var graph = this.graph;
	var container = graph.container;
  if(container != null) {
    // ...
    
  	// Adds basic listeners for graph event dispatching
		mxEvent.addGestureListeners(container, mxUtils.bind(this, function(evt)
		{
			// Condition to avoid scrollbar events starting a rubberband selection
			if (this.isContainerEvent(evt) && ((!mxClient.IS_IE && !mxClient.IS_IE11 && !mxClient.IS_GC &&
				!mxClient.IS_OP && !mxClient.IS_SF) || !this.isScrollEvent(evt)))
			{
				graph.fireMouseEvent(mxEvent.MOUSE_DOWN, new mxMouseEvent(evt));
			}
		}),
		mxUtils.bind(this, function(evt)
		{
			if (this.isContainerEvent(evt))
			{
				graph.fireMouseEvent(mxEvent.MOUSE_MOVE, new mxMouseEvent(evt));
			}
		}),
		mxUtils.bind(this, function(evt)
		{
			if (this.isContainerEvent(evt))
			{
				graph.fireMouseEvent(mxEvent.MOUSE_UP, new mxMouseEvent(evt));
			}
		}));
    
    // ...
  }
}
```

### mxTooltipHandler

下面来看一个简单的官方handler实例，从而学习如何自定义一个handler。

`mxTooltipHandler` 是内置的handler，实现提示节点信息的功能：

```javascript
function mxTooltipHandler(graph, delay)
{
	if (graph != null)
	{
		this.graph = graph;
		this.delay = delay || 500;
		this.graph.addMouseListener(this);
	}
};
```

通过`this.graph.addMouseListener(this);` 安装listener，必然有相对应的`mouseDown`、`mouseMove`、`mouseUp` 三个方法：

```javascript
mxTooltipHandler.prototype.mouseDown = function(sender, me)
{
	this.reset(me, false);
	this.hideTooltip();
};

/**
 * Function: mouseMove
 * 
 * Handles the event by updating the rubberband selection.
 */
mxTooltipHandler.prototype.mouseMove = function(sender, me)
{
	if (me.getX() != this.lastX || me.getY() != this.lastY)
	{
		this.reset(me, true);
		var state = this.getStateForEvent(me);
		
		if (this.isHideOnHover() || state != this.state || (me.getSource() != this.node &&
			(!this.stateSource || (state != null && this.stateSource ==
			(me.isSource(state.shape) || !me.isSource(state.text))))))
		{
			this.hideTooltip();
		}
	}
	
	this.lastX = me.getX();
	this.lastY = me.getY();
};

/**
 * Function: mouseUp
 * 
 * Handles the event by resetting the tooltip timer or hiding the existing
 * tooltip.
 */
mxTooltipHandler.prototype.mouseUp = function(sender, me)
{
	this.reset(me, true);
	this.hideTooltip();
};
```

这个handler相对简单易懂，官方还有提供了`mxConnectionHandler` (连接)、`mxRubberband` (框选)、`mxSelectionCellsHandler` (选择)等handler，可以自由选择相关功能等组合，当然，也可以自定义实现handler。

