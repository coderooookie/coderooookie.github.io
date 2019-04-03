---
title: Ray Tracing in the next week - Solid Txtures
date: 2019-4-3 12:36
categories:
- 游戏开发
- Ray Tracing
---
上一篇内容讲了层次包围盒，试了一下发现效果好像提升不明显。
这一篇讲纹理相关内容，相对来说这章内容较简单，因为前后一章都会比较难。。。
纹理对于图像来说就是决定了一个表面的颜色。前面内容中对材料初始化用到的内容是一个颜色向量，颜色也就是构成纹理的内容。
先实现一个纹理的类。
```c++
#include "vec3.hpp"

class texture {
public:
    virtual vec3 value(float u, float v, const vec3&p) const = 0;
};

class constant_texture : public texture {
public:
    constant_texture() {}
    constant_texture(vec3 c) : color(c) {}
    virtual vec3 value(float u, float v, const vec3& p) const {return color;}
    vec3 color;
};

class checker_texture : public texture {
public:
    checker_texture() {}
    checker_texture(texture* t0, texture* t1):even(t0), odd(t1){}
    virtual vec3 value(float u, float v, const vec3& p) const {
        float sines = sin(10*p.x()) * sin(10*p.x()) * sin(10*p.x());
        if (sines < 0)
            return odd->value(u, v, p);
        else
            return even->value(u, v, p);
    }

    texture* odd;
    texture* even;
};
```
其中`checker_texture`是根据射线坐标来实现交替出现的纹理。
我们可以重新生成一个简单的场景来测试这个纹理。
```c++
hitable* two_spheres(){
    texture* checker = new checker_texture(new constant_texture(vec3(0.3,0.2,0.1)), new constant_texture(vec3(0.9,0.9,0.9)));
    int n = 50;
    hitable **list = new hitable*[n+1];
    list[0] = new sphere(vec3(0, -10, 0), 10, new lambertian(checker));
    list[1] = new sphere(vec3(0, 10, 0), 10, new lambertian(checker));
    return new bvh_node(list, 2, 0, 1);
}
```
之后运行代码，将场景替换为上述的`two_spheres`，可以得到最终效果。
