# Materials & Shaders

Veil's shader source is stored under `assets/shaders` and is written in Slang.

## Current layout

```text
assets/shaders/
├─ StandardMaterial.slang
├─ core/
│  ├─ ACES.slang
│  ├─ BRDF.slang
│  ├─ Camera.slang
│  ├─ ColorSpace.slang
│  ├─ MaterialAttributes.slang
│  ├─ ModelTypes.slang
│  ├─ PBRLighting.slang
│  ├─ Skinning.slang
│  └─ Transforms.slang
├─ model/
│  └─ StandardModel.slang
└─ tools/
   ├─ MaterialPreview.slang
   └─ MaterialPreviewToneMap.slang
```

The `core` directory contains reusable shader functionality. Model-specific composition is kept in `model`, while preview/editor shaders live under `tools`.

## Material authoring

The Material Editor uses shader information as part of its authoring workflow. Slang source is valuable to the tool pipeline because reflection can expose material-facing parameters and custom attributes used to construct editing controls.

Runtime rendering does not need to perform the same editor reflection workflow. Runtime-facing shader artifacts can therefore remain optimized for rendering while the tools retain access to the source information required for authoring.

## Direction

Shader modules should avoid duplicating complete material implementations merely because model vertex processing differs. Shared material and lighting behavior belongs in reusable modules, while static/skeletal model paths can provide the vertex-processing differences needed by each model type.

!!! note
    Shader organization is actively evolving alongside the renderer and Material Editor. This page documents the current source layout and architectural intent rather than promising stable filenames or entry points.
