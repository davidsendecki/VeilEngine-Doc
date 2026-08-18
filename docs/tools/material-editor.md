# Material Editor

The Material Editor is Veil's dedicated material-authoring application. Its current source is separated into application setup, commands, dialogs, documents, material logic, panels, preview rendering, resources, shader handling, and textures.

## Responsibilities

The editor combines several workflows that belong together from an author's perspective:

- create, load, edit, and save material documents;
- inspect material-facing shader parameters;
- expose appropriate controls in the UI;
- compile/import texture data used by materials;
- preview the material with the editor renderer.

## Shader reflection

Slang source is used by the authoring pipeline so shader reflection can discover material parameters and editor-oriented metadata. Attributes can describe how a parameter should be presented rather than forcing every shader parameter to be hard-coded into the Material Editor.

This makes the material UI increasingly data-driven: the shader describes its material-facing inputs and the editor reflects that information into controls.

## Preview rendering

The Material Editor has dedicated preview-rendering code and tool-specific shaders under `assets/shaders/tools`. Preview rendering is an editor concern and can evolve independently from the runtime scene renderer while still sharing common shader functionality where appropriate.

## Status

The Material Editor already has a meaningful first-pass architecture, but its shader reflection, UI, and rendering integration remain areas where the implementation may continue to be simplified as ToolCore2 evolves.
