# BitNet Gemmini Quick Start Guide

## Building BitNet-Optimized Gemmini

### 1. Generate Verilog (for simulation or ASIC flow)
```bash
cd /ssd_scratch/chipyard
source env.sh
cd sims/verilator
make CONFIG=BitNetGemminiRocketConfig verilog
```

### 2. Build Verilator Simulator
```bash
cd /ssd_scratch/chipyard/sims/verilator
make CONFIG=BitNetGemminiRocketConfig
```

### 3. Build VCS Simulator (if available)
```bash
cd /ssd_scratch/chipyard/sims/vcs
make CONFIG=BitNetGemminiRocketConfig
```

## Configuration Parameters

The BitNet configuration (`GemminiConfigs.bitnetConfig`) includes:

```scala
weightType = SInt(2.W)              // 2-bit ternary weights
spatialArrayWeightType = SInt(2.W)  // Spatial array uses 2-bit weights
dataflow = Dataflow.WS              // Weight-stationary dataflow
has_ternary_weights = true          // Enable ternary optimization
sp_capacity = CapacityInKilobytes(128)
acc_capacity = CapacityInKilobytes(64)
// ... other parameters inherited from defaultConfig
```

## Weight Format

### Encoding Ternary Weights
Weights must be encoded as 2-bit signed integers:
- `-1` = `0b11` (two's complement)
- `0`  = `0b00`
- `1`  = `0b01`

### Example Weight Matrix Encoding
```c
// Example: 4x4 ternary weight matrix
elem_t weights[4][4] = {
    { 1, -1,  0,  1},  // Row 0
    {-1,  1,  1,  0},  // Row 1
    { 0, -1, -1,  1},  // Row 2
    { 1,  0,  1, -1}   // Row 3
};
```

## Software Usage

### 1. Include Gemmini Header
```c
#include "gemmini.h"
```

### 2. Configure Gemmini
```c
// Set weight-stationary dataflow
gemmini_config_ex(WS, 0, 0);

// Flush Gemmini
gemmini_flush(0);
```

### 3. Load Ternary Weights
```c
// Load weights to scratchpad
// Weights should already be in ternary format {-1, 0, 1}
gemmini_config_ld(WEIGHT_STRIDE);
gemmini_mvin(weights_addr, weight_spad_addr);
```

### 4. Perform Matrix Multiplication
```c
// Standard Gemmini matmul commands work with ternary weights
gemmini_config_ex(WS, 0, 0);
gemmini_preload(bias_addr, output_addr);
gemmini_compute_accumulated(input_addr, weight_addr);
```

## Hardware Operation

### Normal Weights (has_ternary_weights = false)
```
MAC: output = accumulator + (input × weight)
```

### Ternary Weights (has_ternary_weights = true)
```
if weight == 1:
    output = accumulator + input
elif weight == -1:
    output = accumulator - input
else:  # weight == 0
    output = accumulator
```

## Performance Considerations

### Optimal Use Cases
1. **BitNet LLM Inference**: Designed specifically for BitNet 1.58b models
2. **Weight-Stationary Dataflow**: Best performance with WS dataflow
3. **Large Batch Sizes**: Amortizes weight loading overhead

### Memory Bandwidth
- Ternary weights use 2 bits vs 8 bits (4× reduction)
- More weights fit in scratchpad
- Reduced memory bandwidth requirements

### Throughput
- Same cycle count as standard Gemmini for matmul
- Power savings from eliminated multipliers
- Potentially higher frequency due to simpler logic

## Verification

### 1. Functional Tests
```bash
cd /ssd_scratch/chipyard/generators/gemmini/software/gemmini-rocc-tests
# Create BitNet-specific tests
# Run on simulator
spike --extension=gemmini pk your_bitnet_test
```

### 2. Hardware Simulation
```bash
cd /ssd_scratch/chipyard/sims/verilator
./simulator-chipyard-BitNetGemminiRocketConfig your_test.riscv
```

## Troubleshooting

### Issue: Incorrect Results
- **Check weight encoding**: Ensure weights are properly encoded as -1, 0, 1
- **Verify dataflow**: BitNet config uses WS dataflow by default
- **Check matrix dimensions**: Must align with tile/mesh dimensions

### Issue: Compilation Errors
- **Ensure all files are updated**: Run `make clean` before rebuilding
- **Check Scala version**: Chisel requires specific Scala versions
- **Verify imports**: Ensure all necessary imports are present

### Issue: Performance Not as Expected
- **Profile with counters**: Use Gemmini performance counters
- **Check memory access patterns**: Optimize for spatial locality
- **Verify scratchpad usage**: Ensure weights/activations fit in scratchpad

## Example Application

### BitNet Matrix Multiplication
```c
#include "gemmini.h"

#define DIM 16

int main() {
    // Allocate matrices (using 2-bit ternary weights)
    elem_t A[DIM][DIM];  // Input activations (8-bit)
    elem_t W[DIM][DIM];  // Ternary weights (2-bit: -1, 0, 1)
    elem_t C[DIM][DIM];  // Output (8-bit)
    
    // Initialize with ternary weights
    for (int i = 0; i < DIM; i++) {
        for (int j = 0; j < DIM; j++) {
            W[i][j] = (i + j) % 3 - 1;  // -1, 0, or 1
        }
    }
    
    // Configure Gemmini for ternary weights
    gemmini_flush(0);
    gemmini_config_ex(WS, 0, 0);
    
    // Perform matrix multiplication
    tiled_matmul_auto(DIM, DIM, DIM,
                      (elem_t*)A, (elem_t*)W, NULL, (elem_t*)C,
                      DIM, DIM, DIM, DIM,
                      MVIN_SCALE_IDENTITY, MVIN_SCALE_IDENTITY,
                      ACC_SCALE_IDENTITY,
                      NO_ACTIVATION, 1, 0, 0,
                      false, false, false, false, false,
                      0);
    
    gemmini_fence();
    return 0;
}
```

## Additional Resources

- **Gemmini Documentation**: `/ssd_scratch/chipyard/generators/gemmini/README.md`
- **BitNet Paper**: "BitNet: Scaling 1-bit Transformers for Large Language Models"
- **Chipyard Documentation**: https://chipyard.readthedocs.io/
- **Implementation Details**: `BITNET_IMPLEMENTATION.md`
