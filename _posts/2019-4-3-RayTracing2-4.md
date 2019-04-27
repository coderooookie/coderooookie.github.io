---
title: Ray Tracing in the next week - Perlin Noise
date: 2019-4-4 12:43
categories:
- 游戏开发
- Ray Tracing
---

本章主要讲柏林噪声，据说柏林噪声会比较难，不过接触过接触过电子信号或者图形学相关的应该都会对此已经有了一定了解。

柏林噪声经常用来生出一些具有一定随机性的纹理贴图。他的一个关键点在于其伪随机性，也就是重复相同的输入，会得到相同的输出；另一个关键点在于其简洁快速。

这个的实现主要是先实现好随机的数组，然后每次输入作为数组的小标去取数组的内容，就可以做到伪随机了。代码也不算难：

```c++
#ifndef perlin_h
#define perlin_h
#include "vec3.hpp"

//插值函数
inline float trilinear_interp(float c[2][2][2], float u, float v, float w){
    float accum = 0;
    for (int i = 0; i < 2; i++)
    for (int j = 0; j < 2; j++)
    for (int k = 0; k < 2; k++)
    accum += (i*u + (1-i)*(1-u)) * (j*v + (1-j)*(1-v)) * (k*w + (1-k)*(1-w)) * c[i][j][k];
    return accum;
}

class perlin {
public:
    float noise(const vec3& p) const {
        float u = p.x() - floor(p.x());
        float v = p.y() - floor(p.y());
        float w = p.z() - floor(p.z());
        int i = floor(p.x());
        int j = floor(p.y());
        int k = floor(p.z());
        float c[2][2][2];
        for (int di = 0; di < 2; di ++)
            for (int dj = 0; dj < 2; dj++)
                for (int dk = 0; dk < 2; dk++)
                    c[di][dj][dk] = ranfloat[perm_x[(i+di)&255] ^ perm_y[(j+dj) & 255] ^ perm_z[(k+dk) & 255]];
        return trilinear_interp(c, u, v, w);
    }

    static float *ranfloat;
    static int *perm_x;
    static int *perm_y;
    static int *perm_z;
};

//产生一堆0到1之间的随机数
static float* perlin_generate(){
    float* p = new float[256];
    for (int i = 0; i < 256; i++){
        p[i] = drand48();
    }
    return p;
}

//打乱一个数组
void permute(int *p, int n){
    for (int i = n-1; i > 0; i--){
        int target = int(drand48() * (i+1));
        int tmp = p[i];
        p[i] = p[target];
        p[target] = tmp;
    }
    return;
}

//产生一个长度为256的数组，并且打乱0-255这些数字
static int* perlin_generate_perm(){
    int* p = new int[256];
    for (int i = 0; i < 256; i++){
        p[i] = i;
    }
    permute(p, 256);
    return p;
}

float *perlin::ranfloat = perlin_generate();
int *perlin::perm_x = perlin_generate_perm();
int *perlin::perm_y = perlin_generate_perm();
int *perlin::perm_z = perlin_generate_perm();

#endif /* perlin_h */
```

这部分代码跑完会发现出现了马赫带 ( Mach bands ) 。因此可以使用hermite cubic来进行插值。
将这部分代码加载`noise`函数中：
```c++
u = u*u*(3-2*u);
v = v*v*(3-2*v);
w = w*w*(3-2*w);
```
实现后的noise函数变成：
```c++
float noise(const vec3& p) const {
    float u = p.x() - floor(p.x());
    float v = p.y() - floor(p.y());
    float w = p.z() - floor(p.z());
    u = u*u*(3-2*u);
    v = v*v*(3-2*v);
    w = w*w*(3-2*w);
    int i = floor(p.x());
    int j = floor(p.y());
    int k = floor(p.z());
    float c[2][2][2];
    for (int di = 0; di < 2; di ++)
        for (int dj = 0; dj < 2; dj++)
            for (int dk = 0; dk < 2; dk++)
                c[di][dj][dk] = ranfloat[perm_x[(i+di)&255] ^ perm_y[(j+dj) & 255] ^ perm_z[(k+dk) & 255]];
    return trilinear_interp(c, u, v, w);
}
```

这样插值后的柏林噪声会比较柔和，但作者认为随机频率太低，因此可以给`noise_texture`加一个参数`scale`，可以手动调节噪声的随机频率，实现不同粒度的噪声。
```c++
class noise_texture : public texture{
public:
    noise_texture(){}
    noise_texture(float sc):scale(sc){}
    virtual vec3 value(float u, float v, const vec3& p) const{
        return vec3(1,1,1) * noise.noise(p);
    }
    perlin noise;
    float scale;
};
```

