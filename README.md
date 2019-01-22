# tinyraytracer-CN


# 256行C++理解RayTracing


---

## 前言
#### 英文原文：https://github.com/ssloy/tinyraytracer/wiki#step-4-lighting
#### 代码地址：https://github.com/ssloy/tinyraytracer
#### 译者：松爱家的小秦

这是我的计算机图形学讲座的另一章。 这次我们谈论光线追踪。 像往常一样，我试图避开第三方库，因为我相信这会让学生了解一下底层发生了什么。

网上有很多光线追踪文章; 然而问题是，几乎所有这些基于已经实现的完整软件上（很难单独拉出来说明里面的机制）。 举例来说，非常着名的businness card ray tracer challenge。 它产生非常令人印象深刻的程序，但很难理解它是如何工作的。 我想告诉你如何自己实现。

注意：仅仅查看我的代码是没有意义的，也不仅仅是用一杯茶来阅读这篇文章。 本文旨在让您使用键盘并实现自己的渲染引擎。 它肯定会比我的好。 

所以，今天的目标是学习如何渲染这样的图像：

![cmd-markdown-logo](https://raw.githubusercontent.com/ssloy/tinyraytracer/homework_assignment/out-envmap-duck.jpg)

## 第1步：将图片写入磁盘

我不想动用窗口管理器，鼠标/键盘处理等等。 我们程序的目标将是在磁盘上保存简单图片。 因此，我们需要做的第一件事就是将图片保存到磁盘。 在这里，您可以找到允许我们执行此操作的代码。 让我列出主文件：

```c++

#include <limits>
#include <cmath>
#include <iostream>
#include <fstream>
#include <vector>
#include "geometry.h"

void render() {
    const int width    = 1024;
    const int height   = 768;
    std::vector<Vec3f> framebuffer(width*height);

    for (size_t j = 0; j<height; j++) {
        for (size_t i = 0; i<width; i++) {
            framebuffer[i+j*width] = Vec3f(j/float(height),i/float(width), 0);
        }
    }

    std::ofstream ofs; // save the framebuffer to file
    ofs.open("./out.ppm");
    ofs << "P6\n" << width << " " << height << "\n255\n";
    for (size_t i = 0; i < height*width; ++i) {
        for (size_t j = 0; j<3; j++) {
            ofs << (char)(255 * std::max(0.f, std::min(1.f, framebuffer[i][j])));
        }
    }
    ofs.close();
}

int main() {
    render();
    return 0;
}

```


只在主函数中调用render（）而没有其他内容。 render（）函数内部是什么？首先，我将帧缓冲定义为Vec3f值的一维数组，这些是简单的三维向量，为每个像素提供（r，g，b）值。向量类存在于文件geometry.h中，我不会在这里描述它：它实际上是对二维和三维向量的一个简单操作（加法，减法，赋值，乘以标量，标量乘积）。

我以ppm格式保存图像。这是保存图像的最简单方法，但并不总是最方便的方式来进一步查看图像。如果您想以其他格式保存，我建议您链接第三方库，例如stb。这是一个很棒的库：您只需要在项目中包含一个头文件stb_image_write.h，它将允许您以最流行的格式保存图像。

因此，这一步的目标是确保我们能够做到
a）在内存中创建图像+分配不同的颜色
b）将结果保存到磁盘。
然后，您可以在第三方软件中查看它。结果如下：

![cmd-markdown-logo](https://raw.githubusercontent.com/ssloy/tinyraytracer/bd36c9857305b3cbd06f5b768bb48a92df9ae68b/out.jpg)





## 第2步，关键的一步：光线追踪

这是整个步骤中最重要和最困难的一步。 我想在我的代码中定义一个球体并绘制它的材料或光照。 这就是我们的结果应该是这样的：

![cmd-markdown-logo](https://raw.githubusercontent.com/ssloy/tinyraytracer/5806eb45e93dab225ab335824cbc3f537d511b28/out.jpg)

为方便起见，我的存储库中每个步骤都有一个提交; Github可以很容易地查看所做的更改。 例如，在这里，第二次提交改变了什么。

首先，如何在计算机内存中代表球体？ 四个数字就足够了：球体中心的三维向量和描述半径的标量：

```c++

struct Sphere {
    Vec3f center;
    float radius;

    Sphere(const Vec3f &c, const float &r) : center(c), radius(r) {}

    bool ray_intersect(const Vec3f &orig, const Vec3f &dir, float &t0) const {
        Vec3f L = center - orig;
        float tca = L*dir;
        float d2 = L*L - tca*tca;
        if (d2 > radius*radius) return false;
        float thc = sqrtf(radius*radius - d2);
        t0       = tca - thc;
        float t1 = tca + thc;
        if (t0 < 0) t0 = t1;
        if (t0 < 0) return false;
        return true;
    }
};
```

 有关射线球交叉算法的详细说明可以在这里找到，我强烈建议你这样做并检查我的代码。
 
 光线跟踪如何工作？ 这很简单。 在第一步，我们用渐变的颜色填充图片：
 
```c++

     for (size_t j = 0; j<height; j++) {
        for (size_t i = 0; i<width; i++) {
            framebuffer[i+j*width] = Vec3f(j/float(height),i/float(width), 0);
        }
    }
``` 


现在，对于每个像素，我们将形成来自原点并穿过像素的光线，然后检查此光线是否与球体相交：

![cmd-markdown-logo](https://camo.githubusercontent.com/ccc20911601b22524a1cc127ea89dfe2f1c100d9/68747470733a2f2f75706c6f61642e77696b696d656469612e6f72672f77696b6970656469612f636f6d6d6f6e732f382f38332f5261795f74726163655f6469616772616d2e7376673f73616e6974697a653d74727565)


如果没有与球体的交点，我们用color1绘制像素，否则用color2绘制：

```c++
Vec3f cast_ray(const Vec3f &orig, const Vec3f &dir, const Sphere &sphere) {
    float sphere_dist = std::numeric_limits<float>::max();
    if (!sphere.ray_intersect(orig, dir, sphere_dist)) {
        return Vec3f(0.2, 0.7, 0.8); // background color
    }
    return Vec3f(0.4, 0.4, 0.3);
}

void render(const Sphere &sphere) {
￼	[...]
    for (size_t j = 0; j<height; j++) {
        for (size_t i = 0; i<width; i++) {
            float x =  (2*(i + 0.5)/(float)width  - 1)*tan(fov/2.)*width/(float)height;
            float y = -(2*(j + 0.5)/(float)height - 1)*tan(fov/2.);
            Vec3f dir = Vec3f(x, y, -1).normalize();
            framebuffer[i+j*width] = cast_ray(Vec3f(0,0,0), dir, sphere);
        }
    }
￼	[...]
}
``` 



在这一点上，我建议你拿一支铅笔并在纸上检查所有的计算（光线球体交叉点和用光线扫描图片）。以防万一，我们的相机由以下参数决定：

> * 图片宽度
> * 图片高度
> * 视角的视野
> * 摄像机位置，Vec3f（0.0.0）

沿着z轴，在负无穷大的方向上查看方向
让我举例说明我们如何计算要跟踪的光线的初始方向。在主循环中我们有这个公式：

```c++
            float x =  (2*(i + 0.5)/(float)width  - 1)*tan(fov/2.)*width/(float)height;
            float y = -(2*(j + 0.5)/(float)height - 1)*tan(fov/2.);
``` 


![cmd-markdown-logo](https://raw.githubusercontent.com/ssloy/tinyraytracer/master/trace.png)


## 第3步：添加更多球体


最难的部分结束了，现在我们的道路很清楚。 如果我们知道如何绘制一个球体，那么添加更多球体将不会花费太多时间。 检查代码中的更改，这是生成的图像：

![cmd-markdown-logo](https://raw.githubusercontent.com/ssloy/tinyraytracer/c19c430151cb659372b4988876173b022164e371/out.jpg)



## 第4步：光线

除了缺乏光线外，图像在各方面都是完美的。在本文的其余部分，我们将讨论照明。让我们添加几个点光源：


```c++
struct Light {
    Light(const Vec3f &p, const float &i) : position(p), intensity(i) {}
    Vec3f position;
    float intensity;
};
``` 

计算真实的全局照明是一项非常非常困难的任务，因此与其他人一样，我们将通过绘制完全非物理但视觉上合理的结果来欺骗眼睛。首先：为什么冬天很冷，夏天很热？因为地球表面的加热取决于太阳光线的入射角。太阳越高于地平线越高，表面越亮。相反，它越低于地平线，它就越暗。在太阳落在地平线之后，光子根本不会到达我们。

回到我们的球体：我们从相机发出一条光线（与光子无关！），它停在一个球体上。我们如何知道交点照明的强度？事实上，检查此点的法向量与描述光的方向的矢量之间的角度就足够了。角度越小，表面照射越好。回想一下，两个向量a和b之间的标量乘积等于向量范数乘以向量之间角度的余弦的乘积：a * b = | a | | B | COS（α（A，B））。如果我们采用单位长度的矢量，点积将给出我们表面照明的强度。

因此，在cast_ray函数中，我们将返回考虑光源的向量颜色而不是常量颜色：

```c++
Vec3f cast_ray(const Vec3f &orig, const Vec3f &dir, const Sphere &sphere) {
    [...]
    float diffuse_light_intensity = 0;
    for (size_t i=0; i<lights.size(); i++) {
        Vec3f light_dir      = (lights[i].position - point).normalize();
        diffuse_light_intensity  += lights[i].intensity * std::max(0.f, light_dir*N);
    }
    return material.diffuse_color * diffuse_light_intensity;
}
``` 
![cmd-markdown-logo](https://raw.githubusercontent.com/ssloy/tinyraytracer/9a728fff2bbebb1eedd86e1ac89f657d43191609/out.jpg)

## 第5步：高光

点积技巧很好地近似了无光泽表面的光照，在文献中称为漫射照明。 如果我们想要绘制闪亮的表面，我们该怎么办？ 我想得到这样的照片：
检查需要修改的次数。 简而言之，光亮表面上的光越亮，视角方向与反射光方向之间的角度越小。

这种暗淡和光泽表面照明的技巧被称为Phong反射模型。 维基对此照明模型有相当详细的描述。

![cmd-markdown-logo](https://raw.githubusercontent.com/ssloy/tinyraytracer/f5ec45c2541feb86b6a30cc3bb04917d60d13e9b/out.jpg)

![cmd-markdown-logo](https://camo.githubusercontent.com/b440e7c18ecdf9c75cb00fbfb67eafce28e625d8/68747470733a2f2f75706c6f61642e77696b696d656469612e6f72672f77696b6970656469612f636f6d6d6f6e732f362f36622f50686f6e675f636f6d706f6e656e74735f76657273696f6e5f342e706e67)


## 第6步：阴影

为什么我们有光，但没有阴影？ 这不好！ 我想要这张照片：
仅仅六行代码就可以实现这一点：绘制每个点时，我们只需确保当前点和光源之间的段不与场景的对象相交。 如果有交叉点，我们跳过当前光源。 只有一个小小的微妙之处：我通过向正常方向移动来扰乱该点：

![cmd-markdown-logo](https://raw.githubusercontent.com/ssloy/tinyraytracer/ef70d1356169dacb3183ad4fcb4c23f1d7003e1b/out.jpg)

```c++
Vec3f shadow_orig = light_dir*N < 0 ? point - N*1e-3 : point + N*1e-3;
``` 

## 第7步：反射

这是令人难以置信的，但要为渲染添加反射，我们只需要添加三行代码：

亲眼看看：当与球体相交时，我们只计算反射光线（借助我们用于镜面高光的相同函数！）并在反射光线的方向上递归调用cast_ray函数。 一定要使用递归深度，我将其设置为4，尝试从0开始标注不同的值，图片中会有什么变化？ 这是我的反射结果和递归深度为4：

![cmd-markdown-logo](https://raw.githubusercontent.com/ssloy/tinyraytracer/c80479d1d22fe98f41b584972affeb43422a23a6/out.jpg)


## 第8步：折射

如果我们知道做反射，折射很容易。 我们需要添加一个函数来计算折射光线（使用Snell定律），并在我们的递归函数cast_ray中添加另外三行代码。 这是最接近的球“由玻璃制成”的结果，它同时反射和折射光线：

![cmd-markdown-logo](https://raw.githubusercontent.com/ssloy/tinyraytracer/b69793bf6e8be54973cad1b18185a67dbf11bad1/out.jpg)

## 结论

我的主要目标是展示有趣（而且简单！）的项目。 我相信，要成为一名优秀的程序员，必须做很多辅助项目。 我不了解你，但我个人并不喜欢会计软件和扫雷游戏，即使代码的复杂程度相当。 几个小时和250行代码为我们提供了光线跟踪器。 几天就可以完成500行软件光栅化器。 学习编程非常酷！


## 关于译者

Github:https://github.com/Qinxianshen

CSDN: https://blog.csdn.net/Qin_xian_shen

个人博客: http://saijiadexiaoqin.cn/

Gitchat:https://gitbook.cn/gitchat/author/59ef0b02a276fd1a69094634

哔哩哔哩：https://space.bilibili.com/126021651/#/

微信公众号：松爱家的小秦

更多LIVE：

[如何利用 Selenium 爬取评论数据？](https://gitbook.cn/gitchat/activity/59ef0fbf54011222e227c720)

[Neo4j 图数据库在社交网络等领域的应用](https://gitbook.cn/gitchat/activity/5a310961259a166307ceadb4)

[如何快速编写小程序商城和安卓 APP 商城](https://gitbook.cn/gitchat/activity/5b628776ff984e633d987f7d)
