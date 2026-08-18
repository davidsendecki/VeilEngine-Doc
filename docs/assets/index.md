# Assets & Formats

Veil separates authoring data from runtime-ready assets. Editor applications and AssetServices are responsible for producing data that runtime subsystems can consume efficiently.

## Current areas

The shared source contains common asset handles and model representations together with dedicated format definitions. At present the compiled-format area contains **VMDL** model data and **VMAP** map data.

Shader sources live separately under `assets/shaders`, with reusable modules shared between runtime and tool-oriented shaders where appropriate.

## Design principle

Compiled formats should be explicit and versionable. Runtime loaders should validate the file before exposing its contents, while editor-side compilers own the conversion from editable/imported data into those runtime structures.
