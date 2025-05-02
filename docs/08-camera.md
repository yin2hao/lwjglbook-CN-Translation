# 第08章 摄像机（Camera）

在本章中，我们将学习如何在渲染的3D场景中移动。这个功能就像拥有一个可以在3D世界中移动的摄像机，实际上这就是用来指代它的术语。

你可以在[这里](https://github.com/lwjglgamedev/lwjglbook/tree/main/chapter-08)找到本章的完整源代码。

## 摄像机简介（Camera introduction）

如果你尝试在OpenGL中搜索特定的摄像机函数，你会发现并没有摄像机的概念，或者说摄像机始终是固定的，位于屏幕中心的(0, 0, 0)位置。因此，我们将通过模拟来实现一个能够在3D场景中移动的摄像机。我们如何实现这一点呢？既然我们无法移动摄像机，那么我们必须同时移动3D空间中的所有对象。换句话说，如果我们不能移动摄像机，我们就移动整个世界。

假设我们希望将摄像机位置沿z轴从初始位置(Cx, Cy, Cz)移动到位置(Cx, Cy, Cz+dz)，以更接近位于坐标(Ox, Oy, Oz)的对象。

![摄像机移动](_static/08/camera_movement.png)

实际上，我们将做的是将对象（实际上是3D空间中的所有对象）向摄像机应该移动的相反方向移动。可以想象这些对象被放在跑步机上。

![实际移动](_static/08/actual_movement.png)

摄像机可以沿三个轴（x、y和z）移动，也可以绕它们旋转（滚动、俯仰和偏航）。

![滚动、俯仰和偏航](_static/08/roll_pitch_yaw.png)

因此，基本上我们必须能够移动和旋转3D世界中的所有对象。我们将如何做到这一点？答案是通过应用另一个变换，将所有对象的所有顶点向摄像机移动的相反方向平移，并根据摄像机的旋转进行旋转。当然，这将通过另一个矩阵来完成，即所谓的**视图矩阵**（View Matrix）。这个矩阵将首先执行平移，然后沿轴旋转。

让我们看看如何构造这个矩阵。如果你还记得变换章节，我们的变换方程是这样的：

$$
\begin{array}{lcl}
Transf & = & \lbrack ProjMatrix \rbrack \cdot \lbrack TranslationMatrix \rbrack \cdot \lbrack  RotationMatrix \rbrack \cdot \lbrack  ScaleMatrix \rbrack \\ 
 & = & \lbrack   ProjMatrix \rbrack  \cdot \lbrack  WorldMatrix \rbrack
\end{array}
$$

视图矩阵应该在乘以投影矩阵之前应用，因此我们的方程现在应该是这样的：

$$
\begin{array}{lcl}
Transf & = & \lbrack  ProjMatrix \rbrack \cdot \lbrack  ViewMatrix \rbrack \cdot \lbrack  TranslationMatrix \rbrack \cdot \lbrack  RotationMatrix \rbrack \cdot \lbrack ScaleMatrix \rbrack \\
  & = & \lbrack ProjMatrix \rbrack \cdot \lbrack  ViewMatrix \rbrack \cdot \lbrack  WorldMatrix \rbrack 
\end{array}
$$

## 摄像机实现（Camera implementation）

让我们开始修改代码以支持摄像机。首先，我们将创建一个名为`Camera`的新类，它将保存摄像机的位置和旋转状态以及其视图矩阵。该类的定义如下：

```java
package org.lwjglb.engine.scene;

import org.joml.*;

public class Camera {

    private Vector3f direction;
    private Vector3f position;
    private Vector3f right;
    private Vector2f rotation;
    private Vector3f up;
    private Matrix4f viewMatrix;

    public Camera() {
        direction = new Vector3f();
        right = new Vector3f();
        up = new Vector3f();
        position = new Vector3f();
        viewMatrix = new Matrix4f();
        rotation = new Vector2f();
    }

    public void addRotation(float x, float y) {
        rotation.add(x, y);
        recalculate();
    }

    public Vector3f getPosition() {
        return position;
    }

    public Matrix4f getViewMatrix() {
        return viewMatrix;
    }

    public void moveBackwards(float inc) {
        viewMatrix.positiveZ(direction).negate().mul(inc);
        position.sub(direction);
        recalculate();
    }

    public void moveDown(float inc) {
        viewMatrix.positiveY(up).mul(inc);
        position.sub(up);
        recalculate();
    }

    public void moveForward(float inc) {
        viewMatrix.positiveZ(direction).negate().mul(inc);
        position.add(direction);
        recalculate();
    }

    public void moveLeft(float inc) {
        viewMatrix.positiveX(right).mul(inc);
        position.sub(right);
        recalculate();
    }

    public void moveRight(float inc) {
        viewMatrix.positiveX(right).mul(inc);
        position.add(right);
        recalculate();
    }

    public void moveUp(float inc) {
        viewMatrix.positiveY(up).mul(inc);
        position.add(up);
        recalculate();
    }

    private void recalculate() {
        viewMatrix.identity()
                .rotateX(rotation.x)
                .rotateY(rotation.y)
                .translate(-position.x, -position.y, -position.z);
    }

    public void setPosition(float x, float y, float z) {
        position.set(x, y, z);
        recalculate();
    }

    public void setRotation(float x, float y) {
        rotation.set(x, y);
        recalculate();
    }
}
```

如你所见，除了旋转和位置外，我们还定义了一些向量来定义前进、上和右方向。这是因为我们正在实现一个自由空间移动摄像机，当我们旋转它时，如果我们想向前移动，我们只想移动到摄像机指向的位置，而不是预定义的轴。我们需要获取这些向量来计算下一个位置将放置在哪里。最后，摄像机的状态存储在一个4x4矩阵中，即视图矩阵，因此每当我们更改位置或旋转时都需要更新它。如你所见，在更新视图矩阵时，我们需要先进行旋转，然后进行平移。如果我们反过来做，我们将不会沿着摄像机位置旋转，而是沿着坐标原点旋转。

`Camera`类还提供了在向前、向上或向右移动时更新位置的方法。在这些方法中，视图矩阵用于根据当前状态计算前进、上或右方法应该在哪里，并相应地增加位置。我们使用出色的JOML库来为我们进行这些计算，同时保持代码非常简单。

## 使用摄像机（Using the Camera）

我们将在`Scene`类中存储一个`Camera`实例，因此让我们进行更改：

```java
public class Scene {
    ...
    private Camera camera;
    ...
    public Scene(int width, int height) {
        ...
        camera = new Camera();
    }
    ...
    public Camera getCamera() {
        return camera;
    }
    ...
}
```

用鼠标控制摄像机会很好。为此，我们将创建一个新类来处理鼠标事件，以便我们可以使用它们来更新摄像机旋转。以下是该类的代码。

```java
package org.lwjglb.engine;

import org.joml.Vector2f;

import static org.lwjgl.glfw.GLFW.*;

public class MouseInput {

    private Vector2f currentPos;
    private Vector2f displVec;
    private boolean inWindow;
    private boolean leftButtonPressed;
    private Vector2f previousPos;
    private boolean rightButtonPressed;

    public MouseInput(long windowHandle) {
        previousPos = new Vector2f(-1, -1);
        currentPos = new Vector2f();
        displVec = new Vector2f();
        leftButtonPressed = false;
        rightButtonPressed = false;
        inWindow = false;

        glfwSetCursorPosCallback(windowHandle, (handle, xpos, ypos) -> {
            currentPos.x = (float) xpos;
            currentPos.y = (float) ypos;
        });
        glfwSetCursorEnterCallback(windowHandle, (handle, entered) -> inWindow = entered);
        glfwSetMouseButtonCallback(windowHandle, (handle, button, action, mode) -> {
            leftButtonPressed = button == GLFW_MOUSE_BUTTON_1 && action == GLFW_PRESS;
            rightButtonPressed = button == GLFW_MOUSE_BUTTON_2 && action == GLFW_PRESS;
        });
    }

    public Vector2f getCurrentPos() {
        return currentPos;
    }

    public Vector2f getDisplVec() {
        return displVec;
    }

    public void input() {
        displVec.x = 0;
        displVec.y = 0;
        if (previousPos.x > 0 && previousPos.y > 0 && inWindow) {
            double deltax = currentPos.x - previousPos.x;
            double deltay = currentPos.y - previousPos.y;
            boolean rotateX = deltax != 0;
            boolean rotateY = deltay != 0;
            if (rotateX) {
                displVec.y = (float) deltax;
            }
            if (rotateY) {
                displVec.x = (float) deltay;
            }
        }
        previousPos.x = currentPos.x;
        previousPos.y = currentPos.y;
    }

    public boolean isLeftButtonPressed() {
        return leftButtonPressed;
    }

    public boolean isRightButtonPressed() {
        return rightButtonPressed;
    }
}
```

`MouseInput`类在其构造函数中注册了一组回调来处理鼠标事件：

* `glfwSetCursorPosCallback`：注册一个回调，当鼠标移动时调用。
* `glfwSetCursorEnterCallback`：注册一个回调，当鼠标进入我们的窗口时调用。即使鼠标不在我们的窗口中，我们也会收到鼠标事件。我们使用此回调来跟踪鼠标何时在我们的窗口中。
* `glfwSetMouseButtonCallback`：注册一个回调，当鼠标按钮被按下时调用。

`MouseInput`类提供了一个`input`方法，应在处理游戏输入时调用。此方法计算鼠标从先前位置的位移并将其存储在`displVec`变量中，以便我们的游戏使用。

`MouseInput`类将在我们的`Window`类中实例化，该类还将提供一个getter来返回其实例。

```java
public class Window {
    ...
    private MouseInput mouseInput;
    ...
    public Window(String title, WindowOptions opts, Callable<Void> resizeFunc) {
        ...
        mouseInput = new MouseInput(windowHandle);
    }
    ...
    public MouseInput getMouseInput() {
        return mouseInput;
    }
    ...    
}
```

在`Engine`类中，我们将在处理常规输入时使用鼠标输入：
```java
public class Engine {
    ...
    private void run() {
        ...
            if (targetFps <= 0 || deltaFps >= 1) {
                window.getMouseInput().input();
                appLogic.input(window, scene, now - initialTime);
            }
        ...
    }
    ...
}
```

现在我们可以修改顶点着色器以使用摄像机的视图矩阵，正如你可能猜到的，它将作为统一变量传递。

```glsl
#version 330

layout (location=0) in vec3 position;
layout (location=1) in vec2 texCoord;

out vec2 outTextCoord;

uniform mat4 projectionMatrix;
uniform mat4 viewMatrix;
uniform mat4 modelMatrix;

void main()
{
    gl_Position = projectionMatrix * viewMatrix * modelMatrix * vec4(position, 1.0);
    outTextCoord = texCoord;
}
```

因此，下一步是在`SceneRender`类中正确创建统一变量，并在每次`render`调用中更新其值：

```java
public class SceneRender {
    ...
    private void createUniforms() {
        ...
        uniformsMap.createUniform("viewMatrix");
        ...
    }    
    ...
    public void render(Scene scene) {
        ...
        uniformsMap.setUniform("projectionMatrix", scene.getProjection().getProjMatrix());
        uniformsMap.setUniform("viewMatrix", scene.getCamera().getViewMatrix());
        ...
    }
}
```

就是这样，我们的基础代码支持摄像机的概念。现在我们需要使用它。我们可以更改处理输入和更新摄像机的方式。我们将设置以下控制：

* 按键“A”和“D”分别将摄像机向左和向右（x轴）移动。
* 按键“W”和“S”分别将摄像机向前和向后（z轴）移动。
* 按键“Z”和“X”分别将摄像机向上和向下（y轴）移动。

当鼠标右键按下时，我们将使用鼠标位置绕x和y轴旋转摄像机。

现在我们准备更新我们的`Main`类来处理键盘和鼠标输入。

```java

public class Main implements IAppLogic {

    private static final float MOUSE_SENSITIVITY = 0.1f;
    private static final float MOVEMENT_SPEED = 0.005f;
    ...

    public static void main(String[] args) {
        ...
        Engine gameEng = new Engine("chapter-08", new Window.WindowOptions(), main);
        ...
    }
    ...
    public void input(Window window, Scene scene, long diffTimeMillis) {
        float move = diffTimeMillis * MOVEMENT_SPEED;
        Camera camera = scene.getCamera();
        if (window.isKeyPressed(GLFW_KEY_W)) {
            camera.moveForward(move);
        } else if (window.isKeyPressed(GLFW_KEY_S)) {
            camera.moveBackwards(move);
        }
        if (window.isKeyPressed(GLFW_KEY_A)) {
            camera.moveLeft(move);
        } else if (window.isKeyPressed(GLFW_KEY_D)) {
            camera.moveRight(move);
        }
        if (window.isKeyPressed(GLFW_KEY_UP)) {
            camera.moveUp(move);
        } else if (window.isKeyPressed(GLFW_KEY_DOWN)) {
            camera.moveDown(move);
        }

        MouseInput mouseInput = window.getMouseInput();
        if (mouseInput.isRightButtonPressed()) {
            Vector2f displVec = mouseInput.getDisplVec();
            camera.addRotation((float) Math.toRadians(-displVec.x * MOUSE_SENSITIVITY),
                    (float) Math.toRadians(-displVec.y * MOUSE_SENSITIVITY));
        }
    }
    ...
}
```

[下一章](./09-loading-more-complex-models.md)