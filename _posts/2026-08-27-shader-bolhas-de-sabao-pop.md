---
title: "Shader de Bolhas de Sabão com Efeito Pop"
description: "Estudo e quebra técnica de um shader de bolhas de sabão desenvolvido no Unity Shader Graph, com Fresnel duplo e efeito de estouro procedural."
author: henriquepereira
date: 2026-08-27 15:00:00 -0300
categories: [Shaders, Unity]
tags: [hlsl, shadergraph, urp, fresnel, dissolve, vfx]
pin: true
math: true
---

### Demonstração Interativa em Tempo Real

Abaixo está a renderização em tempo real do shader simulando a geometria 3D da bolha, o Fresnel e o ciclo de estouro:

{% capture bolha_hlsl %}
float hash(float2 p) {
    return frac(sin(dot(p, float2(12.9898, 78.233))) * 43758.5453);
}

float simpleNoise(float2 uv) {
    float2 i = floor(uv);
    float2 f = frac(uv);
    f = f * f * (3.0 - 2.0 * f);
    float r0 = hash(i);
    float r1 = hash(i + float2(1.0, 0.0));
    float r2 = hash(i + float2(0.0, 1.0));
    float r3 = hash(i + float2(1.0, 1.0));
    return lerp(lerp(r0, r1, f.x), lerp(r2, r3, f.x), f.y);
}

float4 frag(v2f i) : SV_Target
{
    float2 p = (i.uv - 0.5) * 2.2;
    float r2 = dot(p, p);

    if (r2 > 1.0) {
        return float4(0.0, 0.0, 0.0, 0.0);
    }

    float z = sqrt(1.0 - r2);
    float3 N = float3(p.x, p.y, z);
    float3 V = float3(0.0, 0.0, 1.0);
    float3 L = normalize(float3(-0.6, 0.6, 0.8));

    float NdotV = saturate(dot(N, V));
    float fresnelView = pow(1.0 - NdotV, 2.5);

    float NdotL = saturate(dot(N, L));
    float fresnelLight = pow(1.0 - NdotL, 2.8);

    float3 iridescence = 0.5 + 0.5 * cos(NdotV * 6.0 + _Time.y * 1.2 + float3(0.0, 2.0, 4.0));
    float3 bubbleColor = lerp(float3(0.3, 0.8, 1.0), iridescence, 0.55);

    float3 finalRGB = bubbleColor * (fresnelView * 1.5) + fresnelLight * 0.4;
    float alpha = saturate(fresnelView * 1.3 + 0.08);

    float popProgress = smoothstep(0.65, 0.95, frac(_Time.y * 0.3));
    float noise = simpleNoise(i.uv * 20.0);

    if (noise < popProgress) {
        return float4(0.0, 0.0, 0.0, 0.0); // Fragmentos estourados
    }

    return float4(finalRGB, alpha);
}
{% endcapture %}

{% include hlsl-viewer.html id="bolha-demo" code=bolha_hlsl aspect="4:3" %}

---

## 1. Parâmetros Principais

```hlsl
Properties
{
    _Power ("Power", Float) = 0.5                // Espessura e queda do Fresnel
    _Fresnel_Color ("Fresnel Color", Color) = (1, 0, 0, 0) // Cor predominante da borda
    _Pop ("Pop", Range(0, 1)) = 0.5              // Gatilho de estouro da bolha
    _Texture2D ("Texture2D", 2D) = "white" {}    // Mapa de reflexos/iridescência
}
```

---

## 2. A Lógica Principal (HLSL)

```hlsl
SurfaceDescription SurfaceDescriptionFunction(SurfaceDescriptionInputs IN)
{
    SurfaceDescription surface;

    float fresnelView = pow(1.0 - saturate(dot(IN.WorldSpaceNormal, IN.WorldSpaceViewDirection)), _Power);

    float3 lightDir = MainLightDirection();
    float fresnelLight = pow(1.0 - saturate(dot(IN.WorldSpaceNormal, lightDir)), 2.79);

    float4 texColor = SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, IN.uv0.xy);
    float4 reflection = (texColor * fresnelView * fresnelLight) + (_Fresnel_Color * fresnelView);
    surface.BaseColor = reflection.rgb;

    float noise = SimpleNoise(IN.uv0.xy, 25.0);
    float popThreshold = 1.0 - _Pop;
    float alphaCut = step(noise, popThreshold);

    surface.Alpha = fresnelView * alphaCut;
    surface.AlphaClipThreshold = 0.5;

    return surface;
}
```
