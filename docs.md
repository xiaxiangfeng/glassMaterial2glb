# Three.js 真实感渲染：从环境到材质深度解析

我们将一起探索如何运用 Three.js 来创建更逼真的三维场景，特别是理解环境设置的重要性，以及如何调校PBR（Physically Based Rendering，基于物理的渲染）材质，让建筑模型拥有更真实的玻璃感、金属感和墙体质感。

## 一、真实感渲染的基石：PBR 简介

在计算机图形学中，PBR 是一套旨在模拟真实世界光照和材质交互方式的理论和着色技术。它的核心思想是，与其凭经验和"魔法数字"去凑效果，不如遵循物理光学原理来定义材质属性和光照行为。这样不仅能得到更可信的视觉效果，而且材质在不同光照环境下的表现也会更加一致和自然。

项目中的 `MeshStandardMaterial` 和 `MeshPhysicalMaterial` 就是 Three.js 中实现 PBR 的主力。

## 二、核心图形学概念：为真实感铺路

在我们深入代码之前，先了解几个关键的图形学概念：

### 1. 色彩空间 (Color Spaces) 与线性工作流 (Linear Workflow)

想象一下，光在真实世界中是以"线性"方式叠加的（例如，两个光源的亮度直接相加）。但我们的显示器（以及很多图像文件格式，如 JPG、PNG）通常使用"非线性"的色彩空间，最常见的是 sRGB。sRGB 会对颜色值进行一种叫做"伽马校正 (Gamma Correction)"的非线性变换，这让人眼在屏幕上感知到的亮度更自然。

**为什么重要？**
如果我们在进行光照计算（混合颜色、计算反射等）时不考虑这种差异，直接用 sRGB 空间下的颜色值进行线性运算，结果就会不准确，通常会导致场景偏暗，高光部分丢失细节。

**线性工作流**：
核心思想是在光照计算时，将所有颜色输入（如纹理颜色、材质颜色）转换到线性空间，进行物理上正确的计算，然后在渲染的最后阶段，将结果转换回 sRGB 空间（或其他显示设备的目标空间）以正确显示。

在代码中：
```javascript
renderer.outputColorSpace = THREE.SRGBColorSpace;
```
这行代码告诉 Three.js 渲染器的最终输出颜色应该被处理为 sRGB 色彩空间。Three.js 的 PBR 材质和加载器（如 `GLTFLoader`）通常会自动处理纹理和颜色的输入，确保它们在线性空间中参与计算。

### 2. 色调映射 (Tone Mapping)

真实世界的光照强度范围（动态范围）远超普通显示器所能展示的范围。例如，正午阳光下的亮度可能是室内灯光的几千甚至几万倍。HDR（高动态范围）图像可以捕捉这种宽广的亮度信息。

**色调映射**就是将场景中计算出来的高动态范围颜色值"压缩"或"映射"到显示器能够表现的低动态范围（LDR）的过程。好的色调映射算法能在保留场景亮部和暗部细节的同时，产生视觉上愉悦和自然的图像。

在代码中：
```javascript
renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.toneMappingExposure = 1.1;
```
*   `THREE.ACESFilmicToneMapping`: Academy Color Encoding System (ACES) 是一种电影工业常用的色彩管理系统，其 Filmic 色调映射能提供一种接近电影胶片质感的柔和过渡和漂亮的色彩表现，尤其在高光区域处理得很好。
*   `toneMappingExposure`: 控制最终图像的整体亮度，类似于相机中的曝光补偿。增加此值会使场景变亮。

## 三、环境设置：构筑光影世界

环境是决定渲染效果好坏的重中之重。它不仅包括光源，还包括天空和周围能反射光线的物体。

### 1. 基础场景组件

*   `THREE.Scene`: 容纳所有模型、光源、相机等对象的容器。
*   `THREE.PerspectiveCamera`: 模拟人眼或相机看到的透视效果。参数包括视野角度 (FOV)、宽高比 (aspect)、近裁剪面 (near) 和远裁剪面 (far)。
*   `THREE.WebGLRenderer`: 将场景渲染到 HTML canvas 上的"画笔"。

### 2. 光源 (Lighting)

场景中使用了两种主要光源：

