# FrameGUI

FrameGUI is the UI framework currently used by Veil's development tools and is hosted inside ToolCore2.

It provides reusable editor-facing UI infrastructure so individual tools can construct panels and controls without depending directly on a separate third-party editor framework throughout their codebase.

## Role in the tools

FrameGUI sits between tool application code and the lower-level platform/rendering infrastructure. Material Editor and Model Studio can therefore organize their interfaces as panels and widgets while ToolCore2 owns the common application integration.

## Current status

!!! warning "Planned redesign"
    FrameGUI is expected to evolve substantially. The current implementation is useful and actively used, but its API should not yet be considered final.

Documentation will therefore focus on stable concepts as they emerge: widget ownership, layout, panel construction, events/commands, rendering, and application integration. Detailed API reference should be added after the next-generation interface has stabilized enough that documenting every call is worthwhile.
