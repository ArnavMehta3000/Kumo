---
tags:
  - "#Unity"
  - "#Unity/Rendering"
  - Unity/RenderGraph
---
# References
- [Introduction to URP for advanced creators (Book PDF) | Unity](https://unity.com/resources/introduction-to-urp-advanced-creators-unity-6)
 
# Main Principles of a Render Graph
1. You no longer handle resources directly; instead, you use render graph system-specific handles. All RenderGraph APIs use these handles to manipulate resources. The resource types a render graph manages are `RTHandles`, `ComputeBuffers`, and `RendererLists`
2. Actual resource references are only accessible within the execution code of a render pass
3. The framework requires an explicit declaration of render passes. Each render pass must state which resources it reads from and/or writes to
4. There is no persistence between each execution of a render graph. This means that the resources you create inside one execution of the render graph don’t carry over to the next execution
5. For resources that need persistence (e.g., from one frame to another), you can create them outside of a render graph, like regular resources, and import them. They behave like any other render graph resource in terms of dependency tracking, but the graph does not handle their lifetime
6. A render graph mostly uses `RTHandles` for texture resources. This has a number of implications on how to write shader code and how to set them up

# Resource Management
When creating a resource via the render graph, the resource will *not* be created will the created till first pass that needs to write it. Similarly the resource is released after the last pass that needs it has finished reading it.

# Execution Overview
The execution of a render graph is a 3 step process that happens every frame because the graph and dynamically change from frame to frame

## Setup
- The first step is to set up all the render passes
- This is where you declare all the render passes to execute and the resource each render pass uses

## Compilation
- The second step is to compile the graph
- Here the render graph system culls the render passes if no other render pass depends on their outputs
	- This allows for less organized setups
	- This also calculates the lifetime of resources - allowing it to create and release the resources in an efficient way as well as compute the proper synchronization points (when executing passes on the async compute pipeline) 

## Execution
- The render graph executes all the passes that it did not cull, in declaration order
- Resources are created and destroyed as needed
