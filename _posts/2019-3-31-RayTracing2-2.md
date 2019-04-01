---
title: Ray Tracing in the next week - Bouding Volume Hierarchies
date: 2019-3-31 10:59
categories:
- 游戏开发
- Ray Tracing
---
最近在看Ray Tracing 系列，打算继续写博客来记录相关内容。

第二章是关于层次包围盒的内容。
作者认为这部分内容是最难的也是最重要的一部分内容，而这部分能让ray tracing更快，因此也提前放在第二章来讲，这部分也重构了hitable类，为后续rectangle和box做准备。
    
光线碰撞的检测在ray tracer中是最耗时的瓶颈，时间也跟场景物体的个数呈线性相关。然而同一个模型会被重复的判断，因此可以用二叉查找的思想来进行对数级的优化。
由于我们对同一个模型会发射数以万计的射线，因此我们可以先进行预处理，对模型进行排序，之后对每条射线进行次线性的搜索。
最常用的两种排序族分别是：1）区域划分； 2）物体划分；第二种相对来说实现起来比较简单，而且对大多数模型来说速度和第一种差不多。

包围盒的关键思想是找到一个完全覆盖所有物体的盒子。例如说，当你找到一个包围10个物体的包围方盒。此时如果光线不会射中这个方盒，那就肯定不会射中那10个物体。因此相关的伪代码就是这样的：
``` github
if (ray hits bounding object)
    return whether ray hits bounded objects
else
    return false
```
将物体划分为多个子集是非常关键的。各个物体只包围在（属于）某一个包围盒中，各个包围盒可以重复。

![包围盒示意图](https://github.com/coderooookie/coderooookie.github.io/blob/master/img/2-2-1.png)

因此，我们需要一个好方法来进行划分，让包围盒足够紧凑。其次还需要一个方法来判断射线与包围盒子集相交，这得足够快。
在实践中，AABB类型的碰撞盒是最好的，但是当有多重不同类型的复杂模型时，就不太适合用AABB的碰撞盒了。

大多数人用slab的方法，就是将n个维度分别进行分析的AABB方法。
具体来说就是考虑一个维度（例如x）里，射线 p = A + Bt 与x = x0,与x = x1的交点，求得tx0和tx1，得到一个范围（tx0, tx1)。其中tx0 = (x0 - A.x)/B.x
接着分别求第二个维度（例如y）的范围(ty0, ty1)，进而判断两个范围（tx0, tx1)与(ty0, ty1) 是否有交集，有的话就说明在这个二维的区域中射线射中区域了。
三维以及以上的类似。

看起来这个方法很简单，不过这里有一些细节会让这个方法变得麻烦：
- 例如当射线是往负方向的，这样求出来的范围需要表示为(tx1, tx0)。
- 另外，如果射线与区域是平行的，即没有解的话，会得到NaN。
- 如果方向向量B里面有某个值为0，那就会出现除数为0的情况。

先不考虑上面几点，我们可以算出区域(d,D)与(e,E)的交集(f,F)：
``` github
bool overlap(d,D,e,E,f,F)
    f = max(d,e)
    F = min(D,E)
    return (f < F)
```
接下来，考虑实现aabb类如下代码。
```c++
inline float ffmin(float a, float b) {return a < b ? a : b;}
inline float ffmax(float a, float b) {return a > b ? a : b;}

class aabb {
public:
    aabb(){}
    aabb(const vec3& a, const vec3& b){ _min = a; _max = b;}

    vec3 min() const {return _min;}
    vec3 max() const {return _max;}

    bool hit(const ray& r, float tmin, float tmax) const {
        for (int a = 0; a < 3; a++){
            float t0 = ffmin((_min[a]-r.origin()[a])/r.direction()[a], (_max[a]-r.origin()[a])/r.direction()[a]);
            float t1 = ffmax((_min[a]-r.origin()[a])/r.direction()[a], (_max[a]-r.origin()[a])/r.direction()[a]);
            tmin = ffmax(t0, tmin);
            tmax = ffmin(t1, tmax);
            if (tmax <= tmin)
                return false;
        }
        return true;
        }

    vec3 _min;
    vec3 _max;
};
```

核心就是aabb的hit方法。

