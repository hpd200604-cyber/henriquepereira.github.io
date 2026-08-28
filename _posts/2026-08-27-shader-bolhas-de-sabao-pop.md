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
    // Coordenadas corrigidas com aspect ratio para a bolha ser perfeitamente redonda
    float2 uv = i.uv - 0.5;
    float2 p = uv * float2(16.0 / 9.0, 1.0) * 2.3;
    float r2 = dot(p, p);

    // Cor de fundo ambiente suave (gradiente sutil)
    float3 bgColor = float3(0.06, 0.08, 0.12) + (1.0 - length(uv) * 1.2) * float3(0.04, 0.05, 0.07);

    // Pixels fora do raio da bolha mostram o fundo
    if (r2 > 1.0) {
        return float4(bgColor, 1.0);
    }

    // Normal 3D da superfície e vetores de iluminação
    float z = sqrt(1.0 - r2);
    float3 N = float3(p.x, p.y, z);
    float3 V = float3(0.0, 0.0, 1.0);
    float3 L = normalize(float3(-0.6, 0.6, 0.8));

    // 1. Fresnel de visão (bordas brilhantes e centro translúcido)
    float NdotV = saturate(dot(N, V));
    float fresnelView = pow(1.0 - NdotV, 2.5);

    // 2. Fresnel baseado na luz principal (ponto de brilho especular suave)
    float NdotL = saturate(dot(N, L));
    float fresnelLight = pow(1.0 - NdotL, 2.8);

    // 3. Efeito iridescente (interferência de película fina de sabão)
    float3 iridescence = 0.5 + 0.5 * cos(NdotV * 6.0 + _Time.y * 1.5 + float3(0.0, 2.0, 4.0));
    float3 bubbleColor = lerp(float3(0.3, 0.8, 1.0), iridescence, 0.65);

    float3 finalRGB = lerp(bgColor, bubbleColor, fresnelView * 0.7 + 0.1) + (fresnelLight * 0.4) + (fresnelView * bubbleColor * 1.3);

    // 4. Mecânica do Estouro (Pop com Simple Noise e Step)
    float popProgress = smoothstep(0.7, 0.95, frac(_Time.y * 0.25));
    float noise = simpleNoise(i.uv * 18.0);

    if (noise < popProgress) {
        return float4(bgColor, 1.0); // Retorna o fundo quando estoura
    }

    return float4(finalRGB, 1.0);
}
{% endcapture %}

{% include hlsl-viewer.html id="bolha-demo" code=bolha_hlsl aspect="16:9" %}

---

## 1. O Conceito

Este shader foi criado no Unity Universal Render Pipeline (URP) para simular **bolhas de sabão translúcidas**. Ele combina dois comportamentos visuais:
1. **Transparência angular com Fresnel duplo:** a bolha é transparente ao olhar direto e reflete cores nas bordas e em direção à luz.
2. **Estouro procedural (*Pop*):** a película se desfaz em fragmentos orgânicos usando ruído procedural e corte binário de alfa.

---

## 2. Parâmetros Principais

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

## 3. A Lógica Principal (HLSL)

Abaixo está o núcleo da função de fragmento onde toda a matemática e a estética do shader são calculadas:

```hlsl
SurfaceDescription SurfaceDescriptionFunction(SurfaceDescriptionInputs IN)
{
    SurfaceDescription surface;

    // 1. Fresnel de visão (transparência central e brilho de contorno)
    float fresnelView = pow(1.0 - saturate(dot(IN.WorldSpaceNormal, IN.WorldSpaceViewDirection)), _Power);

    // 2. Fresnel especular da luz principal (ponto de brilho curvo)
    float3 lightDir = MainLightDirection();
    float fresnelLight = pow(1.0 - saturate(dot(IN.WorldSpaceNormal, lightDir)), 2.79);

    // 3. Composição de cor e reflexo da película
    float4 texColor = SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, IN.uv0.xy);
    float4 reflection = (texColor * fresnelView * fresnelLight) + (_Fresnel_Color * fresnelView);
    surface.BaseColor = reflection.rgb;

    // 4. Mecânica de estouro com ruído procedural (Simple Noise + Step)
    float noise = SimpleNoise(IN.uv0.xy, 25.0);
    float popThreshold = 1.0 - _Pop;
    float alphaCut = step(noise, popThreshold);

    // 5. Saída de alfa final com descarte dos fragmentos estourados
    surface.Alpha = fresnelView * alphaCut;
    surface.AlphaClipThreshold = 0.5;

    return surface;
}
```

---

## 4. Como funciona o efeito de "Pop":

1. **`SimpleNoise(uv, 25.0)`**: Produz uma distribuição determinística de valores em escala de cinza (`0.0` a `1.0`) na superfície da bolha.
2. **`1.0 - _Pop`**: Converte o slider de estouro em um limiar decrescente.
3. **`step(noise, popThreshold)`**:
   - Enquanto `_Pop == 0.0`, o limiar é `1.0` e quase todo o ruído passa (bolha visível e intacta).
   - Conforme `_Pop` se aproxima de `1.0`, o limiar diminui e o `step` retorna `0.0`, cortando o canal alfa em buracos orgânicos até a bolha sumir por completo.
