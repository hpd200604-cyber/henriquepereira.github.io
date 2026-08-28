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
    f = f * f * (float2(3.0, 3.0) - float2(2.0, 2.0) * f);
    float r0 = hash(i);
    float r1 = hash(i + float2(1.0, 0.0));
    float r2 = hash(i + float2(0.0, 1.0));
    float r3 = hash(i + float2(1.0, 1.0));
    return lerp(lerp(r0, r1, f.x), lerp(r2, r3, f.x), f.y);
}

float4 frag(v2f i) : SV_Target
{
    float2 uv = i.uv - float2(0.5, 0.5);
    float2 p = uv * float2(16.0 / 9.0, 1.0) * 2.2;
    float r2 = dot(p, p);

    float3 bgColor = float3(0.08, 0.11, 0.16) + (1.0 - length(uv) * 1.3) * float3(0.05, 0.07, 0.10);

    if (r2 > 1.0) {
        return float4(bgColor, 1.0);
    }

    float z = sqrt(1.0 - r2);
    float3 N = float3(p.x, p.y, z);
    float3 V = float3(0.0, 0.0, 1.0);
    float3 L = normalize(float3(-0.6, 0.6, 0.8));

    float NdotV = saturate(dot(N, V));
    float fresnelView = pow(1.0 - NdotV, 2.4);

    float NdotL = saturate(dot(N, L));
    float fresnelLight = pow(1.0 - NdotL, 2.8);

    float3 iridescence = float3(0.5, 0.5, 0.5) + 0.5 * cos(NdotV * 6.0 + _Time.y * 1.5 + float3(0.0, 2.0, 4.0));
    float3 bubbleColor = lerp(float3(0.3, 0.8, 1.0), iridescence, 0.65);

    float3 finalRGB = lerp(bgColor, bubbleColor, fresnelView * 0.75 + 0.15) 
                    + (fresnelLight * 0.4) 
                    + (fresnelView * bubbleColor * 1.4);

    float popProgress = smoothstep(0.7, 0.95, frac(_Time.y * 0.25));
    float noise = simpleNoise(i.uv * 18.0);

    if (noise < popProgress) {
        return float4(bgColor, 1.0);
    }

    return float4(finalRGB, 1.0);
}
{% endcapture %}

{% include hlsl-viewer.html id="bolha-demo" code=bolha_hlsl aspect="16:9" %}

---

## 1. Parâmetros Principais

```hlsl
Properties
{
    _Power ("Power", Float) = 0.5
    _Fresnel_Color ("Fresnel Color", Color) = (1, 0, 0, 0)
    _Pop ("Pop", Range(0, 1)) = 0.5
    _Texture2D ("Texture2D", 2D) = "white" {}
}
```

---

## 2. A Lógica Principal (HLSL)

Abaixo está o núcleo da função de fragmento onde toda a matemática e a estética do shader são calculadas:

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

---

## 3. Como funciona o efeito de "Pop":

1. **`SimpleNoise(uv, 25.0)`**: Produz uma distribuição determinística de valores em escala de cinza (`0.0` a `1.0`) na superfície da bolha.
2. **`1.0 - _Pop`**: Converte o slider de estouro em um limiar decrescente.
3. **`step(noise, popThreshold)`**:
   - Enquanto `_Pop == 0.0`, o limiar é `1.0` e quase todo o ruído passa (bolha visível e intacta).
   - Conforme `_Pop` se aproxima de `1.0`, o limiar diminui e o `step` retorna `0.0`, cortando o canal alfa em buracos orgânicos até a bolha sumir por completo.