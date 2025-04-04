- During rendering the GPU will read/write to resources (e.g. back buffer/depth buffer).
- Before rendering we have to *bind* or *link* the resources to the rendering pipeline that are going to be referenced in the frame call. However GPU resources are not bound directly.
- Instead, a resource is referenced through a *descriptor* object, which can be thought of as a lightweight structure that describes the resource to the GPU.
- It behaves like a level of **indirection**. Given a descriptor, the GPU can get the actual resource data and know the required information about it.
- Binding resources is done via descriptors.
- Each stage needs its own descriptor
	- E.g. Using a texture as a render target and shader resource you need to create two resources.
- It is recommended to create descriptors at initialization time, because of *type checking* and *validation*

> A *view* is a synonym for a *descriptor*. Example a *constant buffer view* and *constant buffer descriptor* mean the same thing

# Descriptors in DX12
- Descriptors have a type, and the type implies implies how the resource will be used.

## Types of Descriptors
- **CBV/SRV/RTV** descriptors describe constant buffers, shader resources and unordered access views.
- **Sampler Descriptors** describe sampler resources used in texturing.
- **RTV Descriptors** describe render target resources
- **SRV Descriptors** describe depth/stencil resources

# Descriptor Heap
- Descriptor heaps are array of descriptors. It is the memory backing for all the descriptors of a particular type your application uses
- We need a separate type of descriptor heap for each type of descriptor
	- SRV gets its own heap, RTV gets its own heap and so on...
- Multiple heaps of the same type are also allowed
- Multiple descriptors can also refer to the same resource
	- E.g. Multiple resources referencing different subregions of a resource.