- GPU and CPU need to be synchronized in DX12 unlike in DX11 where that is handled by the drivers
- It is very important when performing resource management
	- Resources cannot be freed if they are currently being referenced in a command list that is being executed on the command queue
	- It is only safe to release those resource after the command queue has finished executing any command that is referencing those resources

# Fences

- A Fence object is used to synchronize commands issued to the **Command Queue**.
- It stores a single value that indicates the last value that was used to **signal** the fence
- A single fence object *can* be used with multiple command queues, but it is *not reliable* to ensure the proper synchronization of commands across command queues
	- Advised to create at least *once* fence object for each command queue
- Multiple command queues can wait on a fence to reach a specific value, but the fence should only be allowed to be signaled from a *single command queue*
- The application must also track a **fence value** that is used to signal the fence

# Command List

- A command list is used to issue copy, compute (dispatch) and draw commands
- In DX12 commands are issued to the **command list** but are *not* executed immidiately
	- All commands in DX12 are deferred -> the commands in a command list are only run on the GPU after they have been executed on the command queue

# Command Queue

- A command queue in DX12 has a very simple interface
- Most common function that is used is `ExecuteCommandLists`
- This is the most common style of using a command queue
```cpp
method IsFenceComplete( _fenceValue )
    return fence->GetCompletedValue() >= _fenceValue
end method

method WaitForFenceValue( _fenceValue )
    if ( !IsFenceComplete( _fenceValue )
        fence->SetEventOnCompletion( _fenceValue, fenceEvent )
        WaitForEvent( fenceEvent )
    end if
end method

method Signal
    _fenceValue <- AtomicIncrement( fenceValue )
    commandQueue->Signal( fence, _fenceValue )
    return _fenceValue
end method

method Render( frameID )
    _commandList <- PopulateCommandList( frameID )
    commandQueue->ExecuteCommandList( _commandList )
    
    _nextFrameID <- Present()
    fenceValues[frameID] = Signal()  // Signal end og current frame
    WaitForFenceValue( fenceValues[_nextFrameID] )
    
    frameID <- _nextFrameID
end method
```
- `IsFenceComplete`: Check to see if the fence’s completed value has been reached.
- `WaitForFenceValue`: Stall the CPU thread until the fence value has been reached.
- `Signal`: Insert a fence value into the command queue. The fence used to signal the command queue will have it’s completed value set when that value is reached in the command queue.
- `Render`: Render a frame. Do not move on to the next frame until that frame’s previous fence value has been reached.
- It is important to understand that each command queue must track it’s own fence and fence value. DirectX 12 defines three different command queue types:
	1. **Copy**: Can be used to issue commands to copy resource data (CPU -> GPU, GPU -> GPU, GPU -> CPU).
	2. **Compute**: Can do everything a **Copy** queue can do and issue compute (dispatch) commands.
	3. **Direct**: Can do everything a **Copy** and a **Compute** queue can do and issue draw commands.

# Command Allocators

- A command allocator is the backing memory used by a Command List
- When creating a command allocator, the type must be specified
- It provides no functionality and must only be accessed indirectly through a command list
- A command allocator can only be used by a single command list at a time
	- It can be reused after the command that were recorded into the command list have finished executing **on the GPU**
	- A fence can be used to check if a the GPU commands have finished executing on the GPU
- The memory allocated by the command allocator is reclaimed using the `Reset` function
- In order to achieve maximum frame-rates for the application, one command allocator per “in-flight” command list should be created.

# Understanding Synchronization - Simplified

## What is a Fence?

A fence is essentially a synchronization primitive that allows the CPU to know when the GPU has completed a particular set of commands. Unlike in DX11 where synchronization was largely handled by the runtime, DX12 requires you to manage this manually.

### How Fences Work

1. **Fence Object**: First, you create an `ID3D12Fence` object
2. **Fence Values**: A fence has a value that can be incremented
3. **Signaling**: When you "signal" a fence, you're asking the GPU to update the fence value when it reaches that point in command execution
4. **Waiting**: The CPU can then wait for the fence to reach a specific value

