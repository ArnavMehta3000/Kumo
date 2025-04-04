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

^dc7a1c

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