# 第09章 - 加载复杂模型：Assimp（Loading more complex models: Assimp）

为了开发游戏，能够加载不同格式的复杂3D模型至关重要。为某些格式编写解析器需要大量工作，即使仅支持单一格式也可能耗时。幸运的是，[**Assimp**](http://assimp.sourceforge.net/)库已可用于解析多种常见3D格式。这是一个C/C++库，能加载多种格式的静态和动态模型。轻量级Java游戏库提供了从Java代码调用的绑定。本章将介绍其使用方法。

本章完整源代码可在[此处](https://github.com/lwjglgamedev/lwjglbook/tree/main/chapter-09)找到。

## 模型加载器

首先在项目的pom.xml中添加Assimp的Maven依赖。我们需要添加编译时和运行时依赖。

```xml
<dependency>
    <groupId>org.lwjgl</groupId>
    <artifactId>lwjgl-assimp</artifactId>
    <version>${lwjgl.version}</version>
</dependency>
<dependency>
    <groupId>org.lwjgl</groupId>
    <artifactId>lwjgl-assimp</artifactId>
    <version>${lwjgl.version}</version>
    <classifier>${native.target}</classifier>
    <scope>runtime</scope>
</dependency>
```

设置依赖后，我们将创建一个名为`ModelLoader`的新类，用于通过Assimp加载模型。该类定义了两个静态公共方法：

```java
package org.lwjglb.engine.scene;

import org.joml.Vector4f;
import org.lwjgl.PointerBuffer;
import org.lwjgl.assimp.*;
import org.lwjgl.system.MemoryStack;
import org.lwjglb.engine.graph.*;

import java.io.File;
import java.nio.IntBuffer;
import java.util.*;

import static org.lwjgl.assimp.Assimp.*;

public class ModelLoader {

    private ModelLoader() {
        // 工具类
    }

    public static Model loadModel(String modelId, String modelPath, TextureCache textureCache) {
        return loadModel(modelId, modelPath, textureCache, aiProcess_GenSmoothNormals | aiProcess_JoinIdenticalVertices |
                aiProcess_Triangulate | aiProcess_FixInfacingNormals | aiProcess_CalcTangentSpace | aiProcess_LimitBoneWeights |
                aiProcess_PreTransformVertices);

    }

    public static Model loadModel(String modelId, String modelPath, TextureCache textureCache, int flags) {
        ...

    }
    ...
}
```

两个方法具有以下参数：

* `modelId`：要加载模型的唯一标识符。

* `modelPath`：模型文件所在路径。这是常规文件路径，不是CLASSPATH相对路径，因为Assimp可能需要加载其他文件，并可能使用与`modelPath`相同的基础路径（例如，Wavefront OBJ文件的材质文件）。如果将资源嵌入JAR文件中，Assimp将无法导入，因此必须是文件系统路径。加载纹理时，我们将使用`modelPath`获取模型所在的基础目录以加载纹理（覆盖模型中定义的任何路径）。这样做是因为某些模型包含开发时本地文件夹的绝对路径，这些路径显然无法访问。

* `textureCache`：对纹理缓存的引用，以避免多次加载同一纹理。

第二个方法有一个额外的参数`flags`。此参数允许调整加载过程。第一个方法调用第二个方法，并传递一些在大多数情况下有用的值：

* `aiProcess_JoinIdenticalVertices`：此标志减少使用的顶点数量，识别可在面之间重复使用的顶点。

* `aiProcess_Triangulate`：模型可能使用四边形或其他几何形状定义其元素。由于我们仅处理三角形，必须使用此标志将所有面分割为三角形（如果需要）。

* `aiProcess_FixInfacingNormals`：此标志尝试反转可能指向内部的法线。

* `aiProcess_CalcTangentSpace`：我们将在实现光照时使用此参数，但它基本上使用法线信息计算切线和副切线。  

* `aiProcess_LimitBoneWeights`：我们将在实现动画时使用此参数，但它基本上限制影响单个顶点的权重数量。

* `aiProcess_PreTransformVertices`：此标志对加载的数据执行一些变换，使模型位于原点，并将坐标校正为匹配OpenGL坐标系。如果遇到模型旋转问题，请确保使用此标志。重要提示：如果模型使用动画，请勿使用此标志，此标志将删除动画信息。

还有许多其他标志可用，可以在LWJGL或Assimp文档中查看。

让我们回到第二个构造函数。首先调用`aiImportFile`方法以加载具有所选标志的模型。

```java
public class ModelLoader {
    ...
    public static Model loadModel(String modelId, String modelPath, TextureCache textureCache, int flags) {
        File file = new File(modelPath);
        if (!file.exists()) {
            throw new RuntimeException("Model path does not exist [" + modelPath + "]");
        }
        String modelDir = file.getParent();

        AIScene aiScene = aiImportFile(modelPath, flags);
        if (aiScene == null) {
            throw new RuntimeException("Error loading model [modelPath: " + modelPath + "]");
        }
        ...
    }
    ...
}
```

构造函数的其余代码如下：

```java
public class ModelLoader {
    ...
    public static Model loadModel(String modelId, String modelPath, TextureCache textureCache, int flags) {
        ...
        int numMaterials = aiScene.mNumMaterials();
        List<Material> materialList = new ArrayList<>();
        for (int i = 0; i < numMaterials; i++) {
            AIMaterial aiMaterial = AIMaterial.create(aiScene.mMaterials().get(i));
            materialList.add(processMaterial(aiMaterial, modelDir, textureCache));
        }

        int numMeshes = aiScene.mNumMeshes();
        PointerBuffer aiMeshes = aiScene.mMeshes();
        Material defaultMaterial = new Material();
        for (int i = 0; i < numMeshes; i++) {
            AIMesh aiMesh = AIMesh.create(aiMeshes.get(i));
            Mesh mesh = processMesh(aiMesh);
            int materialIdx = aiMesh.mMaterialIndex();
            Material material;
            if (materialIdx >= 0 && materialIdx < materialList.size()) {
                material = materialList.get(materialIdx);
            } else {
                material = defaultMaterial;
            }
            material.getMeshList().add(mesh);
        }

        if (!defaultMaterial.getMeshList().isEmpty()) {
            materialList.add(defaultMaterial);
        }

        return new Model(modelId, materialList);
    }
    ...
}
```

我们处理模型中包含的材质。材质定义了组成模型的网格使用的颜色和纹理。然后我们处理不同的网格。一个模型可以定义多个网格，每个网格可以使用模型定义的一种材质。这就是为什么我们在材质之后处理网格并链接到它们，以避免在渲染时重复绑定调用。

如果检查上面的代码，可能会看到许多对Assimp库的调用返回`PointerBuffer`实例。可以将它们视为C指针，它们只是指向包含数据的内存区域。需要提前知道它们保存的数据类型才能处理它们。对于材质，我们迭代该缓冲区创建`AIMaterial`类的实例。在第二种情况下，我们迭代保存网格数据的缓冲区创建`AIMesh`类的实例。

让我们检查`processMaterial`方法。

```java
public class ModelLoader {
    ...
    private static Material processMaterial(AIMaterial aiMaterial, String modelDir, TextureCache textureCache) {
        Material material = new Material();
        try (MemoryStack stack = MemoryStack.stackPush()) {
            AIColor4D color = AIColor4D.create();

            int result = aiGetMaterialColor(aiMaterial, AI_MATKEY_COLOR_DIFFUSE, aiTextureType_NONE, 0,
                    color);
            if (result == aiReturn_SUCCESS) {
                material.setDiffuseColor(new Vector4f(color.r(), color.g(), color.b(), color.a()));
            }

            AIString aiTexturePath = AIString.calloc(stack);
            aiGetMaterialTexture(aiMaterial, aiTextureType_DIFFUSE, 0, aiTexturePath, (IntBuffer) null,
                    null, null, null, null, null);
            String texturePath = aiTexturePath.dataString();
            if (texturePath != null && texturePath.length() > 0) {
                material.setTexturePath(modelDir + File.separator + new File(texturePath).getName());
                textureCache.createTexture(material.getTexturePath());
                material.setDiffuseColor(Material.DEFAULT_COLOR);
            }

            return material;
        }
    }
    ...
}
```

我们首先获取材质颜色，这里是漫反射颜色（通过设置`AI_MATKEY_COLOR_DIFFUSE`标志）。有许多不同类型的颜色，我们将在应用光照时使用，例如我们有漫反射、环境光（用于环境光）、镜面反射（用于光照的镜面反射因子等）。之后，我们检查材质是否定义了纹理。如果有，即存在纹理路径，我们存储纹理路径并将纹理创建委托给`TexturCache`类，如前面的示例。在这种情况下，如果材质定义了纹理，我们将漫反射颜色设置为默认值（黑色）。通过这样做，我们将能够同时使用漫反射颜色和纹理，而无需检查是否存在纹理。如果模型未定义纹理，我们将使用可以结合材质颜色的默认黑色纹理。

`processMesh`方法定义如下。

```java
public class ModelLoader {
    ...
    private static Mesh processMesh(AIMesh aiMesh) {
        float[] vertices = processVertices(aiMesh);
        float[] textCoords = processTextCoords(aiMesh);
        int[] indices = processIndices(aiMesh);

        // 纹理坐标可能未填充。我们至少需要空槽
        if (textCoords.length == 0) {
            int numElements = (vertices.length / 3) * 2;
            textCoords = new float[numElements];
        }

        return new Mesh(vertices, textCoords, indices);
    }
    ...
}
```

网格由一组顶点位置、纹理坐标和**索引缓冲**（Index Buffer）定义。每个元素在`processVertices`、`processTextCoords`和`processIndices`方法中处理。处理完所有数据后，我们检查是否定义了纹理坐标。如果没有，我们仅分配一组纹理坐标为0.0f以确保顶点数组对象的一致性。

`processXXX`方法非常简单，它们只是在`AIMesh`实例上调用相应的方法，返回所需数据并将其存储到数组中：

```java
public class ModelLoader {
    ...
    private static int[] processIndices(AIMesh aiMesh) {
        List<Integer> indices = new ArrayList<>();
        int numFaces = aiMesh.mNumFaces();
        AIFace.Buffer aiFaces = aiMesh.mFaces();
        for (int i = 0; i < numFaces; i++) {
            AIFace aiFace = aiFaces.get(i);
            IntBuffer buffer = aiFace.mIndices();
            while (buffer.remaining() > 0) {
                indices.add(buffer.get());
            }
        }
        return indices.stream().mapToInt(Integer::intValue).toArray();
    }
    ...
    private static float[] processTextCoords(AIMesh aiMesh) {
        AIVector3D.Buffer buffer = aiMesh.mTextureCoords(0);
        if (buffer == null) {
            return new float[]{};
        }
        float[] data = new float[buffer.remaining() * 2];
        int pos = 0;
        while (buffer.remaining() > 0) {
            AIVector3D textCoord = buffer.get();
            data[pos++] = textCoord.x();
            data[pos++] = 1 - textCoord.y();
        }
        return data;
    }

    private static float[] processVertices(AIMesh aiMesh) {
        AIVector3D.Buffer buffer = aiMesh.mVertices();
        float[] data = new float[buffer.remaining() * 3];
        int pos = 0;
        while (buffer.remaining() > 0) {
            AIVector3D textCoord = buffer.get();
            data[pos++] = textCoord.x();
            data[pos++] = textCoord.y();
            data[pos++] = textCoord.z();
        }
        return data;
    }
}
```

可以看到，通过调用`mVertices`方法获取顶点的缓冲区。我们简单地处理它们以创建一个包含顶点位置的浮点数列表。由于该方法仅返回一个缓冲区，可以直接将该信息传递给创建顶点的OpenGL方法。我们不这样做有两个原因。第一个原因是尽量减少对代码库的修改。第二个原因是通过加载到中间结构中，可以执行一些预处理任务甚至调试加载过程。

如果想查看更高效的方法示例，即直接将缓冲区传递给OpenGL，可以查看此[示例](https://github.com/LWJGL/lwjgl3-demos/blob/master/src/org/lwjgl/demo/opengl/assimp/WavefrontObjDemo.java)。

## 使用模型

我们需要修改`Material`类以添加对漫反射颜色的支持：

```java
public class Material {
 
    public static final Vector4f DEFAULT_COLOR = new Vector4f(0.0f, 0.0f, 0.0f, 1.0f);

    private Vector4f diffuseColor;
    ...
    public Material() {
        diffuseColor = DEFAULT_COLOR;
        ...
    }
    ...
    public Vector4f getDiffuseColor() {
        return diffuseColor;
    }
    ...
    public void setDiffuseColor(Vector4f diffuseColor) {
        this.diffuseColor = diffuseColor;
    }
    ...
}
```

在`SceneRender`类中，我们需要创建并在渲染时正确设置材质漫反射颜色：
```java
public class SceneRender {
    ...
    private void createUniforms() {
        ...
        uniformsMap.createUniform("material.diffuse");
    }

    public void render(Scene scene) {
        ...
        for (Model model : models) {
            List<Entity> entities = model.getEntitiesList();

            for (Material material : model.getMaterialList()) {
                uniformsMap.setUniform("material.diffuse", material.getDiffuseColor());
                ...
            }
        }
        ...
    }
    ...
}
```

可以看到，我们为统一变量使用了一个带有`.`的奇怪名称。这是因为我们将在着色器中使用结构体。通过结构体，我们可以将多个类型组合成一个组合类型。可以在片段着色器中看到这一点：

```glsl
#version 330

in vec2 outTextCoord;

out vec4 fragColor;

struct Material
{
    vec4 diffuse;
};

uniform sampler2D txtSampler;
uniform Material material;

void main()
{
    fragColor = texture(txtSampler, outTextCoord) + material.diffuse;
}
```

我们还需要在`UniformsMap`类中添加一个新方法以支持传递`Vector4f`值
```java
public class UniformsMap {
    ...
    public void setUniform(String uniformName, Vector4f value) {
        glUniform4f(getUniformLocation(uniformName), value.x, value.y, value.z, value.w);
    }
}
```

最后，我们需要修改`Main`类以使用`ModelLoader`类加载模型：
```java
public class Main implements IAppLogic {
    ...
    public static void main(String[] args) {
        ...
        Engine gameEng = new Engine("chapter-09", new Window.WindowOptions(), main);
        ...
    }
    ...
    public void init(Window window, Scene scene, Render render) {
        Model cubeModel = ModelLoader.loadModel("cube-model", "resources/models/cube/cube.obj",
                scene.getTextureCache());
        scene.addModel(cubeModel);

        cubeEntity = new Entity("cube-entity", cubeModel.getId());
        cubeEntity.setPosition(0, 0, -2);
        scene.addEntity(cubeEntity);
    }
    ...
}
```

可以看到，`init`方法已经简化了很多，不再在代码中嵌入模型数据。现在我们使用Wavefront格式的立方体模型。可以在`resources\models\cube`文件夹中找到模型文件。那里有以下文件：

* `cube.obj`：主模型文件。实际上是基于文本的格式，可以打开它查看顶点、索引和纹理坐标如何定义并通过定义面粘合在一起。它还包含对材质文件的引用。

* `cube.mtl`：材质文件，定义颜色和纹理。

* `cube.png`：模型的纹理文件。

最后，我们将添加另一个功能来优化渲染。我们将通过应用面剔除减少渲染的数据量。众所周知，立方体由六个面组成，而我们正在渲染所有六个面，即使它们不可见。如果放大立方体内部，可以看到其内部。

无法看到的面应立即丢弃，这就是面剔除的作用。实际上，对于立方体，同一时间只能看到3个面，因此仅通过应用面剔除就可以丢弃一半的面（仅当游戏不需要进入模型内部时才有效）。

对于每个三角形，面剔除检查它是否面向我们，并丢弃不面向该方向的三角形。但是，我们如何知道三角形是否面向我们？OpenGL通过组成三角形的顶点的缠绕顺序来实现这一点。

记得在第一章中，我们可以按顺时针或逆时针顺序定义三角形的顶点。在OpenGL中，默认情况下，逆时针顺序的三角形面向观察者，顺时针顺序的三角形面向后方。这里的关键是，在渲染时根据视角检查此顺序。因此，按逆时针顺序定义的三角形在渲染时可能因为视角而被解释为顺时针顺序定义。

我们将在`Render`类中启用面剔除：

```java
public class Render {
    ...
    public Render() {
        ...
        glEnable(GL_CULL_FACE);
        glCullFace(GL_BACK);
        ...
    }
    ...
}
```

第一行将启用面剔除，第二行声明应剔除（移除）面向后方的面。

如果运行示例，将看到与前一章相同的结果，但如果放大立方体内部，内部面将不会被渲染。可以修改此示例以加载更复杂的模型。
 
[下一章](./10-GUI.md)