*   **环境光 (AmbientLight)**:
    ```javascript
    ambientLight = new THREE.AmbientLight(0xffffff, 0.4);
    scene.add(ambientLight);
    ```
    环境光是一种无方向的光，它会均匀地照亮场景中的所有物体，无论其朝向如何。可以把它想象成光线在环境中多次反弹后形成的漫射背景光。
    *   `0xffffff`: 光的颜色（白色）。
    *   `0.4`: 光的强度。
    环境光本身不能产生阴影，它主要用于提升场景的整体亮度和减少纯黑区域，使暗部不至于死黑一片。

*   **平行光 (DirectionalLight)**:
    ```javascript
    directionalLight = new THREE.DirectionalLight(0xffffff, 1.2);
    directionalLight.position.set(10, 20, 10);
    directionalLight.castShadow = true;
    // ... shadow properties ...
    scene.add(directionalLight);
    ```
    平行光模拟来自非常遥远光源（如太阳）的光线，所有光线都是平行的。
    *   `position`: **非常重要**，平行光的方向是由其位置指向场景原点（或其目标对象）决定的。所以 `directionalLight.position.set(10, 20, 10)` 意味着光从 (10, 20, 10) 点照向原点。
    *   `castShadow = true`: 使此光源能够投射阴影。
    *   **阴影属性**:
        *   `directionalLight.shadow.mapSize.width = 2048;`
        *   `directionalLight.shadow.mapSize.height = 2048;`
            阴影贴图的分辨率。值越大，阴影边缘越清晰，但消耗的性能也越高。
        *   `directionalLight.shadow.camera.near = 0.5;`
        *   `directionalLight.shadow.camera.far = 50;`
            定义了光源"视锥体"的范围，只有在这个范围内的物体才会产生阴影。调整这个范围以紧密包裹场景中需要投射阴影的物体，可以提高阴影质量。
        *   `directionalLight.shadow.bias = -0.0005;`
            阴影偏移量。用于解决"阴影痤疮"（shadow acne，物体自身投射在自己表面的斑驳阴影）或"彼得潘现象"（peter panning，物体与阴影分离）的问题。这是一个需要细调的值。

### 3. 动态天空与太阳模拟 (Dynamic Sky & Sun)

代码使用 `THREE.Sky` 对象来程序化地生成一个动态天空背景，并且这个天空的状态（如浑浊度、太阳位置）会影响场景光照。

```javascript
// 初始化天空
sky = new Sky();
sky.scale.setScalar(450000); // 天空穹顶的大小
scene.add(sky);

sun = new THREE.Vector3(); // 用于存储太阳方向向量
pmremGenerator = new THREE.PMREMGenerator(renderer); // 后续用于生成环境贴图

// 天空着色器uniforms (参数)
const skyUniforms = sky.material.uniforms;
skyUniforms['turbidity'].value = 10;    // 浑浊度
skyUniforms['rayleigh'].value = 3;      // 瑞利散射系数
skyUniforms['mieCoefficient'].value = 0.005; // 米氏散射系数
skyUniforms['mieDirectionalG'].value = 0.7;  // 米氏散射方向性

updateSkyAndSun(); // 根据参数更新天空和太阳
```

*   **`THREE.Sky` 原理**:
    它内部是一个基于大气散射物理模型的着色器。主要模拟两种散射：
    *   **瑞利散射 (Rayleigh Scattering)**: 由尺寸远小于光波长的空气分子引起（如氮气、氧气）。这种散射对短波长的蓝光比长波长的红光更强烈，所以天空是蓝色的，日落是红色的。`skyUniforms['rayleigh']` 控制其强度。
    *   **米氏散射 (Mie Scattering)**: 由尺寸与光波长相当的较大颗粒物引起（如水蒸气、尘埃、烟雾）。这种散射对所有波长的光影响相近，使天空呈现灰白色，并形成太阳周围的光晕。`skyUniforms['turbidity']` (浑浊度)、`skyUniforms['mieCoefficient']` 和 `skyUniforms['mieDirectionalG']` 控制这部分效果。

