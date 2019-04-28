---
title: Unity Shader 入门5：开始
date: 2019-4-29 0:43
categories:
- 游戏开发
- Unity Shader
---

第五章是开始Unity Shader学习之旅，主要都是入门的内容，因此也没什么需要展开的，入门的代码在后面也会通过注释的形式来进行展开。

另外，可以参照官方的Shader文件进行学习和引用，Mac的地址为：`/Applications/Unity/Unity.app/Contents/CGIncludes`

```c++
Shader "Unlit/Shader_5.1" {
    Properties {
        _Color ("Color Tint", Color) = (1, 1, 1, 1)
    }
    SubShader {
        Pass {
            CGPROGRAM

            #pragma vertex vert
            #pragma fragment frag

            uniform fixed4 _Color;

            struct a2v {
                float4 vertex : POSITION;
                float3 normal : NORMAL;
                float4 texcoord : TEXCOORD0;
            };

            struct v2f {
                float4 pos : SV_POSITION;
                fixed3 color : COLOR0;
            };

            v2f vert(a2v v) {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                o.color = v.normal * 0.5 + fixed3(0.5, 0.5, 0.5);
                return o;
            }

            fixed4 frag(v2f i) : SV_Target {
                fixed3 c = i.color;
                c *= _Color.rgb;
                return fixed4(c, 1.0);
            }

            ENDCG
        }
    }
}
```
