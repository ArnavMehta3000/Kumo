## Signal

- Means - GPU says "I'm done up to here"
	- This tells the fence: "When you (the GPU) finish executing everything up to this point, set the fence's value to `value`."

## Wait

- Means - CPU waits for the GPU to catchup
	- - Check the fence value with `fence->GetCompletedValue()`
	- Or block until it reaches a value - `SetEventOnCompletion/WaitForSingleObject`

![[FenceSynchronization.png]]
![[FenceSynchronizationSteps.png]]
### Key Code Pattern

```cpp
// Submit work to GPU 
commandQueue->ExecuteCommandLists(1, &commandList); 

// Signal fence after this work completes 
currentFenceValue++; 
commandQueue->Signal(fence, currentFenceValue);

// Later: Check if work is done
if (fence->GetCompletedValue() < currentFenceValue) 
{
	// GPU not done yet, wait for completion 
	fence->SetEventOnCompletion(currentFenceValue, eventHandle); WaitForSingleObject(eventHandle, INFINITE); 
}
```