*   **更新太阳位置 (`updateSkyAndSun` 函数)**:
    ```javascript
    const phi = THREE.MathUtils.degToRad(90 - elevation);
    const theta = THREE.MathUtils.degToRad(azimuth);
    sun.setFromSphericalCoords(1, phi, theta);
    sky.material.uniforms['sunPosition'].value.copy(sun);
    ```
    *   `elevation` (太阳高度角) 和 `azimuth` (太阳方位角) 是通过UI控制的。
    *   `THREE.MathUtils.degToRad()`: 将角度从度转换为弧度，因为三角函数通常使用弧度。
    *   **球坐标 (Spherical Coordinates)**: `sun.setFromSphericalCoords(radius, phi, theta)`
        *   `radius`: 到原点的距离（这里设为1，因为我们主要关心方向）。
        *   `phi`: "极角"或"倾斜角"，是从正Y轴（天顶）向下到太阳方向的夹角。代码中 `90 - elevation` 是因为 `elevation` 通常指从地平线向上的角度。
        *   `theta`: "方位角"，是从正Z轴（或X轴，取决于约定）在XZ平面上旋转到太阳方向投影的夹角。
    *   `sky.material.uniforms['sunPosition'].value.copy(sun)`: 将计算出的太阳方向向量传递给天空着色器。

*   **太阳与平行光的联动**:
    ```javascript
    directionalLight.position.copy(sun.clone().multiplyScalar(sunDistance));
    // ... 根据太阳高度调整光照颜色和强度 ...
    directionalLight.color = sunColor;
    ```
    平行光的位置（从而决定其方向）被设置为与天空着色器中的太阳方向一致。并且，根据太阳高度，平行光的颜色和强度也会动态调整，模拟不同时间段（如日出、正午、日落）的光照变化。

*   **`PMREMGenerator` 与天空的环境光**:
    ```javascript
    const renderTarget = pmremGenerator.fromScene(sky);
    scene.environment = renderTarget.texture;
    scene.background = sky; // 将天空本身设置为背景
    ```
    *   `pmremGenerator.fromScene(sky)`: 这行代码非常重要。`PMREMGenerator` (Pre-filtered Mipmapped Radiance Environment Map Generator) 会将当前天空的视觉效果（包括太阳的亮度和颜色）渲染到一个特殊处理过的纹理中（环境贴图）。
    *   这个纹理不仅包含了天空的图像，还预先计算了不同粗糙度级别下的反射光照信息，PBR材质可以直接使用它来进行高效的基于图像的照明 (Image-Based Lighting, IBL) 和反射。
    *   `scene.environment = renderTarget.texture;`: 将这个生成的天空环境贴图设置为场景的全局环境贴图。PBR材质会自动拾取这个贴图用于计算反射和间接光照。
    *   **这意味着，即使没有HDR文件，动态天空本身也能为场景提供基础的环境光照和反射源。**

### 4. HDR 环境贴图 (HDR Environment Mapping)

虽然动态天空能提供环境光，但高质量的HDR（高动态范围）图像通常能提供更丰富、更真实的光照细节和反射信息。

```javascript
const hdrEnvironmentURL = './kloofendal_48d_partly_cloudy_puresky_4k.hdr';
// ...
new RGBELoader()
    .setPath('')
    .load(hdrEnvironmentURL, (hdrEquirect) => {
        const envMapFromHDR = pmremGenerator.fromEquirectangular(hdrEquirect).texture;
        envMap = envMapFromHDR; // 全局变量，供材质使用
        hdrEquirect.dispose();
        console.log("HDR environment map loaded for reflections.");
        loadModel(); // HDR加载成功后加载模型
    }, null, (error) => {
        console.error('Failed to load HDR environment map:', error);
        envMap = scene.environment; // Fallback: 如果HDR加载失败，使用天空生成的环境贴图
        loadModel();
    });
```

*   **HDR 图像**: `.hdr` 文件通常存储的是高动态范围的图像，它能记录比普通图像（如JPG）更广阔的亮度级别。例如，太阳本身在HDR中可能具有非常高的亮度值，而阴影区域的亮度值则较低，这种对比在普通图像中会被裁剪掉。
*   **`RGBELoader`**: 用于加载 Radiance HDR (`.hdr`) 文件格式。
*   **Equirectangular (等距柱状投影) 格式**: HDR环境贴图常以这种全景图格式存储。
*   **`pmremGenerator.fromEquirectangular(hdrEquirect).texture`**: 这是关键步骤。`PMREMGenerator` 再次登场，这次它将2D的等距柱状投影HDR图像转换成GPU更喜欢使用的格式——通常是带有预过滤Mipmap的立方体贴图 (Cubemap)。
    *   **预过滤 (Pre-filtering)**: 对环境贴图进行不同程度的模糊处理，并存储在Mipmap层级中。当PBR材质的粗糙度较高时，它会采样模糊的环境贴图版本，得到柔和的反射；粗糙度低时，采样清晰的版本，得到锐利的反射。
    *   **`envMap = envMapFromHDR;`**: 将处理后的HDR纹理存储在全局变量 `envMap` 中。
