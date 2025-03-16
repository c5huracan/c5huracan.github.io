# The Struggle to Implement Deviceless Constants in tinygrad: A Tale of Unexpected Challenges

## Introduction

Deep learning frameworks like tinygrad often make architectural decisions that become deeply embedded in the codebase. One such decision is the assignment of devices to all tensors, including constants. While this approach provides consistency, it can lead to inefficiencies in specific scenarios such as the Winograd convolution algorithm. This blog post details the challenging journey to implement "deviceless constants" in tinygrad, the unexpected roadblocks encountered, and the lessons learned.

## The Vision: Cleaner Code Generation

The goal was simple: allow constant tensors to remain deviceless until they need to interact with device-specific operations. Consider this example from the Winograd convolution implementation:

```python
from tinygrad import Tensor
N = 3
c = Tensor.stack([Tensor(i+1) for i in range(N)])
x = Tensor.rand(N)
(c * x).realize()
```

The hope was to generate cleaner kernel code like:

```
kernel void E_3(device float* data0, const device float* data1, uint3 gid [[threadgroup_position_in_grid]], uint3 lid [[thread_position_in_threadgroup]]) {
  float val0 = *(data1+0);
  float val1 = *(data1+1);
  float val2 = *(data1+2);
  *(data0+0) = val0;
  *(data0+1) = (2.0f*val1);
  *(data0+2) = (3.0f*val2);
}
```

## The Reality: Cascading Challenges

What seemed like a straightforward modification quickly turned into a series of unexpected challenges:

### 1. Pervasive Device Assumptions

The tinygrad codebase assumes devices are always present, with `None` not being a valid device value. This assumption permeates many parts of the code:

- The `unwrap` function in helpers.py asserts that values are not `None`
- UOp's device property doesn't handle `None`
- Tensor operations assume device information is always available

### 2. Method Accessibility Issues

Testing revealed that many methods, including the critical `stack` and `cat` operations, weren't accessible on Tensor instances despite being defined in the class:

```
=== Checking for specific methods ===
Has 'cat' method: False
Has 'stack' method: False

=== Testing cat method ===
Cat failed: 'Tensor' object has no attribute 'cat'
```

This unexpected behavior suggested issues with method inheritance or decorators, further complicating the implementation.

### 3. Error Cascade

Each fix attempted led to new errors:

```
Traceback (most recent call last):
  File "/home/steve/projects/tinygrad/test/qt-stack.py", line 5, in <module>
    c = first.stack(*tensors[1:])
AttributeError: 'Tensor' object has no attribute 'stack'
```

Fixing one issue revealed another:

```
AssertionError: 'Tensor' object has no attribute '_device'
```

And another:

```
File "/home/steve/projects/tinygrad/tinygrad/helpers.py", line 61, in unwrap
    assert x is not None
AssertionError
```

## Attempted Solutions

Several approaches were tried to work around these challenges:

### 1. UOp Device Property Modification

An attempt was made to modify the device property to handle None:

```python
@property
def device(self) -> str|tuple[str, ...]|None:
    return None if self._device is None else cast(str|tuple[str, ...], unwrap(self._device))
```

### 2. Tensor Initialization Updates

The initialization was updated to preserve None for device:

```python
# When handling device
if isinstance(device, str): 
    self.lazydata:UOp = data if data.device == device else data.copy_to_device(device)
elif device is None:
    self.lazydata = data  # Keep deviceless data as is
```

### 3. Operation Device Inheritance

Device inheritance was implemented in `_apply_uop`:

```python
def _apply_uop(self, fxn:Callable, *x:Tensor, **kwargs) -> Tensor:
    # Determine device: if any input has a device, use that device
    device = next((t.device for t in (self,)+x if t.device is not None), None)
    # ...
```

### 4. Alternative Implementation Attempts

When it was discovered that stack and cat methods weren't accessible, workarounds were attempted:

```python
# Direct implementation
tensors = [Tensor(i+1) for i in range(N)]
unsqueezed = [t.unsqueeze(0) for t in tensors]
first = unsqueezed[0]
# But even this failed because 'unsqueeze' wasn't accessible either
```

## Implications and Lessons Learned

This journey revealed several important insights about working with established ML frameworks:

1. **Architectural Assumptions Run Deep**: Assumptions like "all tensors have a device" can be deeply embedded in a codebase, making changes more complex than anticipated.

2. **Method Accessibility Is Not Guaranteed**: Just because a method is defined in a class doesn't mean it's accessible on instances. Decorators, metaclasses, and other Python features can affect method resolution.

3. **Framework Internals Matter**: Understanding the internal architecture of a framework is crucial before attempting modifications.

4. **Incremental Testing Is Essential**: Testing each change incrementally would have revealed issues earlier and saved time.

5. **Documentation Often Lags Implementation**: The actual behavior of the framework may differ from what documentation or code comments suggest.

## Current Status and Future Work

The implementation remains incomplete due to the unexpected challenges with method accessibility. Options for moving forward include:

1. **Framework Modification**: Making more extensive changes to tinygrad's core to properly support deviceless constants.

2. **Alternative Approaches**: Finding different ways to optimize the Winograd convolution without relying on deviceless constants.

3. **Contributing Upstream**: Working with tinygrad maintainers to address the method accessibility issues discovered.

## Conclusion

While the attempt to implement deviceless constants in tinygrad didn't succeed as planned, it provided valuable insights into the framework's architecture and the challenges of modifying established ML systems.

The key insight remains valid: constants don't inherently need a device until they interact with device-specific operations. However, implementing this insight in practice requires navigating the complex reality of framework internals, architectural assumptions, and unexpected behaviors.

This experience highlights the importance of understanding a system deeply before attempting to modify it, and the value of incremental testing when working with complex codebases. Sometimes the most valuable lessons come from the challenges faced rather than the successes achieved.