### Basic Fence Pattern

```cpp
// Create a fence
ComPtr<ID3D12Fence> fence;
device->CreateFence(0, D3D12_FENCE_FLAG_NONE, IID_PPV_ARGS(&fence));

// Current fence value we're tracking
UINT64 fenceValue = 1;

// Submit commands to the command queue
commandQueue->ExecuteCommandLists(1, commandListPtr);

// Signal the fence - this queues an instruction to set the fence value when GPU reaches this point
commandQueue->Signal(fence.Get(), fenceValue);

// Check if GPU has completed work
if (fence->GetCompletedValue() < fenceValue)
{
    // Create an event to wait on
    HANDLE eventHandle = CreateEvent(nullptr, false, false, nullptr);
    
    // Have the fence trigger the event when it reaches our value
    fence->SetEventOnCompletion(fenceValue, eventHandle);
    
    // Wait for the event to be triggered
    WaitForSingleObject(eventHandle, INFINITE);
    
    // Close the event handle
    CloseHandle(eventHandle);
}

// Increment fence value for next frame
fenceValue++;
```

### Understanding Fence Values

Think of fence values as ticket numbers:
1. When you signal a fence with value X, you're telling the GPU: "When you finish all commands submitted so far, set this fence's value to X"
2. The fence value starts at 0 by default
3. You typically increment the fence value each time you signal (1, 2, 3...)
4. The `GetCompletedValue()` method returns the last value the GPU has set

### Practical Example: Frame Synchronization

In a typical rendering loop, you might use fences to ensure you don't overwrite resources that the GPU is still using:
```cpp
struct FrameContext
{
    ComPtr<ID3D12CommandAllocator> commandAllocator;
    UINT64 fenceValue;
};

// Create frame contexts for triple buffering
const UINT FRAME_COUNT = 3;
FrameContext frameContexts[FRAME_COUNT];
UINT frameIndex = 0;
UINT64 fenceValue = 0;

// In your render loop:
void RenderFrame()
{
    // Get the frame resources for the current frame
    FrameContext& currentFrame = frameContexts[frameIndex];
    
    // Wait if the GPU hasn't finished processing this frame's commands from last time
    if (fence->GetCompletedValue() < currentFrame.fenceValue)
    {
        fence->SetEventOnCompletion(currentFrame.fenceValue, fenceEvent);
        WaitForSingleObject(fenceEvent, INFINITE);
    }
    
    // Reset command allocator and list for new commands
    currentFrame.commandAllocator->Reset();
    commandList->Reset(currentFrame.commandAllocator.Get(), nullptr);
    
    // Record commands...
    
    // Execute command list
    commandQueue->ExecuteCommandLists(1, &commandListPtr);
    
    // Signal fence with new value
    fenceValue++;
    commandQueue->Signal(fence.Get(), fenceValue);
    
    // Update frame context's fence value
    currentFrame.fenceValue = fenceValue;
    
    // Move to next frame
    frameIndex = (frameIndex + 1) % FRAME_COUNT;
}
```

### Common Scenarios for Using Fences

1. **Frame Synchronization**: Ensuring previous frames are complete before reusing resources
2. **Resource Transitions**: Making sure a resource is no longer in use before changing its state
3. **Upload Operations**: Confirming upload of data to GPU memory is complete
4. **Multi-GPU Synchronization**: Coordinating work across multiple GPUs
### Key Concepts to Remember

1. **The fence value is just a number**: It has no inherent meaning other than what you assign to it
2. **Signal means "set the fence to this value when you get here"**: It's an instruction to the GPU
3. **GetCompletedValue() tells you what the GPU has processed so far**: If less than your target value, GPU is still working
4. **SetEventOnCompletion() is non-blocking**: It just sets up the notification; WaitForSingleObject() is what actually blocks