接着在原有的hitable类上增加虚函数。
```c++
class hitable {
public:
    virtual bool hit(const ray& r, float t_min, float t_max, hit_record& rec) const = 0;
    virtual bool bounding_box(float t0, float t1, aabb& box) const = 0;
};
```
`bounding_box`方法用于获取其包围盒，对于sphere类的实现如下。
```c++
bool sphere::bounding_box(float t0, float t1, aabb& box) const {
    box = aabb(center - vec3(radius, radius, radius), center + vec3(radius, radius, radius));
    return true;
}
```
而对于`moving_sphere`类，而只需要考虑将两个aabb拼起来。
```c++
aabb surrounding_box(aabb box0, aabb box1){
    vec3 small(fmin(box0.min().x(), box1.min().x()),
    fmin(box0.min().y(), box1.min().y()),
    fmin(box0.min().z(), box1.min().z()));
    vec3 big(fmin(box0.max().x(), box1.max().x()),
    fmin(box0.max().y(), box1.max().y()),
    fmin(box0.max().z(), box1.max().z()));
    return aabb(small, big);
}
```
对于`hitable_list`类的话如下。
```c++
bool hitable_list::bounding_box(float t0, float t1, aabb& box) const {
    if (list_size < 1) return false;
    aabb temp_box;
    bool first_true = list[0]->bounding_box(t0, t1, temp_box);
    if (!first_true) return false;
    else box = temp_box;
    for (int i = 1; i < list_size;i++){
        if (list[i]->bounding_box(t0, t1, temp_box)){
            box = surrounding_box(box, temp_box);
        }else{
            return false;
        }
    }
    return true;
}
```
接下来要考虑比较关键的类`bvh_node`，也就是层次包围盒节点的类，这个类当然也应该是可以hit到的，因此继承`hitable`类。它的实现比较关键的就是考虑`hit`函数和构造函数。
对于前者主要是要递归地判断左节点和右节点是否有hit中；而对于后者，则是需要考虑怎么将一个list进行分类，作为`bvh_node`的左右节点。代码如下。
```c++

#include "hitable.h"

class bvh_node : public hitable {

public:
    bvh_node() {}
    bvh_node(hitable **l, int n, float time0, float time1);
    virtual bool hit(const ray& r, float tmin, float tmax, hit_record& rec) const;
    virtual bool bounding_box(float t0, float t1, aabb& box) const;
    hitable *left;
    hitable *right;
    aabb box;
};

bool bvh_node::bounding_box(float t0, float t1, aabb& b) const {
    b = box;
    return true;
}

bool bvh_node::hit(const ray&r, float t_min, float t_max, hit_record& rec) const {
    if (box.hit(r, t_min, t_max)){
        hit_record left_rec, right_rec;
        bool hit_left = left->hit(r, t_min, t_max, left_rec);
        bool hit_right = right->hit(r, t_min, t_max, right_rec);
        if (hit_left && hit_right){
            if (left_rec.t < right_rec.t)
                rec = left_rec;
            else
                rec = right_rec;
            return true;
        }else if (hit_left){
            rec = left_rec;
            return true;
        }else if (hit_right){
            rec = right_rec;
            return true;
        }else
        return false;
    }
    else
        return false;
}

int box_x_compare(const void* a, const void *b){
    aabb box_left, box_right;
    hitable *ah = *(hitable**)a;
    hitable *bh = *(hitable**)b;
    if (!ah->bounding_box(0, 0, box_left) || !bh->bounding_box(0, 0, box_right))
        std::cerr << "no bounding box in bvh_node constructor\n";
    if (box_left.min().x() - box_right.min().x() < 0.0)
        return -1;
    else
        return 1;
}

int box_y_compare(const void* a, const void *b){

}

int box_z_compare(const void* a, const void *b){

}

bvh_node::bvh_node(hitable **l, int n, float time0, float time1){
    int axis = int(3*drand48());
    if (axis == 0)
        qsort(l, n, sizeof(hitable *), box_x_compare);
    else if (axis == 1)
        qsort(l, n, sizeof(hitable *), box_y_compare);
    else
        qsort(l, n, sizeof(hitable *), box_z_compare);
    if (n == 1){
        left = right = l[0];
    }else if (n == 2){
        left = l[0];
        right = l[1];
    }else{
        left = new bvh_node(l, n/2, time0, time1);
        right = new bvh_node(l + n/2, n - n/2, time0, time1);
    }
    aabb box_left,box_right;
    if (!left->bounding_box(time0, time1, box_left) || !right->bounding_box(time0, time1, box_right))
        std::cerr << "no bounding box in bvh_node constructor\n";
    box = surrounding_box(box_left, box_right);
}
```
作者对bvh_node的构造分为`n`为1和2以及其他的情况，边界考虑得比较清楚。
以上，现在就可以将bvh_node用到之前的代码中了。等下次我再试试看效率变化是否明显。
