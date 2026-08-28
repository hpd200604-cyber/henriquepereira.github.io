---
title: "Shader em HLSL Rodando ao Vivo"
description: "Exemplo de como escrever em HLSL nativo da Unity e vê-lo executar diretamente na página."
author: henriquepereira
date: 2026-08-27 12:00:00 -0300
categories: [Shaders, Unity]
tags: [hlsl, unity, interactive, webgl]
pin: true
math: true
---

Abaixo está o shader executando em tempo real no navegador, alimentado diretamente por código **HLSL**:

{% capture hlsl_shader %}
float4 frag(v2f i) : SV_Target
{
    float2 uv = i.uv;
    float3 t = float3(_Time.y, _Time.y, _Time.y);
    float3 col = float3(0.5, 0.5, 0.5) + 0.5 * cos(t + uv.xyx + float3(0.0, 2.0, 4.0));
    return float4(col, 1.0);
}
{% endcapture %}

{% include hlsl-viewer.html id="exemplo-hlsl" code=hlsl_shader aspect="16:9" %}