```c++
pthread_mutex_t mutex1;

struct thread_data {
    int nx;
    int ny;
    int ns;
    int i;
    int j;
    camera* cam;
    hitable* world;
    vec3** result;
};

void* generateColor(void* threadArgs){
    thread_data* threadData;
    threadData = (thread_data*) threadArgs;
    vec3 col(0,0,0);
    for (int s=0; s <threadData->ns; s++){
        float u = float(threadData->i + drand48())/threadData->nx;
        float v = float(threadData->j + drand48())/threadData->ny;
        ray r = threadData->cam->get_ray(u, v);
        col += color(r, threadData->world, 0);
    }
    col /= float(threadData->ns);
    col = vec3(sqrt(col[0]), sqrt(col[1]), sqrt(col[2]));
    pthread_mutex_lock(&mutex1);
    *(threadData->result)[threadData->j*threadData->nx + threadData->i] = col;
    pthread_mutex_unlock(&mutex1);
    pthread_exit(NULL);
}

pthread_mutex_init(&mutex1,NULL);
    vec3 *colorResult[nx*ny];
    pthread_t tids[nx*ny];
    for (int j = ny-1; j >= 0; j--){
        for (int i = 0; i < nx; i++){
            thread_data threadData = { nx,ny,ns,i,j,&cam,world,colorResult};
    //            pthread_create(pthread_t  _Nullable * _Nonnull, const pthread_attr_t * _Nullable, void * _Nullable (* _Nonnull)(void * _Nullable), void * _Nullable)
            int ret = pthread_create(&tids[j*nx + i], NULL, generateColor, &threadData);
            if (ret != 0)
            {
                std::cout << "pthread_create error: error_code=" << ret << std::endl;
            }
        }
    }
    for (int j = ny-1; j >= 0; j--){
        for (int i = 0; i < nx; i++){
        //            vec3 p = r.point_at_parameter(2.0);
            int ir = int(255.99 * (*colorResult[j*nx + i]).r());
            int ig = int(255.99 * (*colorResult[j*nx + i]).g());
            int ib = int(255.99 * (*colorResult[j*nx + i]).b());
            outfile << ir << " " << ig << " " << ib << "\n";
        }
    }
```
得出结果后，作者认为依然还是有一些blocky looking, 因此将柏林噪声里面的随机值从float改为vec3。插值函数也随之进行修改。
```c++
static vec3* perlin_generate(){
    vec3 *p = new vec3[256];
    for (int i = 0; i < 256; ++i)
        p[i] = unit_vector(vec3(-1 + 2*drand48(), -1 + 2*drand48(), -1 + 2*drand48()));
    return p;
}

inline float trilinear_interp(vec3 c[2][2][2], float u, float v, float w){
    float uu = u*u*(3-2*u);
    float vv = v*v*(3-2*v);
    float ww = w*w*(3-2*w);
    float accum = 0;
    for (int i = 0; i < 2; i++)
        for (int j = 0; j < 2; j++)
            for (int k = 0; k < 2; k++){
                vec3 weight_v(u-i,v-j,w-k);
                accum += (i*uu + (1-i)*(1-uu)) * (j*vv + (1-j)*(1-vv)) * (k*ww + (1-k)*(1-ww)) * dot(c[i][j][k], weight_v);
            }
    return accum;
}
```

在实际应用中，柏林噪声经常是用多层柏林噪声来叠加而成的，这样得到的效果更好，因此在类中增加一个函数。
```C++
    float turb(const vec3&p, int depth = 7) const {
        float accum = 0;
        vec3 temp_p = p;
        float weight = 1.0;
        for (int i = 0; i < depth; i++){
            accum += weight * noise(temp_p);
            weight *= 0.5;
            temp_p *= 2;
        }
        return fabs(accum);
    }
```

最后，作者利用正弦函数，实现一个大理石效果的玻璃噪声。效果看起来还不错。
```C++
class noise_texture : public texture{
public:
    noise_texture(){scale = 1;}
    noise_texture(float sc):scale(sc){}
    virtual vec3 value(float u, float v, const vec3& p) const{
    //        return vec3(1,1,1) * noise.noise(p * scale);
    //        return vec3(1,1,1) * noise.turb(scale * p);
        return vec3(1,1,1)*0.5*(1+sin(scale*p.z() + 10 * noise.turb(p)));
    }
    perlin noise;
    float scale;
};
```

前段时间因为项目在赶进度，回到家里实在是没动力继续看和写blog，因此耽搁了接近一个月，现在回来继续，不过考虑到近期的一些安排，后面应该会写unity  shader相关的内容。
整理一下本书相关的内容后面的大概内容如下：
1. 2-5 Image Texture Mapping ：在ray tracer中加入纹理
2. 2-6 Rectangles and Lights ：在ray tracer中加入正方形和灯光
3. 2-7 Instances ：实现cornell box的实例化
4. 2-8 Volumes：烟雾相关的内容
5. 2-9 A Scene Testing All New Features：本书相关总结
而unity shader相关的内容计划如下:
1. U-5 开始Unity Shader学习之旅
2. U-6 Unity中的基础光照
3. U-7 基础纹理
4. U-8 透明效果
5. U-9 更复杂的光照
6. U-10 高级纹理
7. U-11 让动画动起来
8. U-12 屏幕后处理效果
9. U-13 使用深度和法线纹理
10. U-14 非真实感渲染
11. U-15 使用噪声
如果以上内容都完成，下一步会考虑PBR或者其他相关Unity Shader的比较有意思的内容。