*   **材质中的应用**:
    ```javascript
    // 在金属和玻璃材质中
    envMap: envMap || scene.environment, // 优先使用加载的HDR，否则用天空的
    ```
    PBR材质的 `envMap` 属性被设置为这个处理过的HDR纹理。这使得物体能够反射出HDR环境中的细节，并且场景光照也会受到这个HDR环境的影响（IBL）。
*   **注意**: 代码中，`scene.environment` 是先由 `sky` 生成的，然后如果HDR加载成功，`envMap` 变量会指向HDR纹理，并在材质中优先使用 `envMap`。如果HDR加载失败，材质会回退到使用 `scene.environment`（即天空生成的环境图）。这是一个很好的容错处理。

## 四、PBR 材质详解：赋予物体灵魂

现在我们来看看项目中定义的几种PBR材质。PBR材质的核心在于用一组直观且符合物理意义的参数来描述表面特性。

### 通用PBR属性

*   **`color` (颜色)**: 物体的基础颜色。对于非金属，这是光线散射出物体表面时呈现的颜色。对于金属，它更像是反射光线的色调。
*   **`metalness` (金属度)**: 0 到 1 的值。
    *   `0`: 非金属 (Dielectric)。光线一部分被表面反射（高光通常是白色的），一部分进入物体内部发生散射后再射出（这部分带有物体颜色）。
    *   `1`: 金属 (Metallic)。光线几乎完全在表面被反射，并且反射光会带上金属本身的颜色（例如金是黄色的反射，铜是红褐色的反射）。
    *   介于 0 和 1 之间的值可以模拟脏污的金属或带有金属薄层的非金属。
*   **`roughness` (粗糙度)**: 0 到 1 的值。描述表面的微观平整度。
    *   `0`: 完美光滑（如镜子、平静水面），反射非常清晰锐利。
    *   `1`: 非常粗糙（如粉笔、毛毡），反射非常模糊，接近漫反射。
*   **`envMap` (环境贴图)**: 指定一个纹理（通常是预处理过的Cubemap，如我们之前生成的 `envMapFromHDR` 或 `pmremGenerator.fromScene(sky).texture`）作为反射和环境光照的来源。**这是PBR材质产生真实反射效果的关键。**
*   **`envMapIntensity` (环境贴图强度)**: 控制环境贴图对材质影响的强度。

### 1. 金属窗框材质 (`metalFrameMaterial`)

```javascript
const metalFrameMaterial = new THREE.MeshPhysicalMaterial({
    color: 0x777777,       // 基础颜色，对于金属，通常较暗或中性，因为主要颜色来自反射
    metalness: 1.0,        // 纯金属
    roughness: 0.1,        // 较低的粗糙度，使其具有光泽和清晰反射 (之前是0.2，调整为0.1效果更好)
    envMap: envMap || scene.environment,
    envMapIntensity: 1.5   // 环境反射强度
});
```
*   **原理**: 金属的视觉特征主要来自于其对环境的强烈反射。
    *   `metalness: 1.0` 告诉渲染器按照金属的光学模型处理光线。
    *   `roughness: 0.1` (较低值) 意味着表面相对光滑，能够形成较为清晰的环境反射。如果这个值很高（比如0.8），金属看起来会像磨砂的，反射会非常模糊。
    *   `envMap` 提供了反射的内容。没有 `envMap`，金属看起来会很平淡，失去了金属光泽。
    *   `color: 0x777777` (灰色) 作为基础色，有色的环境光反射叠加上后会形成最终的金属色感。对于纯净金属，有时也会使用更亮的基色（如白色），让反射的颜色完全主导。

### 2. 玻璃幕墙材质 (`glassCurtainWallMaterial`)

