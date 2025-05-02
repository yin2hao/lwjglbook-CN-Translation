# 更多关于渲染（More on render）

在本章中，我们将继续讨论**OpenGL**是如何渲染内容的。我们将绘制一个四边形（quad）而不是三角形，并向**网格**（Mesh）设置额外的数据，比如给每个顶点设置一个颜色。

你可以在[这里](https://github.com/lwjglgamedev/lwjglbook/tree/main/chapter-04)找到本章的完整源代码。

# 网格修改（Mesh modification）

正如我们一开始所说的，我们想要绘制一个四边形。一个四边形可以通过使用两个三角形构建出来，如下图所示。

![Quad coordinates](_static/04/quad_coordinates.png)

如你所见，每个三角形都是由三个顶点组成的。第一个三角形由顶点V1、V2和V4构成（橙色的那个），第二个三角形由顶点V4、V2和V3构成（绿色的那个）。顶点是以逆时针顺序指定的，因此传递的浮点数组将是\[V1, V2, V4, V4, V2, V3\]。因此，该形状的数据可以是：

```java
float[] positions = new float[] {
    -0.5f,  0.5f, 0.0f,
    -0.5f, -0.5f, 0.0f,
     0.5f,  0.5f, 0.0f,
     0.5f,  0.5f, 0.0f,
    -0.5f, -0.5f, 0.0f,
     0.5f, -0.5f, 0.0f,
}
```

上面的代码仍然存在一些问题。为了表示四边形，我们重复了顶点坐标。我们把V2和V4的坐标传递了两次。对于这个小形状来说似乎问题不大，但想象一下一个更加复杂的三维模型。我们将会多次重复坐标，就像下面的图所示（其中一个顶点可以被六个三角形共享）。

![Dolphin](_static/04/dolphin.png)

最终我们将需要更多的内存，因为存在这些重复的信息。但更大的问题不在于此，最大的问题是我们将在着色器（shader）中为相同的顶点重复处理。这时候，索引缓冲（Index Buffers）就能派上用场了。为了绘制四边形，我们只需要以这种方式指定每个顶点一次：V1、V2、V3、V4。每个顶点在数组中都有一个位置。V1的位置是0，V2是1，以此类推：

| V1 | V2 | V3 | V4 |
| --- | --- | --- | --- |
| 0 | 1 | 2 | 3 |

然后，我们通过引用它们在数组中的位置来指定绘制这些顶点的顺序：

| 0 | 1 | 3 | 3 | 1 | 2 |
| --- | --- | --- | --- | --- | --- |
| V1 | V2 | V4 | V4 | V2 | V3 |

所以我们需要修改我们的`Mesh`类，来接受另一个参数：索引数组（indices array），并且现在要绘制的顶点数量将是索引数组的长度。还要记住，现在我们仅使用三个浮点数来表示一个顶点的位置，但我们还想为每个顶点关联颜色。因此，我们需要这样修改`Mesh`类：

```java
public class Mesh {
    ...
    public Mesh(float[] positions, float[] colors, int[] indices) {
        numVertices = indices.length;
        ...
        // 颜色VBO
        vboId = glGenBuffers();
        vboIdList.add(vboId);
        FloatBuffer colorsBuffer = MemoryUtil.memCallocFloat(colors.length);
        colorsBuffer.put(0, colors);
        glBindBuffer(GL_ARRAY_BUFFER, vboId);
        glBufferData(GL_ARRAY_BUFFER, colorsBuffer, GL_STATIC_DRAW);
        glEnableVertexAttribArray(1);
        glVertexAttribPointer(1, 3, GL_FLOAT, false, 0, 0);

        // 索引VBO
        vboId = glGenBuffers();
        vboIdList.add(vboId);
        IntBuffer indicesBuffer = MemoryUtil.memCallocInt(indices.length);
        indicesBuffer.put(0, indices);
        glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, vboId);
        glBufferData(GL_ELEMENT_ARRAY_BUFFER, indicesBuffer, GL_STATIC_DRAW);
        ...
        MemoryUtil.memFree(colorsBuffer);
        MemoryUtil.memFree(indicesBuffer);
    }
    ...
}           
```

在我们创建用于存储位置的VBO之后，我们需要创建另一个用于颜色数据的VBO。然后我们再创建一个用于索引的VBO。创建索引VBO的过程与之前类似，但要注意类型现在是`GL_ELEMENT_ARRAY_BUFFER`。由于我们正在处理整数，所以需要创建一个`IntBuffer`而不是`FloatBuffer`。现在，**顶点数组对象**（VAO）将包含三个VBO：一个用于位置，一个用于颜色，一个用于索引，并且将在渲染时使用索引。

之后，我们需要在`SceneRender`类中更改绘制调用以使用索引：

```java
public class SceneRender {
    ...
    public void render(Scene scene) {
        ...
        scene.getMeshMap().values().forEach(mesh -> {
                    glBindVertexArray(mesh.getVaoId());
                    glDrawElements(GL_TRIANGLES, mesh.getNumVertices(), GL_UNSIGNED_INT, 0);
                }
        );
        ...
    }
    ...
}
```

`glDrawElements`方法的参数是：

* mode：指定绘制时的图元类型，这里是三角形。这里没有变化。
* count：指定要渲染的元素数量。
* type：指定索引数据中值的类型。这里我们使用的是整数。
* indices：指定开始渲染时索引数据的偏移量。

现在我们可以在`Main`类中创建一个新的Mesh，带有额外的顶点参数（颜色）和索引：

```java
public class Main implements IAppLogic {
    ...
    public static void main(String[] args) {
        ...
        Engine gameEng = new Engine("chapter-04", new Window.WindowOptions(), main);
        ...
    }
    ...
    public void init(Window window, Scene scene, Render render) {
        float[] positions = new float[]{
                -0.5f, 0.5f, 0.0f,
                -0.5f, -0.5f, 0.0f,
                0.5f, -0.5f, 0.0f,
                0.5f, 0.5f, 0.0f,
        };
        float[] colors = new float[]{
                0.5f, 0.0f, 0.0f,
                0.0f, 0.5f, 0.0f,
                0.0f, 0.0f, 0.5f,
                0.0f, 0.5f, 0.5f,
        };
        int[] indices = new int[]{
                0, 1, 3, 3, 1, 2,
        };
        Mesh mesh = new Mesh(positions, colors, indices);
        scene.addMesh("quad", mesh);
    }
    ...
}
```

现在我们需要修改着色器（shader），不是因为索引，而是为了使用每个顶点的颜色。顶点着色器（`scene.vert`）如下所示：

```glsl
#version 330

layout (location=0) in vec3 position;
layout (location=1) in vec3 color;

out vec3 outColor;

void main()
{
    gl_Position = vec4(position, 1.0);
    outColor = color;
}
```

在输入参数中，你可以看到我们接收了一个新的`vec3`类型的颜色数据，并且我们直接返回它，供片段着色器（`scene.frag`）使用，如下所示：

```glsl
#version 330

in  vec3 outColor;
out vec4 fragColor;

void main()
{
    fragColor = vec4(outColor, 1.0);
}
```

我们仅使用输入的颜色参数作为片段颜色返回。需要注意的是，在片段着色器中使用时，颜色值将被插值，因此结果会像下面这样。

![Colored quad](_static/04/colored_quad.png)

[下一章](./05-perspective-projection.md)