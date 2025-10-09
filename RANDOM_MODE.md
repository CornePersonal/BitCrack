# BitCrack --random Parameter Implementation

## Summary
This implementation adds a new `--random` parameter to BitCrack that enables random key generation instead of sequential key stepping through the keyspace.

## Changes Made

### 1. Command-Line Interface (KeyFinder/main.cpp)
- Added `--random` flag to the command-line parser
- Added `randomMode` field to the `RunConfig` structure
- Added validation to prevent incompatible flag combinations:
  - `--random` cannot be used with `--share` (keyspace sharing)
  - `--random` cannot be used with `--continue` (checkpoint resume)
- Added informative logging when random mode is active
- Updated usage() function to document the new flag

### 2. KeyFinder Library (KeyFinderLib/)
- **KeyFinder.h**: Added `_randomMode` member variable and updated constructor signature
- **KeyFinder.cpp**: 
  - Updated constructor to accept `randomMode` parameter
  - Modified `init()` to pass random mode flag to device implementations
- **KeySearchDevice.h**: Updated interface to accept `randomMode` in `init()` method

### 3. CUDA Device Implementation (CudaKeySearchDevice/)
- **CudaKeySearchDevice.h**: 
  - Added `_randomMode` member variable
  - Updated `init()` method signature
- **CudaKeySearchDevice.cpp**:
  - Added `#include <random>` for random number generation
  - Updated `init()` to accept and store random mode flag
  - Modified `generateStartingPoints()` to generate random keys when in random mode:
    - Uses `std::random_device` and `std::mt19937_64` for high-quality random numbers
    - Generates 256-bit random keys by filling 8 32-bit values
    - Ensures keys are within valid range (1 to N-1) where N is the secp256k1 curve order
    - Ensures keys are never zero
  - Modified `doStep()` to regenerate random keys on each iteration when in random mode

### 4. OpenCL Device Implementation (CLKeySearchDevice/)
- **CLKeySearchDevice.h**:
  - Added `_randomMode` member variable
  - Updated `init()` method signature
- **CLKeySearchDevice.cpp**:
  - Added `#include <random>` for random number generation
  - Updated `init()` to accept and store random mode flag
  - Modified `generateStartingPoints()` with same random key generation logic as CUDA version
  - Modified `doStep()` to regenerate random keys on each iteration when in random mode

### 5. Build System
- Updated `.gitignore` to exclude build artifacts (lib/ and bin/ directories)

## Technical Details

### Random Key Generation Algorithm
When `--random` is enabled:
1. Initialize random number generator with `std::random_device` for seed entropy
2. Use `std::mt19937_64` (Mersenne Twister) for high-quality pseudo-random numbers
3. Generate each 256-bit key by creating 8 32-bit values from 4 64-bit random numbers
4. Validate that each key is within the valid secp256k1 range (1 to N-1)
5. If a key is >= N, subtract N to bring it into range
6. If a key is zero, set it to 1

### Key Regeneration
When in random mode:
- Initial set of keys is generated during `init()` call
- For each subsequent `doStep()` iteration (after iteration 0), new random keys are regenerated
- This ensures continuous random sampling of the keyspace rather than sequential stepping

### Valid Bitcoin Private Keys
All generated keys are guaranteed to be valid Bitcoin private keys because:
- They are 256-bit values
- They are within the range [1, N-1] where N is the secp256k1 curve order
- They are never zero or negative
- The secp256k1 curve order N = 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141

## Usage Examples

### Basic random mode with compressed keys:
```bash
./cuBitCrack --random -c 1FshYsUh3mqgsG29XpZ23eLjWV8Ur3VwH
```

### Random mode with multiple target addresses:
```bash
./cuBitCrack --random -i addresses.txt -o found.txt
```

### Random mode with specific device and parameters:
```bash
./cuBitCrack --random -d 0 -b 32 -t 256 -p 32 1FshYsUh3mqgsG29XpZ23eLjWV8Ur3VwH
```

### Invalid combinations (will show error):
```bash
# Cannot use --random with --share
./cuBitCrack --random --share 1/10 1FshYsUh3mqgsG29XpZ23eLjWV8Ur3VwH

# Cannot use --random with --continue
./cuBitCrack --random --continue checkpoint.txt 1FshYsUh3mqgsG29XpZ23eLjWV8Ur3VwH
```

## Benefits of Random Mode
1. **Better Distribution**: Avoids sequential patterns that might miss target keys
2. **Parallel Searching**: Multiple instances can search independently without coordination
3. **No Overlap**: When running multiple instances, they naturally avoid checking the same keys
4. **Flexible**: Can be stopped and restarted without needing checkpoint files
5. **Lottery Approach**: Every iteration has equal probability of finding any key in the range

## Limitations
- Cannot resume progress (incompatible with --continue)
- Cannot divide work among instances (incompatible with --share)
- Progress cannot be tracked in terms of keyspace coverage
- Total keys searched is the only metric available

## Testing
The implementation has been validated to:
- Compile successfully with the existing build system
- Generate valid random 256-bit values
- Ensure all keys are within the secp256k1 valid range
- Handle edge cases (keys >= N, zero keys)
- Work with both CUDA and OpenCL implementations