```javascript
const glassCurtainWallMaterial = new THREE.MeshPhysicalMaterial({
    color: 0xadd8e6,        // 淡蓝色玻璃的颜色 (透射色)
    metalness: 0.0,         // 玻璃是非金属
    roughness: 0.05,        // 非常光滑的玻璃，产生清晰反射
    transmission: 0.9,      // 透射率，90%的光线可以穿透玻璃
    ior: 1.52,              // 折射率 (Index of Refraction)，普通玻璃的典型值
    envMap: envMap || scene.environment,
    envMapIntensity: 1.8,   // 环境反射强度 (之前是1.0，调高以增强反射)
    transparent: true,      // 必须开启，以启用透明效果和透射
    opacity: 0.85,          // 不透明度 (之前是0.8)。注意：transmission 和 opacity 会相互影响
    thickness: 0.1          // 玻璃的厚度，影响透射光的衰减和色散（如果启用了相关高级特性）
});
```
玻璃是PBR中比较复杂的材质之一，`MeshPhysicalMaterial` 提供了更多控制参数：

*   **`metalness: 0.0`**: 玻璃是非金属。
*   **`roughness: 0.05`**: 高度抛光的玻璃表面非常光滑，所以粗糙度很低。
*   **`transmission: 0.9`**: 这是实现"透光"效果的核心属性。它表示有多少比例的光线能够穿透材质。值越高，玻璃越透明。
*   **`ior: 1.52`**: 折射率。当光线从一种介质（如空气）进入另一种介质（如玻璃）时会发生弯曲，弯曲的程度由两种介质的折射率决定（基于斯涅尔定律 Snell's Law: \(n_1 \sin\theta_1 = n_2 \sin\theta_2\)）。`ior` 值越高，光线弯曲越明显，透过玻璃看到的物体变形也越大。1.52 是普通玻璃的常见折射率。水大约是1.33，钻石大约是2.42。
*   **`transparent: true`**: 为了让 `transmission` 生效，或者使用 `opacity` 实现传统alpha混合透明，这个属性需要设为 `true`。
*   **`opacity: 0.85`**: 这是传统的Alpha透明度。对于 `MeshPhysicalMaterial` 的透射效果，`opacity` 的影响比较微妙。当 `transmission` 启用时，`opacity` 更多地像一个整体的"可见性"因子。如果希望玻璃有轻微的磨砂感或者整体偏暗淡，可以调整 `opacity`。但主要的透明感应由 `transmission` 控制。
*   **`envMap` 和 `envMapIntensity`**: 玻璃表面也会反射环境，这是玻璃质感的重要组成部分。想象一下窗户上反射出的天空和周围景物。
*   **`thickness: 0.1`**: 对于具有 `transmission` 的材质，`thickness` 可以影响透射光的颜色衰减。如果玻璃有颜色（`color` 属性），光线穿过越厚的玻璃，颜色会显得越深。对于薄玻璃，这个值影响可能不显著，但对于有色厚玻璃或模拟液体等效果时很重要。

**让玻璃更"玻璃"的关键**：
1.  **高 `transmission` 值** (例如 `0.8` 到 `0.95`)。
2.  **低 `roughness` 值** (例如 `0.0` 到 `0.1`) 以获得清晰的反射和透射。
3.  **合适的 `ior` 值** (通常 `1.4` 到 `1.6` 之间是各种玻璃的范围)。
4.  **清晰且细节丰富的 `envMap`**，以及适当的 `envMapIntensity`。
5.  **`transparent: true`**。

### 3. 混凝土与石材墙体材质 (`concreteMaterial`, `stoneMaterial`)

```javascript
// 混凝土
const concreteMaterial = new THREE.MeshStandardMaterial({
    color: 0xd0d0d0,        // 浅灰色混凝土
    metalness: 0.1,         // 通常混凝土是非金属，这里0.1可能想表达非常轻微的金属杂质感或湿润感
    roughness: 0.8,         // 混凝土表面粗糙，反射模糊
    envMap: envMap || scene.environment,
    envMapIntensity: 0.3,
    // bumpScale: 0.005 // (配合bumpMap使用)
});

// 石材/花岗岩
const stoneMaterial = new THREE.MeshStandardMaterial({
    color: 0xb8b8b8,
    metalness: 0.05,        // 类似混凝土，极低的金属度
    roughness: 0.9,         // 石材通常更粗糙
    envMap: envMap || scene.environment,
    envMapIntensity: 0.2
});
```
*   **原理**: 墙体这类材质通常是非金属的、表面粗糙的。
    *   `metalness`: 通常为 `0`。代码中的 `0.1` 或 `0.05` 是一个非常低的值，可能是为了给材质增加一点点微弱的、不易察觉的光泽变化，或者模拟表面某些微小颗粒的反射特性。对于纯粹的混凝土或干燥石材，设为 `0` 更符合物理。
    *   `roughness`: 较高的值 (如 `0.8`, `0.9`)。这使得光线在表面发生广泛的漫反射，形成哑光的外观，反射的环境影像会非常模糊，几乎看不清具体内容，但仍能贡献整体的颜色和亮度。
    *   `envMapIntensity`: 即使是粗糙表面，环境光照依然重要，它能提供间接光照，使得暗部不死黑，并影响物体整体色调。但强度通常比金属或玻璃低。
