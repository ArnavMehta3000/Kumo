---
tags:
  - Unity
  - Unity/Rendering
  - Unity/RenderGraph
  - Unity/RenderPass
---
> [!info]
> This page goes over creating a render pass to be used in a render graph in #Unity


1. Declare a render pass
	- Create a class inheriting from `ScriptableRenderPass`
2. Declare resources that are to be used in the render pass
	- This can be C# variables and render graph resource references
	- The `RecordRenderGraph` function populates the data and the render graph passes it as a parameter to the rendering function

```csharp
class PassData  // Example data
{
    internal TextureHandle copySourceTexture;
}
```

3. Implement the `RecordRenderGraph` function to add and configure one or more render passes in the render graph system
	- Called during render graph [[Render Graph#Setup|configuration step]]
	- Lets you register relevant passes & resources for the render graph execution
	- **Use this function to implement custom rendering**
	- In this method you declare render pass inputs and outputs, but do not add commands to the command buffers