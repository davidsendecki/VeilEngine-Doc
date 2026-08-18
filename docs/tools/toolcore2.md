# ToolCore2

ToolCore2 is the common foundation used by Veil's standalone development tools. Its purpose is to keep application plumbing and reusable editor facilities out of individual tools.

## Current organization

```text
toolcore2/src/
├─ app/
├─ commands/
├─ core/
├─ framegui/
├─ input/
├─ platform/
├─ render/
├─ services/
└─ shared/
```

## Responsibilities

**Application infrastructure** provides the common lifecycle used by tool executables. **Commands** centralize reusable command/shortcut behavior. **FrameGUI** provides the editor UI framework. **Input** and **platform** isolate common interaction/platform work. **Render** provides shared editor rendering infrastructure. **Services** connect tools to reusable functionality such as asset processing.

This division lets Material Editor and Model Studio concentrate on their domain-specific documents, panels, importers, and preview logic instead of each implementing another application framework.

## Service boundary

Tool functionality that is useful across multiple applications should generally be considered for a shared service rather than copied into an individual editor. Asset compilation is a key example: the editor initiates the operation, while the reusable compilation logic belongs in AssetServices.

## Stability

!!! warning
    ToolCore2 is not yet a frozen SDK. Its rendering and GUI architecture in particular may change as the tool renderer and FrameGUI evolve.
