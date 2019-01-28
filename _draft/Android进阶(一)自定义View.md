## 1 坐标系
Android系统里面有两种坐标系：Android坐标系、View坐标系。
### 1.1 Android坐标系
![](https://github.com/KinsomyJS/KinsomyJS.github.io/blob/master/img/view/1.jpg?raw=true)
Android的坐标系是以手机上可见的屏幕左上角顶点为坐标系原点，但是xy轴的方向和我们以前知道的有所不同，需要注意，从原点向右为x轴正方向，而从原点向下为y轴正方向。

`android.view.MotionEvent`下面有两个方法`getRawX()`和`getRawY()`可以获得当前触摸位置的Android坐标系坐标。

### View坐标系
可以说Android坐标系是屏幕绝对坐标，View坐标系是viewgroup的相对坐标，这两者可以结合使用，做到更加精确的控制。
![](https://github.com/KinsomyJS/KinsomyJS.github.io/blob/master/img/view/2.png?raw=true)
| 方法 | 从属 | 含义 |
| ------ | ------ | ------ |
| getX() | MotionEvent | 触摸位置距离控件左边缘距离  |
| getY() | MotionEvent | 触摸位置距离控件上边缘距离 |
| getTop() | View | 当前view到其父布局上边缘距离 |
| getBottom() | View | 当前view下边缘到其父布局上边缘距离 |
| getLeft() | View | 当前view左边缘到其父布局左边缘距离 |
| getRight() | View | 当前view右边缘到其父布局左边缘边缘距离 |