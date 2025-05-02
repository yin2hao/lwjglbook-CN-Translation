# 第02章 - 游戏循环（The Game Loop）

在本章中，我们将通过创建游戏循环开始开发我们的游戏引擎。游戏循环是每个游戏的核心组件。它基本上是一个无尽的循环，负责周期性地处理用户输入、更新游戏状态并渲染到屏幕上。

你可以在[这里](https://github.com/lwjglgamedev/lwjglbook/tree/main/chapter-02)找到本章的完整源代码。

## 基础（The basis）

以下代码片段展示了游戏循环的结构：

```java
while (keepOnRunning) {
    input();
    update();
    render();
}
```

`input`方法负责处理用户输入（按键、鼠标移动等）。`update`方法负责更新游戏状态（敌人位置、人工智能等），最后是`render`方法。那么，这就全部结束了吗？我们的游戏循环完成了吗？还没有。上述代码片段存在许多问题。首先，游戏循环运行的速度将根据运行的机器不同而不同。如果机器足够快，用户甚至无法看清楚游戏中发生了什么。而且，这样的游戏循环将耗尽所有的机器资源。

首先，我们可能希望分别控制游戏状态更新和渲染到屏幕的周期。为什么要这样做？以恒定速率更新游戏状态更重要，尤其是如果我们使用了某种物理引擎。相反，如果渲染未能及时完成，处理游戏循环时渲染旧帧就毫无意义了。我们可以灵活地跳过某些帧。

## 实现（Implementation）

在检查游戏循环之前，让我们先创建一些支持类，这些类将构成引擎的核心。我们首先将创建一个接口来封装游戏逻辑。通过这样做，我们可以使我们的游戏引擎在不同章节之间可重用。该接口将包含初始化游戏资源（`init`）、处理用户输入（`input`）、更新游戏状态（`update`）和清理资源（`cleanup`）的方法。

```java
package org.lwjglb.engine;

import org.lwjglb.engine.graph.Render;
import org.lwjglb.engine.scene.Scene;

public interface IAppLogic {

    void cleanup();

    void init(Window window, Scene scene, Render render);

    void input(Window window, Scene scene, long diffTimeMillis);

    void update(Window window, Scene scene, long diffTimeMillis);
}
```

如你所见，有一些实例类尚未定义（`Window`、`Scene`和`Render`），以及一个名为`diffTimeMillis`的参数，它保存了这些方法调用之间经过的毫秒数。

让我们从`Window`类开始。我们将在此类中封装所有对**GLFW**库的调用，以创建和管理窗口，其结构如下：

```java
package org.lwjglb.engine;

import org.lwjgl.glfw.GLFWVidMode;
import org.lwjgl.system.MemoryUtil;
import org.tinylog.Logger;

import java.util.concurrent.Callable;

import static org.lwjgl.glfw.Callbacks.glfwFreeCallbacks;
import static org.lwjgl.glfw.GLFW.*;
import static org.lwjgl.opengl.GL11.*;
import static org.lwjgl.system.MemoryUtil.NULL;

public class Window {

    private final long windowHandle;
    private int height;
    private Callable<Void> resizeFunc;
    private int width;
    ...
    ...
    public static class WindowOptions {
        public boolean compatibleProfile;
        public int fps;
        public int height;
        public int ups = Engine.TARGET_UPS;
        public int width;
    }
}
```

如你所见，它定义了一些属性来存储窗口句柄、宽度和高度，以及在窗口大小改变时调用的回调函数。它还定义了一个内部类来设置一些用于控制窗口创建的选项：

* `compatibleProfile`：这控制我们是否希望使用旧版本中的函数（已弃用的函数）。
* `fps`：定义目标帧率（FPS）。如果其值小于或等于零，则意味着我们不设置目标帧率，而是使用显示器的刷新率作为目标帧率。为了实现这一点，我们将使用垂直同步，也就是从调用`glfwSwapBuffers`起等待的屏幕刷新次数。
* `height`：期望的窗口高度。
* `width`：期望的窗口宽度。
* `ups`：定义每秒更新次数的目标值（初始化为默认值）。

让我们来看一下`Window`类的构造方法：

```java
public class Window {
    ...
    public Window(String title, WindowOptions opts, Callable<Void> resizeFunc) {
        this.resizeFunc = resizeFunc;
        if (!glfwInit()) {
            throw new IllegalStateException("Unable to initialize GLFW");
        }

        glfwDefaultWindowHints();
        glfwWindowHint(GLFW_VISIBLE, GL_FALSE);
        glfwWindowHint(GLFW_RESIZABLE, GL_TRUE);

        glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 3);
        glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 2);

        //这个不重要
        if (opts.compatibleProfile) {
            glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_COMPAT_PROFILE);//兼容模式
        } else {
            glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE);//核心模式
            glfwWindowHint(GLFW_OPENGL_FORWARD_COMPAT, GL_TRUE);
        }

        if (opts.width > 0 && opts.height > 0) {
            this.width = opts.width;
            this.height = opts.height;
        } else {
            glfwWindowHint(GLFW_MAXIMIZED, GLFW_TRUE);
            GLFWVidMode vidMode = glfwGetVideoMode(glfwGetPrimaryMonitor());
            width = vidMode.width();
            height = vidMode.height();
        }

        windowHandle = glfwCreateWindow(width, height, title, NULL, NULL);
        if (windowHandle == NULL) {
            throw new RuntimeException("Failed to create the GLFW window");
        }

        //注册一个回调函数，当窗口的​帧缓冲区大小​（实际渲染区域）发生变化时（如窗口缩放、全屏切换），自动调用该函数
        glfwSetFramebufferSizeCallback(windowHandle, (window, w, h) -> resized(w, h));

        // ​​GLFW 错误回调的设置​​，用于捕获并处理 GLFW 库运行时的错误信息
        //​​必须在 glfwInit() 之前设置​​，否则早期错误（如库加载失败）无法被捕获
        glfwSetErrorCallback((int errorCode, long msgPtr) ->
                Logger.error("Error code [{}], msg [{}]", errorCode, MemoryUtil.memUTF8(msgPtr))
        );

        //按键回调，此处监听ESC按键
        glfwSetKeyCallback(windowHandle, (window, key, scancode, action, mods) -> {
            keyCallBack(key, action);
        });

        //绑定上下文
        glfwMakeContextCurrent(windowHandle);

        if (opts.fps > 0) {
            glfwSwapInterval(0);
        } else {
            glfwSwapInterval(1);
        }

        glfwShowWindow(windowHandle);

        int[] arrWidth = new int[1];
        int[] arrHeight = new int[1];
        glfwGetFramebufferSize(windowHandle, arrWidth, arrHeight);
        width = arrWidth[0];
        height = arrHeight[0];
    }
    ...
    public void keyCallBack(int key, int action) {
        if (key == GLFW_KEY_ESCAPE && action == GLFW_RELEASE) {
            glfwSetWindowShouldClose(windowHandle, true); // 我们将在渲染循环中检测到这一点
        }
    }
    ...
}
```

??? note "为什么arrWidth，arrHeight要用数组"
    ​GLFW 的 C 语言设计​​：
    GLFW 是 C 库，通过指针参数返回多个值（类似 void glfwGetFramebufferSize  ..., int* width, int* height）。


我们首先设置一些窗口提示来隐藏窗口并设置为可调整大小。之后，我们设置OpenGL版本，并根据窗口选项选择核心或兼容配置文件。然后，如果没有设置首选宽度和高度，我们获取主显示器的尺寸来设置窗口大小。随后，我们通过调用`glfwCreateWindow`创建窗口，并在窗口大小改变或检测到窗口终止（按下`ESC`键）时设置回调函数。如果我们想手动设置目标FPS，则调用`glfwSwapInterval(0)`来禁用垂直同步（Vertical Synchronization），最后显示窗口并获取帧缓冲区大小（用于渲染的窗口部分）。

`Window`类的其余方法用于清理资源、处理窗口大小调整回调、获取窗口大小的getter方法，以及轮询事件和检查窗口是否应关闭的方法。

```java
public class Window {
    ...
    public void cleanup() {
        glfwFreeCallbacks(windowHandle);
        glfwDestroyWindow(windowHandle);
        glfwTerminate();
        GLFWErrorCallback callback = glfwSetErrorCallback(null);
        if (callback != null) {
            callback.free();
        }
    }

    public int getHeight() {
        return height;
    }

    public int getWidth() {
        return width;
    }

    public long getWindowHandle() {
        return windowHandle;
    }
    
    public boolean isKeyPressed(int keyCode) {
        return glfwGetKey(windowHandle, keyCode) == GLFW_PRESS;
    }

    public void pollEvents() {
        glfwPollEvents();
    }

    protected void resized(int width, int height) {
        this.width = width;
        this.height = height;
        try {
            resizeFunc.call();
        } catch (Exception excp) {
            Logger.error("Error calling resize callback", excp);
        }
    }

    public void update() {
        glfwSwapBuffers(windowHandle);
    }

    public boolean windowShouldClose() {
        return glfwWindowShouldClose(windowHandle);
    }
    ...
}
```

`Scene`类将用于保存未来3D场景元素（模型等）。目前，它只是一个空的占位符：

```java
package org.lwjglb.engine.scene;

public class Scene {

    public Scene() {
    }

    public void cleanup() {
        // 目前这里无需做任何处理
    }
}
```

`Render`类现在也是一个占位符，只是清除屏幕：

```java
package org.lwjglb.engine.graph;

import org.lwjgl.opengl.GL;
import org.lwjglb.engine.Window;
import org.lwjglb.engine.scene.Scene;

import static org.lwjgl.opengl.GL11.*;

public class Render {

    public Render() {
        GL.createCapabilities();
    }

    public void cleanup() {
        // 目前这里无需做任何处理
    }

    public void render(Window window, Scene scene) {
        glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
    }
}
```

现在，我们可以在一个名为`Engine`的新类中实现游戏循环，内容如下：

```java
package org.lwjglb.engine;

import org.lwjglb.engine.graph.Render;
import org.lwjglb.engine.scene.Scene;

public class Engine {

    public static final int TARGET_UPS = 30;
    private final IAppLogic appLogic;
    private final Window window;
    private Render render;
    private boolean running;
    private Scene scene;
    private int targetFps;
    private int targetUps;

    public Engine(String windowTitle, Window.WindowOptions opts, IAppLogic appLogic) {
        window = new Window(windowTitle, opts, () -> {
            resize();
            return null;
        });
        targetFps = opts.fps;
        targetUps = opts.ups;
        this.appLogic = appLogic;
        render = new Render();
        scene = new Scene();
        appLogic.init(window, scene, render);
        running = true;
    }

    private void cleanup() {
        appLogic.cleanup();
        render.cleanup();
        scene.cleanup();
        window.cleanup();
    }

    private void resize() {
        // 目前这里无需做任何处理
    }
    ...
}
```

`Engine`类在构造函数中接收窗口标题、窗口选项以及`IAppLogic`接口的实现引用。在构造函数中，它创建了`Window`、`Render`和`Scene`类的实例。`cleanup`方法只是调用其他类的清理资源方法。游戏循环在`run`方法中定义，其内容如下：

```java
public class Engine {
    ...
    private void run() {
        long initialTime = System.currentTimeMillis();
        float timeU = 1000.0f / targetUps;
        float timeR = targetFps > 0 ? 1000.0f / targetFps : 0;
        float deltaUpdate = 0;
        float deltaFps = 0;

        long updateTime = initialTime;
        while (running && !window.windowShouldClose()) {
            window.pollEvents();

            long now = System.currentTimeMillis();
            deltaUpdate += (now - initialTime) / timeU;
            deltaFps += (now - initialTime) / timeR;

            if (targetFps <= 0 || deltaFps >= 1) {
                appLogic.input(window, scene, now - initialTime);
            }

            if (deltaUpdate >= 1) {
                long diffTimeMillis = now - updateTime;
                appLogic.update(window, scene, diffTimeMillis);
                updateTime = now;
                deltaUpdate--;
            }

            if (targetFps <= 0 || deltaFps >= 1) {
                render.render(window, scene);
                deltaFps--;
                window.update();
            }
            initialTime = now;
        }

        cleanup();
    }
    ...
}
```

循环开始时计算两个参数：`timeU`和`timeR`，它们分别控制更新（`timeU`）和渲染（`timeR`）调用之间的最大时间间隔（以毫秒为单位）。如果这些时间间隔被消耗完，我们就需要更新游戏状态或进行渲染。如果目标FPS设置为0，我们将依赖于垂直同步（Vertical Synchronization）刷新率，因此将该值设为`0`。循环开始时轮询窗口事件，之后获取当前时间（以毫秒为单位）。然后计算自上次更新和渲染调用以来的时间差。如果超过了渲染最大时间间隔（或依赖于垂直同步），则调用`appLogic.input`处理用户输入。如果超过了更新最大时间间隔，则调用`appLogic.update`更新游戏状态。如果又超过了渲染最大时间间隔（或依赖于垂直同步），则调用`render.render`进行渲染。

循环结束时调用`cleanup`方法释放资源。

最后，`Engine`类完成如下：

```java
public class Engine {
    ...
    public void start() {
        running = true;
        run();
    }

    public void stop() {
        running = false;
    }
}
```

关于线程的简要说明。GLFW要求从主线程进行初始化。事件轮询也应该在主线程中进行。因此，与通常在游戏中会看到创建单独线程运行游戏循环的方式不同，我们将所有内容都在主线程中执行。这就是为什么我们在`start`方法中没有创建新`Thread`的原因。

最后，我们将`Main`类简化为如下内容：

```java
package org.lwjglb.game;

import org.lwjglb.engine.*;
import org.lwjglb.engine.graph.Render;
import org.lwjglb.engine.scene.Scene;

public class Main implements IAppLogic {

    public static void main(String[] args) {
        Main main = new Main();
        Engine gameEng = new Engine("chapter-02", new Window.WindowOptions(), main);
        gameEng.start();
    }

    @Override
    public void cleanup() {
        // 目前这里无需做任何处理
    }

    @Override
    public void init(Window window, Scene scene, Render render) {
        // 目前这里无需做任何处理
    }

    @Override
    public void input(Window window, Scene scene, long diffTimeMillis) {
        // 目前这里无需做任何处理
    }

    @Override
    public void update(Window window, Scene scene, long diffTimeMillis) {
        // 目前这里无需做任何处理
    }
}
```

我们只需在`main`方法中创建`Engine`实例并启动它。`Main`类还实现了`IAppLogic`接口，目前接口方法都为空。

[下一章](./03-our-first-triangle.md)