.. _sec-file-format:

场景文件格式
=================

Mitsuba 使用简单且通用的 XML 格式来表示场景。我们框架的设计理念是表示离散的功能模块为插件，
由此，一个场景文件可以被认为是关于实例化哪些插件以及这些插件之间如何组合的“配方”。接下来，我
们将通过几个示例来了解场景文件格式的作用。

具有一个网格物体，默认照明和摄像机设置的简单场景如下所示：

.. code-block:: xml

    <scene version="2.0.0">
        <shape type="obj">
            <string name="filename" value="dragon.obj"/>
        </shape>
    </scene>

第一行的 ``version`` 属性表示用来创建场景的 Mitsuba 版本。该信息允许 Mitsuba 能够正确处理该
文件，不用考虑未来场景描述语言的潜在任何变化。

这个例子已经包含了很多关于场景格式方面需要了解的重要信息：包括 *渲染对象* （比如，那些由 ``<scene>`` 或
 ``<shape>`` 标签实例化的对象）对象间可以相互嵌套。每个对象都可以选择表征其特性的 *属性值* （比如，``<string>`` 标签）。
除了根对象（ ``<scene>`` ）之外的所有对象都会引发渲染器从磁盘中搜索和加载相应插件，因此你必须通过 ``type=".."`` 参数提供
插件名称。

对象标签还可以让渲染器知道要实例化的对象是 *什么样* 的类型：例如，任何那些使用 ``<shape>`` 标签加载的插件
都必须符合 *Shape* 接口，在上述的场景示例中指的就是名为 ``obj`` 的插件（它包含了一个 :ref:`Wavefront OBJ 加载器 <shape-obj>` ）。
类似的，你可以编写：

.. code-block:: xml

    <scene version="2.0.0">
        <shape type="sphere">
            <float name="radius" value="10"/>
        </shape>
    </scene>

这个例子加载了一个不同的插件( ``sphere`` )，但该插件仍然是 *Shape* ，表示的是10个世界单位的
:ref:`sphere <shape-sphere>` 。Mitsuba 随附了大量的插件，有关这些插件的详细概览，请参阅下
一章节。

最常见的场景设置说明是声明一个集合体、一些几何物体、一个传感器（例如摄像机）、一个胶片、一个采样器
和一个或多个光源。这里提供了一个稍微复杂一点的例子：

.. code-block:: xml

    <scene version="2.0.0">
        <integrator type="path">
            <!-- Instantiate a path tracer with a max. path length of 8 -->
            <integer name="max_depth" value="8"/>
        </integrator>

        <!-- Instantiate a perspective camera with 45 degrees field of view -->
        <sensor type="perspective">
            <!-- Rotate the camera around the Y axis by 180 degrees -->
            <transform name="to_world">
                <rotate y="1" angle="180"/>
            </transform>
            <float name="fov" value="45"/>

            <!-- Render with 32 samples per pixel using a basic
                 independent sampling strategy -->
            <sampler type="independent">
                <integer name="sample_count" value="32"/>
            </sampler>

            <!-- Generate an EXR image at HD resolution -->
            <film type="hdrfilm">
                <integer name="width" value="1920"/>
                <integer name="height" value="1080"/>
            </film>
        </sensor>

        <!-- Add a dragon mesh made of rough glass (stored as OBJ file) -->
        <shape type="obj">
            <string name="filename" value="dragon.obj"/>

            <bsdf type="roughdielectric">
                <!-- Tweak the roughness parameter of the material -->
                <float name="alpha" value="0.01"/>
            </bsdf>
        </shape>

        <!-- Add another mesh, this time, stored using Mitsuba's own
             (compact) binary representation -->
        <shape type="serialized">
            <string name="filename" value="lightsource.serialized"/>
            <transform name="toWorld">
                <translate x="5" y="-3" z="1"/>
            </transform>

            <!-- This mesh is an area emitter -->
            <emitter type="area">
                <rgb name="radiance" value="100,400,100"/>
            </emitter>
        </shape>
    </scene>

这个例子引入了几个新的对象类型 (``integrator``, ``sensor``, ``bsdf``, ``sampler``, ``film``, 和``emitter``) 
以及属性类型 (``integer``, ``transform``, 和 ``rgb``)。正如你在示例中看到的，对象声明通常发生在顶层，除非存在链接到
另一对象的某种继承关系。例如，BSDFs 通常特定于某个几何物体，它应该表现为 shape 的子类型。同样，sampler 和 film 会影响
sensor 产生光线的方式，以及其记录样本产生的辐射度的方式。下列表格概述了可用的对象类型：

