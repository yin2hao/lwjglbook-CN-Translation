# 第7章 纹理（Textures）

本章我们将学习如何加载纹理、纹理与模型的关联方式以及在渲染过程中如何使用纹理。

你可以在[此处](https://github.com/lwjglgamedev/lwjglbook/tree/main/chapter-07)找到本章的完整源代码。

## 纹理加载（Texture loading）

纹理是一幅映射到模型上用于设置模型像素颜色的图像。你可以将纹理视为包裹在3D模型表面的"皮肤"。具体做法是将图像纹理中的坐标点分配给模型顶点。通过这些信息，OpenGL能够基于纹理图像计算出其他像素应应用的颜色。

![纹理映射](_static/07/texture_mapping.png)

纹理图像不必与模型尺寸相同，它可以更大或更小。当绘制的像素无法直接对应到特定纹理坐标点时，OpenGL会通过插值计算颜色。我们可以在创建特定纹理时控制这一过程。

要为模型应用纹理，我们需要为每个顶点分配纹理坐标。纹理坐标系与模型坐标系有所不同：首先，纹理是二维的，因此坐标只有x和y两个分量；其次，坐标系原点位于图像左上角，x或y的最大值为1。

![纹理坐标](_static/07/texture_coordinates.png)

如何将纹理坐标与位置坐标关联？很简单，就像传递颜色信息一样——我们设置一个VBO，其中包含每个顶点位置对应的纹理坐标。

让我们开始修改代码库，为3D立方体添加纹理支持。第一步是加载用作纹理的图像。为此，我们将使用LWJGL封装的[stb](https://github.com/nothings/stb)库。首先需要在`pom.xml`文件中声明该依赖：

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

接下来我们创建纹理缓存。由于模型经常重用相同纹理，为避免重复加载，我们将已加载的纹理缓存起来。这由`TextureCache`类控制：

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

现在我们需要修改模型定义方式以支持纹理。为此，并准备加载更复杂的模型，我们引入新的`Material`类。这个类将保存纹理路径和`Mesh`实例列表。因此，`Model`实例将与`Material`列表而非`Mesh`直接关联。

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

我们需要传递纹理坐标作为另一个VBO，因此修改`Mesh`类以接受包含纹理坐标的浮点数组：

```java
public class Mesh {
    // 保留原有字段...

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
    
    // 保留其他方法...
}
```

## 使用纹理（Using the textures）

现在我们需要在着色器中使用纹理。顶点着色器中我们修改了第二个输入参数（现在它是`vec2`类型）。片段着色器中使用纹理坐标通过`sampler2D`统一变量从纹理采样颜色。

顶点着色器：
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

片段着色器：
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

在`SceneRender`类中，我们首先为纹理采样器创建新的统一变量，然后在渲染过程中使用纹理：

```java
public class SceneRender {
    // 保留原有字段和方法...

    private void createUniforms() {
        // 原有uniform创建...
        uniformsMap.createUniform("txtSampler");
    }

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

我们还需要修改`UniformsMap`类，添加设置采样器值的新方法：

```java
public class UniformsMap {
    // 保留原有字段和方法...

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
            glUniformMatrix4fv(getUniformLocation(uniformName), false, 
                value.get(stack.mallocFloat(16)));
        }
    }
}
```

现在我们已经修改代码库以支持纹理。接下来需要为3D立方体设置纹理坐标。我们的纹理图像文件如下：

![立方体纹理](_static/07/cube_texture.png)

在3D模型中有8个顶点。我们需要为每个顶点定义纹理坐标。由于同一顶点在不同面上需要不同的纹理坐标，我们不得不重复某些顶点。最终顶点数从8个增加到20个。

在下一章我们将学习如何加载3D建模工具生成的模型，这样就不需要手动定义位置和纹理坐标了（对于更复杂的模型来说，手动定义是不现实的）。

最后修改`Main`类中的`init`方法以定义纹理坐标并加载纹理数据：

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
                // V8: V4重复
                -0.5f, 0.5f, -0.5f,
                // V9: V5重复
                0.5f, 0.5f, -0.5f,
                // V10: V0重复
                -0.5f, 0.5f, 0.5f,
                // V11: V3重复
                0.5f, 0.5f, 0.5f,

                // 为右侧面纹理坐标重复的顶点
                // V12: V3重复
                0.5f, 0.5f, 0.5f,
                // V13: V2重复
                0.5f, -0.5f, 0.5f,

                // 为左侧面纹理坐标重复的顶点
                // V14: V0重复
                -0.5f, 0.5f, 0.5f,
                // V15: V1重复
                -0.5f, -0.5f, 0.5f,

                // 为底面纹理坐标重复的顶点
                // V16: V6重复
                -0.5f, -0.5f, -0.5f,
                // V17: V7重复
                0.5f, -0.5f, -0.5f,
                // V18: V1重复
                -0.5f, -0.5f, 0.5f,
                // V19: V2重复
                0.5f, -0.5f, 0.5f,
        };

        float[] textCoords = new float[]{
                // 前面纹理坐标
                0.0f, 0.0f,
                0.0f, 0.5f,
                0.5f, 0.5f,
                0.5f, 0.0f,

                // 后面纹理坐标
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
                // 前面
                0, 1, 3, 3, 1, 2,
                // 顶面
                8, 10, 11, 9, 8, 11,
                // 右侧面
                12, 13, 7, 5, 12, 7,
                // 左侧面
                14, 15, 6, 4, 14, 6,
                // 底面
                16, 18, 19, 17, 16, 19,
                // 后面
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