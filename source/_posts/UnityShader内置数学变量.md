---
title: UnityShader内置数学变量
tags: Unity
categories: 笔记
copyright: true
reward: true
date: 2020-03-14 15:32:32
thumbnail:
---


## Unity内置变换矩阵和常见用法

变量名|描述
---|---
UNITY_MATRIX_V|当前模型的观察矩阵，从世界空间到观察空间 **(V->世界空间-观察矩阵)**
UNITY_MATRIX_P|当前模型的投影矩阵，从观察空间到裁剪空间 **(P->观察空间-裁剪空间)**
UNITY_MATRIX_VP|观察·投影矩阵,从世界空间变换到裁剪空间 **(VP->世界空间-裁剪空间)**
UNITY_MATRIX_MVP|模型·观察·投影矩阵，从模型空间变换到裁剪空间 **(MVP->模型空间-裁剪空间)**
UNITY_MATRIX_MV|模型·观察矩阵，从模型空间变换到观察空间 **(MV->模型空间-观察空间)**
UNITY_MATRIX_T_MV|UNITY_MATRIX_MV的转置矩阵(在UNITY_MATRIX_MV为正交矩阵的情况下，它的逆矩阵就是UNITY_MATRIX_T_MV)。当模型的变换 **只有旋转和统一缩放(假设缩放系数为K)** 的时候，UNITY_MATRIX_MV的逆矩阵就是$\frac{1}{k}$UNITY_MATRIX_MV
UNITY_MATRIX_IT_MV|UNITY_MATRIX_MV的逆转置矩阵，**用于将法线从模型空间变换到观察空间** 。可以用于得到UNITY_MATRIX_MV的逆矩阵。
_Object2World|当前模型矩阵，从模型空间变换到世界空间
_World2Object|_Object2World的逆矩阵，从世界空间变换到模型空间

以上变量都是float4x4类型

## Unity内置的摄像机和屏幕参数

变量名|类型|描述
---|---|---
_WorldSpaceCameraPos|float3|摄像机在世界空间中的位置
_ProjectionParams|float4|x=1.0 或 x=-1.0, y=Near, z=Far, w=1.0+1.0/Far
_ScreenParams|float4|x=width, y=height, z=1.0+1.0/width， w=1.0+1.0/height。 width和height分别是该摄像机渲染目标的像素宽度和高度
_ZBufferParams|float4|x=1-Far/Near, y=Far/Near, z=x/Far, w=y/Far,变量用于线性化Z缓存中的深度值
_unity_OrthoParams|float4|x=width, y=height, z没有定义, w=1.0(是正交摄像机),w=0.0(是透视摄像机)。width和height是正交投影摄像机的宽度和高度(屏幕分辨率)
unity_CameraProject|float4x4|摄像机的投影矩阵
unity_CameraInvProject|float4x4|摄像机投影矩阵的逆矩阵
unity_CameraWorldClipPlanes[6]|float4|摄像机的6个裁剪平面在世界空间下的等式，顺序是：左->右->下->上->近->远裁剪平面

## ComputeScreenPos/VPOS/WPOS

- ComputeScreenPos是Unity内置的函数，返回齐次坐标系下的屏幕坐标值
- VPOS是HLSL中对屏幕像素坐标的语义
- WPOS是CG中对屏幕坐标的语义

VPOS和WPOS在Unity中是等价的，都是float4类型

### 片元在视口空间中的坐标

#### 使用VPOS/WPOS

```cs
fixed4 frag(float4 sp : VPOS) : SV_Target{
  return fixed4(sp.xy/_ScreenParams.xy,0.0,1.0);
}
```

`sp.xy`代表了屏幕空间中的像素坐标, `_ScreenParams.xy`是屏幕分辨率

#### 使用ComputeScreenPos

```cs
struct verOut{
  float4 pos : SV_POSITION;
  float4 scrPOS : TEXCOORD0;
};
vertOut vert(appdata_base v){
  vertOut o;
  o.pos = mul(UNITY_MATRIX_MVP, v.vertex);//将顶点坐标转换为裁剪空间坐标
  o.scrPos = ComputeScreenPos(o.pos);//的到屏幕坐标
  return o;
}
fiexd4 frag(vertOut i){
  float2 wcoord = (i.scrPos.xy/i.scrPos.w);//齐次除法，的到范围在[-1,1]的NDC
  return fixed4(wcoord,0.0,1.0);//坐标映射到范围在[0，1]的视口空间下，的到视口空间中的坐标
}
```

## 运算的一些注意项

- Unity内置矩阵默认是按**列存储**的,所以变换时一般使用右乘方式
- CG中float4x4等类型是按**行优先方式**填充的
- Unity脚本中Matrix4x4类型采用的是列优先方式

>摘录自Unity Shader入门精要 4.8
