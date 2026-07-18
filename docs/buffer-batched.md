# Batched Buffer Read/Write Extension

## Summary

This RFC proposes extending the Luau `buffer` library to support batched read and write operations for both byte-based and bit-based data access. By allowing functions like `buffer.readu32` and `buffer.writeu16` to accept an optional count or variadic arguments, developers can read multiple values at once into `...number` or write multiple values in a single call. This reduces verbosity, minimizes Luau/C boundary crossings for improved performance (especially in interpreted execution), and enables elegant type conversion patterns while maintaining the buffer's raw, unopinionated nature.

## Motivation

Working closely with buffers often requires verbose loops or repeated function calls, particularly when populating vectors or other constructors that accept `...number`. Each individual call crosses the Luau/C boundary, which incurs significant overhead and limits performance gains in interpreted contexts. Batched operations would drastically reduce these crossings, offering a low-hanging fruit for major performance improvements across the language. Additionally, batched writes enable concise type truncation and conversion (e.g., f64 to u16) without manual bit manipulation or intermediate variables. This RFC builds on prior concepts but focuses strictly on `buffer` and `number` types to minimize the implementation surface and maximize ergonomic benefits for existing Luau APIs and user-provided functions that accept variadic numbers.

## Design

### Byte-Based Batched Reads
`buffer.read*` would gain an optional `count` parameter. The type suffix indicates the byte size per value (u8=1, u16=2, u32=4, f32=4, f64=8). The index advances by the number of bytes `buffer.read*` reads per index.

- **Current:** `buffer.read*(buffer: buffer, index: number) -> number`
- **Proposed:** `buffer.read*(buffer: buffer, index: number, count: number?) -> ...number`

Batched reads allow direct forwarding of return values into constructors or any Luau function accepting `...number`.

Example: Consolidating the need for type-specific buffer read/writes originating from [the luau team's RFC](https://github.com/luau-lang/rfcs/pull/198):
Current Luau capabilities:
```luau
local LUAU_VECTOR_SIZE = pcall(function() return vector.one.w end) and 4 or 3
local buf_BytesPerVector = LUAU_VECTOR_SIZE * 4
local buf_VectorCount = 128

-- Function needs to retain logic for correct indexing, and consideration of vector dimension count set by host.
local function buf_ReadVec(buf: buffer, i: number): vector
  return
    buffer.readf32(buf, i),
    buffer.readf32(buf, i + 4),
    buffer.readf32(buf, i + 8),
    if LUAU_VECTOR_SIZE == 4 then
      buffer.readf32(buf, i + 12)
    else
      nil
)

local function doThing(buf: buffer)
  for vIdx = 1, buffer.len(buf), buf_BytesPerVector do
    local v = vector.create(buf_ReadVec(buf, vIdx * buf_BytesPerVector))
    -- ...
  end
end

doThing(buffer.create(buf_VectorCount * buf_BytesPerVector))
```

Proposed batch approach:
```luau
local LUAU_VECTOR_SIZE = pcall(function() return vector.one.w end) and 4 or 3
local buf_BytesPerVector = LUAU_VECTOR_SIZE * 4

-- Logic is expressed more succinctly with lower risk of user error due to less code to consider

local function doThing(buf: buffer)
  for vIdx = 1, buffer.len(buf), buf_BytesPerVector do
    local v = vector.create(buffer.readf32(buf, vIdx, LUAU_VECTOR_SIZE))
    -- ...
  end
end

doThing(buffer.create(buf_VectorCount * buf_BytesPerVector))
```

### Byte-Based Batched Writes
`buffer.write*` functions would accept variadic arguments. Each argument is written sequentially, advancing the index by the number of bytes `buffer.write*` writes per index. This enables elegant truncation of larger number types to smaller ones.

- **Current:** `buffer.write*(buffer: buffer, index: number, value: number) -> ()`
- **Proposed:** `buffer.write*(buffer: buffer, index: number, ...number?) -> ()`

**Example: Convert buffer u32 numbers to buffer u8 numbers, with versatility on buffer choice and index**
*Current Luau capabilities:*
```luau
local function u32_to_u8(from_b: buffer, to_b: buffer, from_i: number, to_i: number, from_n: number)
  local to_cursor = to_i
  local from_cursor = from_i
  for i = 1, math.max(from_n or 1, 1) do
    buffer.writeu8(to_b, to_cursor, buffer.readu32(from_b, from_cursor))
    to_cursor += 1 -- to_buf is u8, need to increment by 1
    from_cursor += 4 -- from_buf is u32, need to increment by 4
  end
end
```

*Proposed batched approach:*
```luau
local function u32_to_u8(from_b: buffer, to_b: buffer, from_i: number, to_i: number, from_n: number)
  buffer.writeu8(to_buf, to_i, buffer.readu32(from_buf, from_i, from_n))
end
```

### Bit-Based Batched Operations (Optional)
`buffer.readbits` and `buffer.writebits` could similarly support batching, where the index is tracked in bits rather than bytes. For example:
`buffer.readbits(buffer: buffer, index: number, width: number, count: number?) -> ...number`

This allows granular access to sequentially contiguous data at the bit level. However, due to dynamic bit-shifting and alignment calculations, this may introduce performance overhead. If bit-based batching proves too costly or acts as a blocker for the core proposal, it can be omitted entirely, with developers continuing to use `bit32` for bitpacking needs.

### Emergent Patterns
Combining extended read/write behaviors enables seamless pipelines, such as reading `...number` inputs as f64 and immediately writing them to a buffer at a smaller size, or converting between different numeric representations without intermediate variables or loops.

## Drawbacks

- **Out-of-Bounds Risks:** Batched operations increase the surface area for index/count mismatches, potentially reading or writing past buffer boundaries. Since buffers are intentionally unopinionated raw data containers, the library will not add heavy safety checks, leaving bounds validation to the developer or relying on runtime error handling.
- **Performance Overhead (Bit-Based):** Bit-based batching requires dynamic bit-shifting, masking, and alignment calculations per value, which may negate performance gains compared to byte-based operations or existing `bit32` utilities.
- **API Complexity:** Adding variadic/count parameters changes function signatures and requires clear documentation to distinguish between byte-indexed and bit-indexed modes, potentially increasing the learning curve for new developers.

## Alternatives

- **Status Quo:** Developers continue writing verbose loops or chaining single-value calls, accepting higher fastcall overhead, reduced ergonomics, and slower interpreted execution.
- **Incremental Type-Specific RFCs:** Creating separate batched functions for each primitive type would bloat Luau's standard library, increase maintenance burden, and cost more fastcall allocations.
- **Omit Bit-Based Batching:** Exclude `readbits`/`writebits` batching entirely to unblock the byte-based portion of the proposal. This keeps the API focused, preserves performance guarantees, and leaves `bit32` as the standard tool for bitpacking.
- **Manual Bit Manipulation:** Continue relying on `bit32` and manual loops for type conversion and bitpacking, which is more verbose, error-prone, and less performant than native batched buffer operations.
