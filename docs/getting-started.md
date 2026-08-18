# Getting Started

!!! warning "Early documentation"
    Veil Engine is under active development. Setup and build details may change as the project matures.

## Repository layout

The engine source is organized as a collection of runtime subsystems and development tools. The root build is generated with Premake.

Typical development flow:

1. Clone the `VeilEngine` repository.
2. Initialize or obtain the required third-party dependencies.
3. Run `GenerateAllProjects.bat` on Windows to generate the project files.
4. Build the desired runtime or tool target.
5. Compile shaders when required using the shader compilation tooling in the repository.

## Runtime and tools

Veil deliberately separates the game/runtime side from its authoring tools. Runtime modules live primarily below `src/`, while editor applications and their shared infrastructure live below `src/tools/`.

The documentation follows the same distinction. Start with **Architecture** for the overall layout, **Engine** for runtime systems, or **Tools** when working on editor applications.

## Current platform status

The project is currently developed primarily on Windows. Platform-specific assumptions should therefore be expected in parts of the build and toolchain while portability work is still ongoing.