.. figtable::
    :label: table-xml-objects
    :caption: 此表列出了不同类型的对象及其各自的标签。它还为每个类别提供了可用的示例插件。

    .. list-table::
        :widths: 17 53 30
        :header-rows: 1

        * - XML tag
          - Description
          - `type` examples
        * - `bsdf`
          - BSDF describe the manner in which light interacts with surfaces in the scene (i.e., the *material*)
          - `diffuse`, `conductor`
        * - `emitter`
          - Emitter plugins specify light sources and their characteristic emission profiles.
          - `constant`, `envmap`, `point`
        * - `film`
          - Film plugins convert measurements into the final output file that is written to disk
          - `hdrfilm`
        * - `integrator`
          - Integrators implement rendering techniques for solving the light transport equation
          - `path`, `direct`, `depth`
        * - `rfilter`
          - Reconstruction filters control how the `film` converts a set of samples into the output image
          - `box`, `gaussian`
        * - `sampler`
          - Sample generator plugins used by the `integrator`
          - `independent`
        * - `sensor`
          - Sensor plugins like cameras are responsible for measuring radiance
          - `perspective`, `orthogonal`
        * - `shape`
          - Shape puglins define surfaces that mark transitions between different types of materials
          - `obj`, `ply`, `serialized`
        * - `texture`
          - Texture plugins represent spatially varying signals on surfaces
          - `bitmap`


Properties
----------

本小节记录了将属性提供给对象的所有方式。如果你更想了解某个插件能接受的属性有哪些，
可以参阅 :ref:`plugin documentation <sec-plugins>`。

Numbers
*******

整数值和浮点值可以按如下方式传递：

.. code-block:: xml

    <integer name="int_property" value="1234"/>
    <float name="float_property" value="-1.5e3"/>

请注意，必须按照对象所期望的格式传递，即你不能将整数属性传递给需要与该名称关联的浮点属性的对象。

Booleans
********

布尔值可以按照如下方式进行传递：

.. code-block:: xml

    <boolean name="bool_property" value="true"/>

Strings
*******

传递字符串同样也很简单：

.. code-block:: xml

    <string name="string_property" value="This is a string"/>

Vectors, Positions
******************

点和向量可以按照如下方式进行指定：

.. code-block:: xml

    <point name="point_property" value="3, 4, 5"/>
    <vector name="vector_property" value="3, 4, 5"/>

.. note::

    Mitsuba 没有规定具体的距离度量单位（米、厘米、英寸等）。你需要考虑的唯一一个要求就是在
    整个场景中始终遵循一个约定即可。

RGB Colors
**********

在 Mitsuba 中颜色空间表示要么使用 ``<rgb>`` 标签，要么使用 ``<spectrum>`` 标签指定。
RGB 颜色的释出可以如下：

.. code-block:: xml

    <rgb name="color_property" value="0.2, 0.8, 0.4"/>

取决于渲染器当前激活的变体模式是哪一个。 例如， ``scalar_rgb`` 简单的将颜色转发到底层插件中，
不做任何修改。相比之下，``scalar_spectral`` 在 RGB 值没有意义的光谱域中运作---糟糕的是，每种
RGB 颜色都对应了一个无限大的光谱集合。Mitsuba 使用了 Jakob 和 Hanika 方法 :cite:`Jakob2019Spectral` 
在所有可能选项中选择一个较合理的平滑光谱。下面是一个例子：

.. image:: ../../../resources/data/docs/images/variants/upsampling.jpg
  :width: 100%
  :align: center


Color spectra
*************

一个指定特定颜色值的更准确方式是使用 ``<spectrum>`` 标签，以 *纳米级* 为单位记录了多离散波长的反射/强度值。

.. code-block:: xml

    <spectrum name="color_property" value="400:0.56, 500:0.18, 600:0.58, 700:0.24"/>

产生的光谱是在波长间进行线性插值得到的，并规定在指定波长范围之外时等于0。下面精简的创建了一个跨波长均匀的光谱：

.. code-block:: xml

    <spectrum name="color_property" value="0.5"/>

当光谱功率或反射率分布是通过测量获得的时（例如，以10纳米为间隔），通常会很难处理，
并且可能会使场景描述文件变得杂乱。因此我们还提供了一种从外部文件中加载光谱的方式：

.. code-block:: xml

    <spectrum name="color_property" filename="measured_spectrum.spd"/>

该文件每行应包含一次测量，相应的波长以纳米为单位，测量值以空格分隔。在文件中进行注释也是允许的。
下面提供了一个示例：

.. code-block:: text

    # This file contains a measured spectral power/reflectance distribution
    406.13 0.703313
    413.88 0.744563
    422.03 0.791625
    430.62 0.822125
    435.09 0.834000
    ...

有关 Mitsuba 2 光谱信息的详细信息，请参阅文档中插件那一章的 :ref:`相应小节 <sec-spectra>` 。
For more details regarding spectral information in Mitsuba 2, please have a look
at the :ref:`corresponding section <sec-spectra>` in the plugin documentation.

Transformations
***************

变换是唯一一个需要多个标签的属性。其想法是，从标识开始，可以使用一系列指令完成一个变换过程。
例如，一个包含平移和旋转的变换可以写成这样：

.. code-block:: xml

    <transform name="trafo_property">
        <translate value="-1, 3, 4"/>
        <rotate y="1" angle="45"/>
    </transform>


