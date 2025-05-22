# 第7章 纹理（Textures）

本章我们将学习如何加载纹理、纹理与模型的关联方式以及在渲染过程中如何使用纹理。

你可以在[此处](https://github.com/lwjglgamedev/lwjglbook/tree/main/chapter-07)找到本章的完整源代码。

## 纹理加载

纹理是一幅映射到模型上用于设置模型像素颜色的图像。你可以将纹理视为包裹在3D模型表面的"皮肤"。具体做法是将图像纹理中的坐标点分配给模型顶点。通过这些信息，OpenGL能够基于纹理图像计算出其他像素应应用的颜色。

![纹理映射](_static/07/texture_mapping.png)

纹理图像不必与模型尺寸相同，它可以更大或更小。当绘制的像素无法直接对应到特定纹理坐标点时，OpenGL会通过插值计算颜色。我们可以在创建特定纹理时控制这一过程。

要为模型应用纹理，我们需要为每个顶点分配纹理坐标。纹理坐标系与模型坐标系有所不同：首先，纹理是二维的，因此坐标只有x和y两个分量；其次，坐标系原点位于图像左上角，x或y的最大值为1。

![纹理坐标](_static/07/texture_coordinates.png)

如何将纹理坐标与位置坐标关联？很简单，就像传递颜色信息一样——我们设置一个VBO，其中包含每个顶点位置对应的纹理坐标。

让我们开始修改代码库，为3D立方体添加纹理支持。第一步是加载用作纹理的图像。为此，我们将使用LWJGL封装的[stb](https://github.com/nothings/stb)库。首先需要在`pom.xml`文件中声明该依赖，包括本地库：

```xml
<dependency>
    <groupId>org.lwjgl</groupId>
    <artifactId>lwjgl-stb</artifactId>
    <version>${lwjgl.version}</version>
</dependency>
[...]
<dependency>
    <groupId>org.lwjgl</groupId>
    <artifactId>lwjgl-stb</artifactId>
    <version>${lwjgl.version}</version>
    <classifier>${native.target}</classifier>
    <scope>runtime</scope>
</dependency>
```

首先创建一个新的`Texture`类，用于执行加载纹理所需的所有步骤：

```java
package org.lwjglb.engine.graph;

import org.lwjgl.system.MemoryStack;

import java.nio.*;

import static org.lwjgl.opengl.GL30.*;
import static org.lwjgl.stb.STBImage.*;

public class Texture {

    private int textureId;
    private String texturePath;

    public Texture(int width, int height, ByteBuffer buf) {
        this.texturePath = "";
        generateTexture(width, height, buf);
    }

    public Texture(String texturePath) {
        try (MemoryStack stack = MemoryStack.stackPush()) {
            this.texturePath = texturePath;
            IntBuffer w = stack.mallocInt(1);
            IntBuffer h = stack.mallocInt(1);
            IntBuffer channels = stack.mallocInt(1);

            ByteBuffer buf = stbi_load(texturePath, w, h, channels, 4);
            if (buf == null) {
                throw new RuntimeException("Image file [" + texturePath + "] not loaded: " + stbi_failure_reason());
            }

            int width = w.get();
            int height = h.get();

            generateTexture(width, height, buf);

            stbi_image_free(buf);
        }
    }

    public void bind() {
        glBindTexture(GL_TEXTURE_2D, textureId);
    }

    public void cleanup() {
        glDeleteTextures(textureId);
    }

    private void generateTexture(int width, int height, ByteBuffer buf) {
        textureId = glGenTextures();

        glBindTexture(GL_TEXTURE_2D, textureId);
        glPixelStorei(GL_UNPACK_ALIGNMENT, 1);
        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_NEAREST);
        glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, width, height, 0,
                GL_RGBA, GL_UNSIGNED_BYTE, buf);
        glGenerateMipmap(GL_TEXTURE_2D);
    }

    public String getTexturePath() {
        return texturePath;
    }
}
```

在构造函数中我们首先要做的事情是为库分配 `IntBuffer`，以用于接收图像的宽度、高度和通道数量。然后我们调用 `stbi_load` 方法将图像实际加载到一个 `ByteBuffer` 中。该方法需要以下几个参数：

* `filePath`：文件的绝对路径。由于 stb 库是一个本地库（native library），它并不了解 `CLASSPATH`，因此我们使用的是普通的文件系统路径。
* `width`：图像宽度，该值将由函数填充。
* `height`：图像高度，该值也将由函数填充。
* `channels`：图像的原始通道数。
* `desired_channels`：期望的图像通道数。我们传入的是 4，即 RGBA。

需要注意的一点是，出于历史原因，OpenGL 要求纹理图像的尺寸（每个维度上的 texel 数）必须是 2 的幂（2、4、8、16，等等）。虽然如今的 OpenGL 驱动大多已经不再强制这一限制，但如果你遇到一些问题，可以尝试将图像尺寸调整为 2 的幂次方。

接下来的步骤是在 GPU 中上传纹理数据。这一过程会在 `generateTexture` 方法中完成。首先我们需要调用 `glGenTextures` 函数生成一个新的纹理标识符。接着通过 `glBindTexture` 绑定该纹理。然后，我们需要告诉 OpenGL 如何解包 RGBA 字节数据。由于每个分量正好是一个字节，因此我们在调用 `glPixelStorei` 函数时使用 `GL_UNPACK_ALIGNMENT`。

最后，我们通过调用 `glTexImage2D` 函数来加载纹理数据。

`glTexImage2D` 方法的参数如下：

* `target`：指定目标纹理类型，这里使用 `GL_TEXTURE_2D`。
* `level`：指定细节级别（Level-of-Detail），0 表示基础图像，n 表示第 n 层 mipmap（更多内容稍后介绍）。
* `internal format`：指定纹理中颜色组件的数量。
* `width`：纹理图像的宽度。
* `height`：纹理图像的高度。
* `border`：必须为 0。
* `format`：像素数据的格式，这里是 RGBA。
* `type`：像素数据的数据类型，这里使用的是无符号字节。
* `data`：包含纹理数据的缓冲区。

之后，我们通过调用 `glTexParameteri` 函数来指定在纹理坐标无法与像素一一对应的情况下应如何采样纹理——此处我们选择最近邻插值（Nearest Filtering）。再之后我们生成 mipmap。mipmap 是从高分辨率纹理生成的一系列低分辨率纹理图，它们在模型缩放时自动使用，提升性能与视觉质量。我们通过调用 `glGenerateMipmap` 函数来生成这些 mipmap。

到这里，我们就成功加载并上传了纹理数据。接下来要做的就是使用它。

现在我们将创建一个纹理缓存。由于模型经常会重复使用相同的纹理，因此我们不希望每次都重新加载同一张纹理，而是只加载一次并复用。这项功能将由 `TextureCache` 类来实现：

```java
package org.lwjglb.engine.graph;

import java.util.*;

public class TextureCache {

    public static final String DEFAULT_TEXTURE = "resources/models/default/default_texture.png";

    private Map<String, Texture> textureMap;

    public TextureCache() {
        textureMap = new HashMap<>();
        textureMap.put(DEFAULT_TEXTURE, new Texture(DEFAULT_TEXTURE));
    }

    public void cleanup() {
        textureMap.values().forEach(Texture::cleanup);
    }

    public Texture createTexture(String texturePath) {
        return textureMap.computeIfAbsent(texturePath, Texture::new);
    }

    public Texture getTexture(String texturePath) {
        Texture texture = null;
        if (texturePath != null) {
            texture = textureMap.get(texturePath);
        }
        if (texture == null) {
            texture = textureMap.get(DEFAULT_TEXTURE);
        }
        return texture;
    }
}
```

正如你所见，我们只是将加载的纹理存储在一个`Map`中，并在纹理路径为null（即模型没有纹理）的情况下返回一个默认纹理。这个默认纹理是一个黑色图像，它可以与那些未定义纹理但定义了颜色的模型结合使用，因此我们可以在片元着色器中将二者结合。`TextureCache`类的实例将被保存在`Scene`类中：

```java
public class Scene {
    ...
    private TextureCache textureCache;
    ...
    public Scene(int width, int height) {
        ...
        textureCache = new TextureCache();
    }
    ...
    public TextureCache getTextureCache() {
        return textureCache;
    }
    ...
}
```

现在我们需要更改模型的定义方式，以添加对纹理的支持。为此，也为了为接下来章节中将加载的更复杂模型做好准备，我们将引入一个名为 `Material` 的新类。这个类将包含纹理路径以及一个 `Mesh` 实例的列表。因此，我们将把 `Model` 实例与一个 `Material` 列表关联，而不再是 `Mesh` 列表。在接下来的章节中，`Material` 还将能够包含其他属性，比如漫反射颜色或高光颜色等。

`Material`类定义如下：

```java
package org.lwjglb.engine.graph;

import java.util.*;

public class Material {

    private List<Mesh> meshList;
    private String texturePath;

    public Material() {
        meshList = new ArrayList<>();
    }

    public void cleanup() {
        meshList.forEach(Mesh::cleanup);
    }

    public List<Mesh> getMeshList() {
        return meshList;
    }

    public String getTexturePath() {
        return texturePath;
    }

    public void setTexturePath(String texturePath) {
        this.texturePath = texturePath;
    }
}
```

由于`Mesh`实例现在属于`Material`类，我们需要相应修改`Model`类：

```java
package org.lwjglb.engine.graph;

import org.lwjglb.engine.scene.Entity;

import java.util.*;

public class Model {

    private final String id;
    private List<Entity> entitiesList;
    private List<Material> materialList;

    public Model(String id, List<Material> materialList) {
        this.id = id;
        entitiesList = new ArrayList<>();
        this.materialList = materialList;
    }

    public void cleanup() {
        materialList.forEach(Material::cleanup);
    }

    public List<Entity> getEntitiesList() {
        return entitiesList;
    }

    public String getId() {
        return id;
    }

    public List<Material> getMaterialList() {
        return materialList;
    }
}
```

正如我们之前所说，我们需要传递纹理坐标作为另一个VBO。因此，我们将修改`Mesh`类以接受包含纹理坐标的浮点数组，而不是颜色。`Mesh`类修改如下：

```java
public class Mesh {
    ...
    public Mesh(float[] positions, float[] textCoords, int[] indices) {
        numVertices = indices.length;
        vboIdList = new ArrayList<>();

        vaoId = glGenVertexArrays();
        glBindVertexArray(vaoId);

        // Positions VBO
        int vboId = glGenBuffers();
        vboIdList.add(vboId);
        FloatBuffer positionsBuffer = MemoryUtil.memCallocFloat(positions.length);
        positionsBuffer.put(0, positions);
        glBindBuffer(GL_ARRAY_BUFFER, vboId);
        glBufferData(GL_ARRAY_BUFFER, positionsBuffer, GL_STATIC_DRAW);
        glEnableVertexAttribArray(0);
        glVertexAttribPointer(0, 3, GL_FLOAT, false, 0, 0);

        // Texture coordinates VBO
        vboId = glGenBuffers();
        vboIdList.add(vboId);
        FloatBuffer textCoordsBuffer = MemoryUtil.memCallocFloat(textCoords.length);
        textCoordsBuffer.put(0, textCoords);
        glBindBuffer(GL_ARRAY_BUFFER, vboId);
        glBufferData(GL_ARRAY_BUFFER, textCoordsBuffer, GL_STATIC_DRAW);
        glEnableVertexAttribArray(1);
        glVertexAttribPointer(1, 2, GL_FLOAT, false, 0, 0);

        // Index VBO
        vboId = glGenBuffers();
        vboIdList.add(vboId);
        IntBuffer indicesBuffer = MemoryUtil.memCallocInt(indices.length);
        indicesBuffer.put(0, indices);
        glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, vboId);
        glBufferData(GL_ELEMENT_ARRAY_BUFFER, indicesBuffer, GL_STATIC_DRAW);

        glBindBuffer(GL_ARRAY_BUFFER, 0);
        glBindVertexArray(0);

        MemoryUtil.memFree(positionsBuffer);
        MemoryUtil.memFree(textCoordsBuffer);
        MemoryUtil.memFree(indicesBuffer);
    }
    ...
}
```

## 使用纹理

现在我们需要在着色器中使用纹理。在顶点着色器中，我们改变了第二个输入参数，因为它现在是`vec2`类型（我们也改变了参数名称）。顶点着色器，就像处理颜色一样，只是将纹理坐标传递给片段着色器使用。

```glsl
#version 330

layout (location=0) in vec3 position;
layout (location=1) in vec2 texCoord;

out vec2 outTextCoord;

uniform mat4 projectionMatrix;
uniform mat4 modelMatrix;

void main()
{
    gl_Position = projectionMatrix * modelMatrix * vec4(position, 1.0);
    outTextCoord = texCoord;
}
```

在片段着色器中，我们必须使用纹理坐标来通过采样纹理（通过一个 `sampler2D` 统一变量）来设置像素颜色。

```glsl
#version 330

in vec2 outTextCoord;

out vec4 fragColor;

uniform sampler2D txtSampler;

void main()
{
    fragColor = texture(txtSampler, outTextCoord);
}
```

现在我们将看到这一切如何在`SceneRender`类中使用。首先，我们需要为纹理采样器创建一个新的统一变量。

```java
public class SceneRender {
    ...
    private void createUniforms() {
        ...
        uniformsMap.createUniform("txtSampler");
    }
    ...
}
```

现在，我们可以在渲染过程中使用纹理：

```java
public class SceneRender {
    ...
    public void render(Scene scene) {
        shaderProgram.bind();

        uniformsMap.setUniform("projectionMatrix", scene.getProjection().getProjMatrix());

        uniformsMap.setUniform("txtSampler", 0);

        Collection<Model> models = scene.getModelMap().values();
        TextureCache textureCache = scene.getTextureCache();
        for (Model model : models) {
            List<Entity> entities = model.getEntitiesList();

            for (Material material : model.getMaterialList()) {
                Texture texture = textureCache.getTexture(material.getTexturePath());
                glActiveTexture(GL_TEXTURE0);
                texture.bind();

                for (Mesh mesh : material.getMeshList()) {
                    glBindVertexArray(mesh.getVaoId());
                    for (Entity entity : entities) {
                        uniformsMap.setUniform("modelMatrix", entity.getModelMatrix());
                        glDrawElements(GL_TRIANGLES, mesh.getNumVertices(), GL_UNSIGNED_INT, 0);
                    }
                }
            }
        }

        glBindVertexArray(0);

        shaderProgram.unbind();
    }
}
```

正如你所见，我们首先将纹理采样器统一变量设置为 `0`。让我们解释一下为什么这样做。显卡有几个空间或插槽来存储纹理。每个这样的空间称为一个纹理单元。当我们使用纹理时，必须设置我们想要使用的纹理单元。在这种情况下，我们只使用一个纹理，所以我们将使用纹理单元 `0`。统一变量的类型是 `sampler2D`，它将保存我们想要使用的纹理单元的值。当我们遍历模型和材质时，我们从缓存中获取与每个材质关联的纹理，通过调用 `glActiveTexture` 函数并传入参数 `GL_TEXTURE0` 来激活纹理单元，然后绑定它。这就是我们将纹理单元与纹理标识符关联起来的方式。

我们还需要修改`UniformsMap`类，添加一个接受整数的新方法来设置采样器值，该方法也将被称为`setUniform`，但接受统一变量的名称和一个整数值。由于设置矩阵的`setUniform`方法和这个新方法之间会重复一些代码，我们将提取获取统一变量位置的代码部分到一个名为`getUniformLocation`的新方法中。`UniformsMap`类的更改如下所示：

```java
public class UniformsMap {
    ...
    private int getUniformLocation(String uniformName) {
        Integer location = uniforms.get(uniformName);
        if (location == null) {
            throw new RuntimeException("Could not find uniform [" + uniformName + "]");
        }
        return location.intValue();
    }

    public void setUniform(String uniformName, int value) {
        glUniform1i(getUniformLocation(uniformName), value);
    }

    public void setUniform(String uniformName, Matrix4f value) {
        try (MemoryStack stack = MemoryStack.stackPush()) {
            glUniformMatrix4fv(getUniformLocation(uniformName), false, value.get(stack.mallocFloat(16)));
        }
    }
    ...
}
```

现在，我们已经修改了代码库以支持纹理。现在我们需要为我们的3D立方体设置纹理坐标。我们的纹理图像文件将是这样的：

![立方体纹理](_static/07/cube_texture.png)

在我们的3D模型中，有八个顶点。让我们看看如何做到这一点。首先定义每个顶点的前面的纹理坐标。

![前面纹理坐标](_static/07/cube_texture_front_face.png)

| 顶点 | 纹理坐标 |
| --- | --- |
| V0 | \(0.0, 0.0\) |
| V1 | \(0.0, 0.5\) |
| V2 | \(0.5, 0.5\) |
| V3 | \(0.5, 0.0\) |

现在，让我们定义顶面的纹理映射。

![顶面纹理坐标](_static/07/cube_texture_top_face.png)

| 顶点 | 纹理坐标 |
| --- | --- |
| V4 | \(0.0, 0.5\) |
| V5 | \(0.5, 0.5\) |
| V0 | \(0.0, 1.0\) |
| V3 | \(0.5, 1.0\) |

正如你所见，我们遇到了一个问题，我们需要为相同的顶点（V0和V3）设置不同的纹理坐标。如何解决这个问题？唯一的解决方法是重复一些顶点并关联不同的纹理坐标。对于顶面，我们需要重复四个顶点并为它们分配正确的纹理坐标。

由于正面、背面和侧面使用相同的纹理，我们不需要重复所有这些顶点。你可以在源代码中找到完整的定义，但我们需要从8个点增加到20个。

在接下来的章节中，我们将学习如何加载由3D建模工具生成的模型，这样我们就不需要手动定义位置和纹理坐标了（顺便说一句，对于更复杂的模型来说，手动定义是不切实际的）。

我们只需要修改`Main`类中的`init`方法来定义纹理坐标并加载纹理数据：

```java
public class Main implements IAppLogic {
    private Entity cubeEntity;

    public static void main(String[] args) {
        Main main = new Main();
        Engine gameEng = new Engine("chapter-07", new Window.WindowOptions(), main);
        gameEng.start();
    }

    public void init(Window window, Scene scene, Render render) {
        float[] positions = new float[]{
                // V0
                -0.5f, 0.5f, 0.5f,
                // V1
                -0.5f, -0.5f, 0.5f,
                // V2
                0.5f, -0.5f, 0.5f,
                // V3
                0.5f, 0.5f, 0.5f,
                // V4
                -0.5f, 0.5f, -0.5f,
                // V5
                0.5f, 0.5f, -0.5f,
                // V6
                -0.5f, -0.5f, -0.5f,
                // V7
                0.5f, -0.5f, -0.5f,

                // 为顶面纹理坐标重复的顶点
                // V8: V4 repeated
                -0.5f, 0.5f, -0.5f,
                // V9: V5 repeated
                0.5f, 0.5f, -0.5f,
                // V10: V0 repeated
                -0.5f, 0.5f, 0.5f,
                // V11: V3 repeated
                0.5f, 0.5f, 0.5f,

                // 为右侧面纹理坐标重复的顶点
                // V12: V3 repeated
                0.5f, 0.5f, 0.5f,
                // V13: V2 repeated
                0.5f, -0.5f, 0.5f,

                // 为左侧面纹理坐标重复的顶点
                // V14: V0 repeated
                -0.5f, 0.5f, 0.5f,
                // V15: V1 repeated
                -0.5f, -0.5f, 0.5f,

                // 为底面纹理坐标重复的顶点
                // V16: V6 repeated
                -0.5f, -0.5f, -0.5f,
                // V17: V7 repeated
                0.5f, -0.5f, -0.5f,
                // V18: V1 repeated
                -0.5f, -0.5f, 0.5f,
                // V19: V2 repeated
                0.5f, -0.5f, 0.5f,
        };

        float[] textCoords = new float[]{
                // 正面纹理坐标
                0.0f, 0.0f,
                0.0f, 0.5f,
                0.5f, 0.5f,
                0.5f, 0.0f,

                // 背面纹理坐标
                0.0f, 0.0f,
                0.5f, 0.0f,
                0.0f, 0.5f,
                0.5f, 0.5f,

                // 顶面纹理坐标
                0.0f, 0.5f,
                0.5f, 0.5f,
                0.0f, 1.0f,
                0.5f, 1.0f,

                // 右侧面纹理坐标
                0.0f, 0.0f,
                0.0f, 0.5f,

                // 左侧面纹理坐标
                0.5f, 0.0f,
                0.5f, 0.5f,

                // 底面纹理坐标
                0.5f, 0.0f,
                1.0f, 0.0f,
                0.5f, 0.5f,
                1.0f, 0.5f,
        };

        int[] indices = new int[]{
                // 正面
                0, 1, 3, 3, 1, 2,
                // 顶面
                8, 10, 11, 9, 8, 11,
                // 右侧面
                12, 13, 7, 5, 12, 7,
                // 左侧面
                14, 15, 6, 4, 14, 6,
                // 底面
                16, 18, 19, 17, 16, 19,
                // 背面
                4, 6, 7, 5, 4, 7,
        };

        Texture texture = scene.getTextureCache().createTexture("resources/models/cube/cube.png");
        Material material = new Material();
        material.setTexturePath(texture.getTexturePath());
        List<Material> materialList = new ArrayList<>();
        materialList.add(material);

        Mesh mesh = new Mesh(positions, textCoords, indices);
        material.getMeshList().add(mesh);
        Model cubeModel = new Model("cube-model", materialList);
        scene.addModel(cubeModel);

        cubeEntity = new Entity("cube-entity", cubeModel.getId());
        cubeEntity.setPosition(0, 0, -2);
        scene.addEntity(cubeEntity);
    }

    // 保留其他方法...
}
```

最终效果如下图所示：

![带纹理的立方体](_static/07/cube_with_texture.png)

[下一章](./08-camera.md)