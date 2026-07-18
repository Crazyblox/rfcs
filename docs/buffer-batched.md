# Batched Buffer Read/Write Operations

## Summary

This RFC proposes extending the Luau `buffer` library to support batched read and write operations for both byte-aligned and bit-aligned data access. By allowing functions like `buffer.readu32` and `buffer.writeu16` to accept an optional count or variadic arguments, developers can read multiple values at once into `...number` or write multiple values in a single call. This reduces verbosity, minimizes Luau/C boundary crossings for improved performance (especially in interpreted execution), and enables elegant type conversion patterns while maintaining the buffer's raw, unopinionated nature.

## Motivation

Working closely with buffers often requires verbose loops or repeated function calls, particularly when populating vectors or other constructors that accept `...number`. Each individual call crosses the Luau/C boundary, which incurs significant overhead and limits performance gains in interpreted contexts. Batched operations would drastically reduce these crossings, offering a low-hanging fruit for major performance improvements across the language. Additionally, batched writes enable concise type truncation and conversion (e.g., f64 to u16) without manual bit manipulation or intermediate variables. This RFC builds on [prior concepts](https://github.com/luau-lang/rfcs/pull/198) but focuses strictly on `buffer` and `number` types to minimize the implementation surface and maximize ergonomic benefits for existing Luau APIs and user-provided functions that accept variadic numbers.

## Design

### Byte-Aligned Batched Reads
Functions like `buffer.readu32` would gain an optional `count` parameter. The type suffix indicates the byte size per value (u8=1, u16=2, u32=4, f32=4, f64=8). The index advances by `count * sizeof(type)` bytes.

- **Current:** `buffer.readu32(buffer: buffer, index: number) -> number`
- **Proposed:** `buffer.readu32(buffer: buffer, index: number, count: number?) -> ...number`

Batched reads allow direct forwarding of return values into constructors or any Luau function accepting `...number`:
```luau
local x, y, z = buffer.readu32(buf, 0, 3)
local v = vector.create(x, y, z)
-- Or directly: local v = vector.create(buffer.readu32(buf, 0, 3))
```

### Byte-Aligned Batched Writes
Functions like `buffer.writeu16` would accept variadic arguments. Each argument is written sequentially, advancing the index by `sizeof(type)` bytes per value. This enables elegant truncation of larger number types to smaller ones.

- **Current:** `buffer.writeu32(buffer: buffer, index: number, value: number) -> ()`
- **Proposed:** `buffer.writeu32(buffer: buffer, index: number, ...number?) -> ()`

**Example: Converting `...number` (f64) to u16**
*Current Luau capabilities:*
```luau
local buf = buffer.create(4)
local val1, val2 = 3.14159, 2.71828
-- Requires manual truncation or intermediate steps
buffer.writeu16(buf, 0, math.floor(val1))
buffer.writeu16(buf, 2, math.floor(val2))
```

*Proposed batched approach:*
```luau
local buf = buffer.create(4)
local val1, val2 = 3.14159, 2.71828
buffer.writeu16(buf, 0, val1, val2) -- Automatically truncates f64 inputs to u16
local a, b = buffer.readu16(buf, 0, 2) -- a=3, b=2
```

### Bit-Aligned Batched Operations (Optional)
`buffer.readbits` and `buffer.writebits` could similarly support batching, where the index is tracked in bits rather than bytes. For example:
`buffer.readbits(buffer: buffer, index: number, width: number, count: number?) -> ...number`

This allows granular access to sequentially contiguous data at the bit level. However, due to dynamic bit-shifting and alignment calculations, this may introduce performance overhead. If bit-based batching proves too costly or acts as a blocker for the core proposal, it can be omitted entirely, with developers continuing to use `bit32` for bitpacking needs.

### Emergent Patterns
Combining extended read/write behaviors enables seamless pipelines, such as reading `...number` inputs as f64 and immediately writing them to a buffer at a smaller size, or converting between different numeric representations without intermediate variables or loops.

## Drawbacks

- **Out-of-Bounds Risks:** Batched operations increase the surface area for index/count mismatches, potentially reading or writing past buffer boundaries. Since buffers are intentionally unopinionated raw data containers, the library will not add heavy safety checks, leaving bounds validation to the developer or relying on runtime error handling.
- **Performance Overhead (Bit-Aligned):** Bit-based batching requires dynamic bit-shifting, masking, and alignment calculations per value, which may negate performance gains compared to byte-aligned operations or existing `bit32` utilities.
- **API Complexity:** Adding variadic/count parameters changes function signatures and requires clear documentation to distinguish between byte-indexed and bit-indexed modes, potentially increasing the learning curve for new developers.

## Alternatives

- **Status Quo:** Developers continue writing verbose loops or chaining single-value calls, accepting higher fastcall overhead, reduced ergonomics, and slower interpreted execution.
- **Incremental Type-Specific RFCs:** Creating separate batched functions for each primitive type would bloat Luau's standard library, increase maintenance burden, and cost more fastcall allocations.
- **Omit Bit-Aligned Batching:** Exclude `readbits`/`writebits` batching entirely to unblock the byte-aligned proposal. This keeps the API focused, preserves performance guarantees, and leaves `bit32` as the standard tool for bitpacking.
- **Manual Bit Manipulation:** Continue relying on `bit32` and manual loops for type conversion and bitpacking, which is more verbose, error-prone, and less performant than native batched buffer operations.