Mathematically, each incremental transformation in the sequence is
left-multiplied onto the current one. The following choices are available:

* 平移:

  .. code-block:: xml

      <translate value="-1, 3, 4"/>

* 绕指定轴逆时针旋转，角度以度为单位给出：

  .. code-block:: xml

      <rotate value="0.701, 0.701, 0" angle="180"/>

* 放缩。 可以通过传递负值得到翻转：

  .. code-block:: xml

      <scale value="5"/>        <!-- uniform scale -->
      <scale value="2, 1, -1"/> <!-- non-uniform scale -->

* 按照行主优先顺序显式声明一个 4x4 矩阵：

  .. code-block:: xml

      <matrix value="0 -0.53 0 -1.79 0.92 0 0 8.03 0 0 0.53 0 0 0 0 1"/>

* 按照行主优先顺序显式声明一个 3x3 矩阵。在渲染系统内部，它将被转化为 4x4 方阵，其最后一行、最后一列与单位矩阵相同。

  .. code-block:: xml

      <matrix value="0.57 0.2 0 0.1 -1 0 0 0 1"/>


* `lookat` 变换 -- 主要用于摄像机设置.  `origin` 坐标指定了摄像机原点， `target` 指向摄像机看的方向，  `up` (optional)参数决定了最终渲染图的 *向上方向*。

  .. code-block:: xml

      <lookat origin="10, 50, -800" target="0, 0, 0" up="0, 1, 0"/>

References
----------

通常，你会发现自己在许多地方会重复使用同一个的对象（例如材质等）。为了避免一次又一次的浪费内存的声明，
你可以通过使用引用来进行改善。下面是一个例子，说明了它是如何工作的：

.. code-block:: xml

    <scene version="2.0.0">
        <texture type="bitmap" id="my_image">
            <string name="filename" value="textures/my_image.jpg"/>
        </texture>

        <bsdf type="diffuse" id="my_material">
            <!-- Reference the texture named my_image and pass it
                 to the BRDF as the reflectance parameter -->
            <ref name="reflectance" id="my_image"/>
        </bsdf>

        <shape type="obj">
            <string name="filename" value="meshes/my_shape.obj"/>

            <!-- Reference the material named my_material -->
            <ref id="my_material"/>
        </shape>
    </scene>

通过在对象声明中提供唯一的标识符 `id` 值，对象就可以在实例化时绑定到该标识符了。在随后
引用此标识符（通过使用 ``<ref id=".."/>`` 标签）就可以添加该实例到父对象中了。

.. note::

    请注意，此功能旨在高效地处理在多个位置引用材质、纹理和分割介质，所以它不是能用于实例化几何体的。
    (实例化几何体可以使用 `instance` 插件， 但这还不是 Mitsuba 2 的一部分，不过在将来会添加上)

.. _sec-scene-file-format-params:

Default parameters
------------------

场景中可以包含通过命令行提供的命名参数：

.. code-block:: xml

    <bsdf type="diffuse">
        <rgb name="reflectance" value="$reflectance"/>
    </bsdf>

在该示例中，如果加载场景时没有从命令行按照 ``-Dreflectance=...`` 格式显式提供参数，则会报错。
为方便起见，可以在未给出命令行参数时优先使用指定的默认参数值。 其语法为：

.. code-block:: xml

    <default name="reflectance" value="something"/>

其必须位于 XML 文件中参数加载之前。

Including external files
------------------------

一个场景可以被分割成多个片段，以增强可读性。 如需包含外部文件，请使用以下命令：
A scene can be split into multiple pieces for better readability. To include an
external file, please use the following command:

.. code-block:: xml

    <include filename="nested-scene.xml"/>

在该示例中，文件 ``nested-scene.xml`` 必须是一个根部带有 ``<scene>`` 标签的正确场景文件。

此功能与 ``mitsuba`` 命令行渲染器的 ``-D key=value`` 标志配合使用通常会非常方便。这样就可
以通过更改命令行参数来包含场景配置的不同变体，而无需接触 XML 文件：

.. code-block:: xml

    <include filename="nested-scene-$version.xml"/>

Aliases
-------

有时将一个对象与多个标识关联会非常有用。这可以使用 ``alias as=".."`` 标签来完成：

.. code-block:: xml

    <bsdf type="diffuse" id="my_material_1"/>
    <alias id="my_material_1" as="my_material_2"/>

该语句将漫反射模型绑定到了 ``my_material_1`` 和 ``my_material_2`` 两个标识符上。

External resource folders
-------------------------

使用 ``path`` 标签，可以往搜索路径中添加新的路径。这会非常有用，尤其是当某些网格和纹理存储在不同的目录中时（例如，当与其他场景共享这些资源时）。
如果路径是相对路径，Mitsuba 2首先尝试在场景文件目录中进行搜索，随后查找搜索路径列表上的其他路径（在命令行中使用 ``-a <path1>;<path2>;..`` 添加路径）。

.. code-block:: xml

    <path value="../../my_resources"/>