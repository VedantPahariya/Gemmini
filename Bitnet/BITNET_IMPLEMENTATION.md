# BitNet Ternary Weight Support Implementation

## Overview
This implementation adds support for BitNet LLM acceleration with ternary weights {-1, 0, 1} to the Gemmini systolic array accelerator. The ternary weight optimization eliminates multiplications by replacing them with simple additions and subtractions.

## What Changed

### 1. Core PE Implementation (`PE.scala`)

**MacUnit Class:**
- Added `has_ternary_weights: Boolean = false` parameter
- Implemented conditional logic:
  - When `has_ternary_weights = true`: Uses MuxCase to select add/subtract/passthrough based on weight value
    - `weight = 1` → `output = accumulator + input`
    - `weight = -1` → `output = accumulator - input`
    - `weight = 0` → `output = accumulator` (no change)
  - When `has_ternary_weights = false`: Uses standard MAC operation `io.in_c.mac(io.in_a, io.in_b)`

**PE Class:**
- Added `has_ternary_weights: Boolean = false` parameter
- Passes parameter to MacUnit instantiation

### 2. Hierarchy Updates

**Tile.scala:**
- Added `has_ternary_weights: Boolean = false` parameter
- Passes parameter to all PE instantiations

**Mesh.scala:**
- Added `has_ternary_weights: Boolean = false` parameter
- Passes parameter to all Tile instantiations

**MeshWithDelays.scala:**
- Added `has_ternary_weights: Boolean = false` parameter
- Passes parameter to Mesh instantiation

**ExecuteController.scala:**
- Updated MeshWithDelays instantiation to pass `has_ternary_weights` from config

### 3. Configuration Support

**GemminiConfigs.scala:**
- Added `has_ternary_weights: Boolean = false` parameter to `GemminiArrayConfig` case class
- Created `bitnetConfig` preset with:
  - `weightType = SInt(2.W)` - Optimized 2-bit encoding for ternary weights
  - `spatialArrayWeightType = SInt(2.W)` - Matches weight type
  - `dataflow = Dataflow.WS` - Weight-stationary optimal for ternary
  - `has_ternary_weights = true` - Enables ternary optimization
  - `sp_capacity = CapacityInKilobytes(128)` - Adjusted scratchpad
  - `acc_capacity = CapacityInKilobytes(64)` - Adjusted accumulator

**Configs.scala:**
- Added `BitNetGemminiConfig` class for BitNet-optimized Gemmini configuration

**chipyard/GemminiConfigs.scala:**
- Added `BitNetGemminiRocketConfig` extending `BitNetGemminiConfig` for complete SoC configuration

## How to Use

### 1. BitNet Configuration
Use the `BitNetGemminiRocketConfig` when building a Chipyard SoC:

```bash
cd chipyard/sims/verilator  # or sims/vcs
make CONFIG=BitNetGemminiRocketConfig
```

### 2. Custom BitNet Configuration
You can create custom configurations by extending the base bitnetConfig:

```scala
val myBitnetConfig = GemminiConfigs.bitnetConfig.copy(
  meshRows = 32,
  meshColumns = 32,
  // other custom parameters...
)
```

### 3. Software Integration
When programming for BitNet Gemmini:
- Ensure weights are encoded as 2-bit signed integers: -1, 0, 1
- Use weight-stationary dataflow for optimal performance
- The accelerator will automatically use add/subtract instead of multiply

## Key Benefits

1. **Hardware Efficiency:**
   - Eliminates multipliers from MAC units when ternary mode is enabled
   - Reduces power consumption and area
   - Maintains high throughput for BitNet LLM inference

2. **Backward Compatibility:**
   - Default `has_ternary_weights = false` preserves all existing configurations
   - Standard Gemmini configurations remain unchanged
   - No runtime overhead for non-ternary modes

3. **Compile-Time Optimization:**
   - Hardware synthesizes only the needed logic path based on configuration
   - No runtime switching overhead between ternary and standard modes

## Implementation Details

### Ternary Weight Logic
The implementation uses Chisel's `MuxCase` to efficiently select the operation:

```scala
io.out_d := MuxCase(io.in_c, Seq(
  weight_is_pos_one -> (io.in_c + io.in_a).asTypeOf(dType),
  weight_is_neg_one -> (io.in_c - io.in_a).asTypeOf(dType)
  // weight_is_zero -> io.in_c (default case)
))
```

### Weight Encoding
- Ternary weights use 2-bit signed integer representation
- `-1` encoded as `2'b11` (two's complement)
- `0` encoded as `2'b00`
- `1` encoded as `2'b01`

## Testing Recommendations

1. **Functional Testing:**
   - Create BitNet-specific test cases in `apps/` directory
   - Verify ternary operations match reference implementations
   - Test with various matrix sizes

2. **Performance Validation:**
   - Compare cycle counts with standard integer matmul
   - Measure power consumption improvements
   - Validate throughput for target BitNet models

3. **Integration Testing:**
   - Test with actual BitNet LLM workloads
   - Verify end-to-end inference accuracy
   - Validate memory bandwidth utilization

## Files Modified

1. `/ssd_scratch/chipyard/generators/gemmini/src/main/scala/gemmini/PE.scala`
2. `/ssd_scratch/chipyard/generators/gemmini/src/main/scala/gemmini/Tile.scala`
3. `/ssd_scratch/chipyard/generators/gemmini/src/main/scala/gemmini/Mesh.scala`
4. `/ssd_scratch/chipyard/generators/gemmini/src/main/scala/gemmini/MeshWithDelays.scala`
5. `/ssd_scratch/chipyard/generators/gemmini/src/main/scala/gemmini/ExecuteController.scala`
6. `/ssd_scratch/chipyard/generators/gemmini/src/main/scala/gemmini/GemminiConfigs.scala`
7. `/ssd_scratch/chipyard/generators/gemmini/src/main/scala/gemmini/Configs.scala`
8. `/ssd_scratch/chipyard/generators/gemmini/chipyard/GemminiConfigs.scala`

## Future Enhancements

1. **Weight Encoding Optimization:**
   - Consider 1.58-bit encoding schemes from BitNet papers
   - Explore compressed weight storage formats

2. **Accumulator Optimization:**
   - Analyze maximum accumulation values for BitNet models
   - Potentially reduce accumulator bit width

3. **Dynamic Switching:**
   - Add runtime switching between ternary and standard modes (if needed)
   - Requires additional control signals and multiplexing

4. **Mixed Precision:**
   - Support mixed ternary/standard layers in the same model
   - Enable flexible deployment strategies