*   **提升真实感**: 对于这类材质，除了基础的PBR参数，使用纹理贴图是提升真实感的关键：
    *   **`map` (颜色贴图/漫反射贴图)**: 提供表面详细的颜色和图案。
    *   **`normalMap` (法线贴图)**: 模拟表面细微的凹凸细节（如砖缝、石材质地），极大地增强立体感，而无需增加模型的多边形数量。法线贴图存储的是表面法线方向的扰动信息。
    *   **`roughnessMap` (粗糙度贴图)**: 允许表面不同区域有不同的粗糙度。例如，混凝土上湿润的部分可能比干燥的部分更光滑。
    *   **`metalnessMap` (金属度贴图)**: 如果表面有金属和非金属混合的部分（如生锈的铁皮）。
    *   **`aoMap` (环境光遮蔽贴图)**: 预计算的阴影信息，用于表现缝隙、角落等难以接收到直接光照的区域，增加深度感。
    代码中提到了 `bumpScale`，它通常与 `bumpMap` (凹凸贴图) 配合使用。凹凸贴图是一种简单的灰度图，通过模拟表面高度变化来改变法线，也能增加细节，但效果通常不如法线贴图精确。

## 五、模型集成与后续调整

### 1. GLTF 模型加载与材质应用

```javascript
loader.load(modelUrl, function(gltf) {
    model = gltf.scene;
    scene.add(model);
    
    model.traverse((child) => { // 遍历模型的所有子对象
        if (child.isMesh) {     // 如果是网格模型
            child.castShadow = true;
            child.receiveShadow = true;
            console.log('Found mesh in GLB:', child.name);

            // 根据网格名称应用材质
            if (child.name === "窗框") {
                child.material = metalFrameMaterial;
            } else if (child.name === "墙体_1") {
                child.material = concreteMaterial;
            } // ... 其他材质分配
        }
    });
    // ... 模型居中和缩放 ...
});
```
*   `GLTFLoader`: Three.js 推荐的现代三维模型格式加载器。
*   `model.traverse()`: 这是一个非常实用的方法，可以递归地访问模型及其所有子节点。通过检查 `child.isMesh` 和 `child.name`，您可以为导入模型中的特定部分指定在代码中创建的自定义材质。这是替换或增强glTF内部材质的常用方法。
*   **模型居中与缩放**: 代码中通过 `THREE.Box3` 计算模型的边界框，然后将其平移到场景中心并缩放到合适的大小，这对于保证模型在场景中正确显示非常重要。

### 2. 调试与优化建议

*   **控制台日志**: 您在代码中加入了很多 `console.log`，这对于调试模型加载、材质应用等非常有帮助。
*   **Dat.GUI 或类似UI**: 对于PBR材质参数的调整，使用一个实时调试UI（如 Dat.GUI）会非常高效。您可以直接在浏览器中拖动滑块改变参数值，立即看到效果，而无需反复修改代码和刷新页面。项目已经实现了一个控制面板，这是非常棒的实践！
*   **检查HDR加载**: 确保HDR环境贴图 (`kloofendal_48d_partly_cloudy_puresky_4k.hdr`) 能够被正确加载。如果网络问题或路径问题导致加载失败，材质的反射效果会大打折扣（虽然有天空作为后备，但HDR细节更丰富）。注意浏览器控制台中的网络请求和错误信息。
*   **光照强度**: 环境光、平行光的强度，以及 `toneMappingExposure`，这些都会显著影响材质的最终观感。金属和玻璃对光线变化尤为敏感。
*   **循序渐进**: 调整材质时，可以先从一个基本的光照环境开始，单独调整一种材质，观察其在不同视角下的变化。例如，先确保金属的反射清晰，再调整玻璃的透明度和反射，最后是墙体的质感。

希望这篇详细的解析能帮助您更深入地理解项目中的渲染技术和PBR材质原理，并为您的后续创作提供启发！
