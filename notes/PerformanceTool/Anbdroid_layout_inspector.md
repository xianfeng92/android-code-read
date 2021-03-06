# Debug Your layout with Layout Inspector

Layout Inspector 是 Android studio 中自带的视图层次结构分析工具,其可以在运行时从 Android Studio IDE 中检查应用程序的视图层次结构。


## Open the Layout Inspector

按以下步骤操作：

1. 在连接的设备或模拟器上运行应用

2. 点击 Tools > Android > Layout Inspector

3. 在出现的 Choose Process 对话框中，选择您想要检查的应用进程，然后点击 OK。


默认情况下，Choose Process 对话框只会为 Android Studio 中当前打开的项目列出进程，并且该项目必须是在设备上运行。点击 Show all processes 可以检查设备上的
其他应用。对于已取得 root 权限的设备或者没有安装 Google Play 商店的模拟器，我们可以看到所有正在运行的应用。否则，只能看到可以调试的运行中应用。

![The Choose Process dialog]()

布局检查器会捕获快照，将它保存为 .li 文件。如下图所示，布局检查器将显示以下内容：

![layout-inspector-callouts_2x]()

1. View Tree：视图在布局中的层次结构

2. Layout Inspector toolbar:布局检查器工具栏

3. Screenshot：带有可视边界( layout bounds)的设备屏幕截图

4. Properties Table：选定视图的布局属性


## Select a view

只有在 View Tree 或者 screenshot 点击一下,即可选中对应的视图,Properties Table 中将会显示选中视图的所有布局属性。如果我们的布局中包括重叠视图(overlapping views)，
则默认情况下，只有最前面的视图可以在屏幕截图中点击。想要选择不在最前面的视图，可以在 View Tree 中单击该视图。


## Isolate a view

在复杂的布局中，可以隔离单个视图，以便只在 View Tree 中显示布局的子集并在屏幕截图中呈现。只能在设备仍处于连接状态时隔离视图。隔离视图需要设备呈现布局，以便布局检查器
可以进行另一个屏幕截图。

要隔离视图，请执行以下操作之一：

* 双击屏幕截图中的视图

* 在 View Tree 中右键单击视图，然后选择“Render Subtree Preview”


## Hide layout bounds

要隐藏布局元素的边界框，右键单击 View Tree 中的元素，然后取消选择“ Show layout bounds”


## Zoom in and use a reference grid to inspect layout details

使用 Layout Editor toolbar 中的按钮可以控制屏幕截图的网格覆盖和缩放级别：

* 点击 Zoom In, 即可放大屏幕截图

* 点击 zoom out,即可缩小屏幕截图

* 点击 Actual Size , 即可以屏幕截图中的一个像素(px)对应设备上的一个像素的放大率查看布局

* 点击 Grid, 要覆盖像素网格. 注意:Grid仅在高缩放级别可用

## Compare app layout to a reference image overlay

To compare your app layout with a reference image, such as a UI mockup, you can load a bitmap image overlay in the Layout Inspector.

* To load an overlay, click Load Overlay  at the top of the Layout Inspector. The overlay is scaled to fit the layout.

* To adjust the transparency of the overlay, use the Alpha slider.

* To remove the overlay, click Clear Overlay .

## Take a new screenshot to capture layout changes

如果设备上的布局发生更改，则布局检查器不会自动更新。 要捕获布局更改，请再次单击 Tools > Layout Inspector 以创建新的屏幕截图。





[layout-inspector](https://developer.android.com/studio/debug/layout-inspector.html)

