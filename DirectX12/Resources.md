- Several resources are required to render scenes, we as the programmer must decide how those buffer resources are created
- There are several different way that GPU resources are allocated within a heap memory (`ID3D12Heap`)
	- Committed Heap
	- Placed Heap
	- Reserved Heap

## Committed Resources
- A committed resource is created using the `ID3D12Device::CreateCommittedResource` function
	- This function creates both the resource and the implicit heap that is large enough to hold the resource
	- The resource is also mapped to the heap
- Committed resources are easy to manage because we don't need to be concerned with placing the resource in the heap
- These resources are ideal for allocated large resources like textures or statically sized resources (the size of the resource does not change)
- The are also commonly used to create large resource in an upload heap  that can be used for uploading dynamic vertex/index buffers (e.g. ImGui)
## Placed Resource
- A places resource is explicitly placed in a heap at a specific offset within the heap
- Before a placed resource can be created, a heap is created using `ID3D12Dvice::CreateHeap`
- The placed resource is then created inside that heap using the `ID3D12Device::CreatePlacedResource`
- Placed Resources offer better performance because the heap does not need to allocated from global GPU memory for each resource
- This comes with manual memory management (of the heap) by the programmer
- When using placed resources, there are some limitations that must be taken into consideration
- The size of the heap that will be used for the placed resources must be known in advance.
- Creating larger than necessary heaps is not a good idea because the only way to reclaim the GPU memory used by the heap is to either **evict** the heap or completely destroy it. 
	- Keep in mind since you can evict an entire heap from GPU memory, any resources that are currently placed in the heap must not be referenced in a command list that is being executed on the GPU
- Depending on the GPU architecture, the type of resource that you can allocate within a particular heap may be limited
	- For example, buffer resources (vertex buffer, index buffer, constant buffer, structure buffer, etc..) can only be placed in a heap that was created with the `[ALLOW_ONLY_BUFFERS]` heap flag. Render target and depth/stencil resources can only be placed in a heap that was created with the `[ALLOW_ONLY_RT_DS_TEXTURES]` heap flag. Non render target textures can only be placed in a heap that was created with the `[ALLOW_ONLY_NON_RT_DS_TEXTURES]` heap flag. Adapters that support [heap tier 2](https://msdn.microsoft.com/en-us/library/dn986743\(v=vs.85\).aspx) and higher can create heaps using the `[ALLOW_ALL_BUFFERS_AND_TEXTURES]` heap flag to allow any resource type to be placed within that heap. Since the heap tier is dependent on the GPU architecture, most applications will probably be written assuming only [heap tier 1](https://msdn.microsoft.com/en-us/library/dn986743\(v=vs.85\).aspx#D3D12_RESOURCE_HEAP_TIER_1) support
## Reserved Resources
- Reserved Resources are created without specifying a heap to place the resource in
- Reserved Resources are created using the `ID3D12Device::CreateReservedResource` function
- before a reserved resource can be used, it must be mapped to a heap using a `ID3D12CommandQueue::UpdateTileMappings` function
- They can be created such that they are larger than that can fit in a single heap. Portions of the reserved resource can be mapped (and unmapped) using one or more heaps residing in physical GPU memory
- Very useful for rendering techniques that use sparse voxel octrees without exceeding GPU memory budgets

# Pipeline State Objects (PSO's)
- The PSO contains most of the state that is required to configure the rendering (or compute) pipeline. It contains the following information
	- Shader (VS/PS/DS/HS/GS)
	- Input layout
	- Primitive topology
	- Blend state
	- Rasterizer state
	- Depth-Stencil state
	- Depth-Stencil format
	- Number of render targets and render target formats
	- Multi-sample description
	- Stream output buffer description
	- Root Signature
- There are a few additional parameters that need to be set outside the pipeline state
	- Vertex and index buffer
	- Stream output buffer
	- Render targets
	- Descriptor heaps
	- Shader parameters (constant buffers, read-write buffers, read-write textures)
	- Viewports
	- Scissor rectangle's
	- Constant blend factor
	- Stencil reference value
	- Primitive topology and adjacency information

# Root Signature
- A root signature is similar to a C++ function signature
	- It defines the parameters that are passed to the shader pipeline
- The values that are bound to pipeline are called **root arguments**
	- The arguments are passed to the shader can change without changing the root signature parameters

> A **root signature** is so similar to the concept of a **function signature** that we can call it a **shader signature**
 
- The root parameters in the root signature not only define the type of the parameters that are expected in the shader, they also define the shader registers and register spaces to bind the arguments to in the shader.

## Shader Register and Register Spaces
- Shader parameters must be bound to a register
- E.g. 
	- Constant buffers are bound to `b` registers (`b0` to `bN`)
	- Shader resources views are bound to `t` registers (`t0` to `tN`)
	- Unordered access views are bound to `u` registers (`u0` to `uN`)
	- Texture samplers are bound to `s` registers (`s0` to `sN`)
- Differences between DX11 and DX12 regarding binding slots
	- Different resources could be bound to the same register slot if they were used in different shader stages of the rendering pipeline. For example, a constant buffer could be bound to register b0 in the vertex shader and a different constant buffer could be bound to register b0 in the pixel shader without causing overlap.
	- DirectX 12 root signatures require that all shader parameters be defined for all stages of the rendering pipeline and registers used across different shader stages may not overlap. In order to provide a work-around for legacy shaders that may violate this rule, Shader Model 5.1 introduces **register spaces**. Resources can overlap register slots as long as they don’t also overlap register spaces

> **IT IS VERY IMPORTANT TO UNDERSTAND THE SHADER REGISTER AND REGISTER SPACE overlapping rules**

## Root Signature Parameters
- A root signature can contain any number of parameters. They are one of the following types:
	- `D3D12_ROOT_PARAMTER_TYPE_32BIT_CONSTANTS`: 32-bit root constants
	- `D3D12_ROOT_PARAMETER_TYPE_CBV`: Inline CBV descriptor
	- `D3D12_ROOT_PARAMETER_TYPE_SRV`: Inline SRV descriptor
	- `D3D12_ROOT_PARAMETER_TYPE_UAV`: Inline UAV descriptor
	- `D3D12_ROOT_PARAMETER_TYPE_DESCRIPTOR_TABLE`: Descriptor table

### 32-Bit Constants
- Constant buffer data can be passed to the shader without the need to create a constant buffer resource by using 32-bit constants
	- Dynamic indexing into the constant buffer data is not supported for constant data that is stored in the root signature space
### Inline Descriptors
- Descriptors can be placed directly in the root signature without requiring a descriptor heap
- Only CBV and buffer resources (SRV, UAV) resources containing 32-bit (`float`, `uint` or `sint`) components can be accessed using inline descriptors in the root signature. Inline UAV descriptors for buffer resources cannot contain counters
	- 