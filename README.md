# VanitySearch-Bitrack with Optimization for BTC Puzzle

This project is an optimized version of VanitySearch tailored for finding Bitcoin vanity addresses, specifically targeting the BTC Puzzle. It leverages the power of CUDA-enabled GPUs and incorporates several key modifications to significantly enhance performance and usability.

## Project Structure and Codebase Overview

The project is written primarily in C++ and CUDA C++, with a clear separation of concerns across different files and directories. Here's a breakdown of the main components:

*   **`main.cpp`**:
    *   This is the entry point of the application.
    *   It handles command-line argument parsing (`-v`, `-gpuId`, `-batchSize`, `-i`, `-o`, `-start`, `-range`, `-m`, `-stop`).
    *   Initializes the `Timer` and `Secp256K1` libraries.
    *   Sets up the search parameters based on user input, including the key space range (`-start`, `-range`).
    *   Reads target addresses or prefixes from an input file (`-i`) or directly from the command line.
    *   Creates an instance of the `VanitySearch` class to manage the search process.
    *   Launches a separate thread (`monitorKeypress`) to listen for user input (specifically 'p' to pause/resume).
    *   Starts the main search loop by calling `v->Search()`.
    *   Manages the pause/resume functionality based on the `Pause` atomic variable.
    *   Prints initial and final statistics.

*   **`Vanity.h` / `Vanity.cpp`**:
    *   Defines the `VanitySearch` class, which orchestrates the entire vanity address search.
    *   Manages the list of target addresses/prefixes and an efficient lookup table (`addresses`, `usedAddress`, `usedAddressL`) for fast checking of generated addresses.
    *   The `initAddress` function parses input addresses and prepares them for the lookup table, handling different address types (P2PKH, P2SH, Bech32) and distinguishing between full addresses and prefixes.
    *   The `Search` method is responsible for launching and managing the GPU search threads.
    *   The `FindKeyGPU` method is executed by each GPU thread. It initializes the `GPUEngine` for a specific GPU, sets the starting keys for the GPU threads using the optimized `getGPUStartingKeys` function, launches the CUDA kernel, and processes the results returned from the GPU.
    *   `getGPUStartingKeys` implements the optimized batch key generation using ECC addition and batch modular inverse to efficiently compute the starting public keys for a batch of private keys.
    *   `checkAddr` and `checkAddrSSE` are used to check if a generated hash160 matches any of the target addresses/prefixes in the lookup table.
    *   `checkPrivKey` verifies a found address by computing the public key from the corresponding private key and comparing the generated address.
    *   `output` handles writing found keys to the console and an output file.
    *   `PrintStats` displays real-time statistics about the search progress.

*   **`GPU/GPUEngine.h` / `GPU/GPUEngine.cu`**:
    *   Defines the `GPUEngine` class, which serves as the interface between the CPU code and the CUDA (GPU) kernels.
    *   Handles CUDA device selection and initialization.
    *   Manages memory allocation and data transfer between the host (CPU) and device (GPU). This includes allocating memory for input keys, target addresses/lookup tables, and the output buffer for found keys. Pinned memory is used for efficient data transfer.
    *   `SetAddress` and `SetPattern` transfer the target addresses/lookup tables to the GPU.
    *   `SetKeys` transfers the initial public keys for each GPU thread to the device.
    *   `Launch` executes the CUDA kernel (`comp_keys`) on the GPU and retrieves the results from the output buffer. It also includes logic to handle potential overflow of the output buffer and reduce the batch size if needed.
    *   `GPU/GPUEngine.cu` contains the CUDA kernel `comp_keys`, which is the core of the GPU computation. This kernel runs on multiple threads in parallel on the GPU. Each thread takes its assigned starting public key and iteratively computes the public keys for subsequent private keys within its assigned range using highly optimized elliptic curve operations (point addition and doubling) implemented in CUDA.
    *   For each computed public key, the kernel calculates the corresponding Bitcoin address hash (RIPEMD160 of the SHA256 of the public key) and checks if the initial part of the hash (or the full hash for full address search) matches any of the target addresses/prefixes stored in the GPU's lookup table.
    *   If a match is found, the relevant information (thread ID, increment, endomorphism flag, hash) is stored in an output buffer on the GPU.
    *   After the kernel completes its execution for a batch, the results from the output buffer are transferred back to the CPU.

*   **`SECP256k1.h`**:
    *   Defines the `Secp256K1` class, which encapsulates the operations related to the secp256k1 elliptic curve used in Bitcoin.
    *   Provides methods for computing public keys from private keys (`ComputePublicKey`), adding and doubling points on the curve (`Add`, `Double`, `AddDirect`, `DoubleDirect`), getting hash160 from public keys (`GetHash160`), and generating different types of Bitcoin addresses (P2PKH, P2SH, Bech32) (`GetAddress`).
    *   Includes methods for handling private keys, such as decoding WIF format (`DecodePrivateKey`) and getting the WIF representation (`GetPrivAddress`).
    *   Contains the generator point `G` and the curve order `order`.
    *   (Note: The implementation details for these methods are likely in a separate `.cpp` file, which was not found in the provided file list).

*   **`Point.h`**:
    *   Defines the `Point` class, representing a point on the elliptic curve in Jacobian coordinates (x, y, z).
    *   Includes methods for initialization, checking for the point at infinity (`isZero`), equality comparison, setting point values, reducing coordinates, and converting to a string representation.

*   **`Int.h` / `Int.cpp`**:
    *   Defines the `Int` class, a fixed-size big integer class capable of handling the large numbers (256-bit) required for private keys and elliptic curve arithmetic.
    *   Provides a wide range of arithmetic operations (addition, subtraction, multiplication, division, shifting) and modular arithmetic operations (GCD, Mod, ModInv, Montgomery Multiplication, ModAdd, ModSub, ModMul, ModSquare, ModCube, ModDouble, ModExp, ModNeg, ModSqrt).
    *   Includes specific modular arithmetic functions optimized for the secp256k1 field and order (`ModMulK1`, `ModSquareK1`, `ModMulK1order`, `ModAddK1order`, `ModSubK1order`, `ModNegK1order`, `ModPositiveK1`).
    *   Includes methods for setting and getting values in different bases (decimal, hexadecimal, binary) and from byte arrays.
    *   Handles signed and unsigned operations.

*   **`Base58.h` / `Base58.cpp`**:
    *   Provides functions for encoding and decoding data using the Base58 encoding scheme, which is commonly used for Bitcoin addresses and private keys in Wallet Import Format (WIF).
    *   Handles leading zeros correctly during encoding and decoding.

*   **`Bech32.h` / `Bech32.cpp`**:
    *   Provides functions for encoding and decoding data using the Bech32 encoding scheme, used for SegWit addresses (P2WPKH and P2WSH).
    *   Includes functions for converting between 8-bit and 5-bit data representations and calculating the Bech32 checksum.

*   **`hash/` directory**:
    *   Contains implementations of cryptographic hash functions, including SHA256, SHA512, and RIPEMD160. These are essential for generating Bitcoin addresses from public keys.

*   **`GPU/` directory (Supporting GPU files)**:
    *   Includes various header files (`GPUBase58.h`, `GPUCompute.h`, `GPUGroup.h`, `GPUHash.h`, `GPUMath.h`, `GPUWildcard.h`, `RCGpuUtils.h`) that likely contain declarations and definitions for GPU-accelerated helper functions and data structures used by the CUDA kernel in `GPUEngine.cu` for tasks like Base58 encoding, point computations, hashing, and wildcard matching directly on the GPU.

*   **`GPU/defs.h`**:
    *   Contains definitions used by the GPU code, including `BLOCK_SIZE`, which determines the number of threads per block in the CUDA kernel.

*   **`Timer.h` / `Timer.cpp`**:
    *   Provides utility functions for precise timing and sleeping, used for performance measurement and controlling the search loop.

*   **`Random.h` / `Random.cpp`**:
    *   Provides functions for generating random numbers.

*   **`Wildcard.h` / `Wildcard.cpp`**:
    *   Provides utility functions for matching strings against patterns containing wildcards.

## How it Works

The VanitySearch-Bitrack application works by iteratively searching for private keys that correspond to Bitcoin addresses matching a given set of target addresses or prefixes. The core of the search is performed on the GPU for maximum efficiency.

1.  **Initialization:**
    *   The program starts by parsing command-line arguments to configure the search (GPU to use, search range, target addresses, output file, etc.).
    *   Target addresses/prefixes are loaded and processed into an efficient lookup table structure in memory.
    *   The SECP256k1 library is initialized.
    *   A `VanitySearch` object is created to manage the search.

2.  **Key Space Management:**
    *   The total search space is defined by the `-start` and `-range` parameters, representing a range of private keys.
    *   This range is divided into smaller chunks or "batches" that can be processed efficiently by the GPU.

3.  **GPU Search Loop:**
    *   The `VanitySearch::Search` method launches one or more GPU threads (currently configured for a single GPU).
    *   Each GPU thread initializes a `GPUEngine` instance, which sets up the CUDA context for the assigned GPU.
    *   The `getGPUStartingKeys` function calculates the initial public keys for a batch of private keys. This is a critical optimization that uses batch modular inverse to speed up the setup.
    *   The `GPUEngine::SetKeys` method transfers these starting public keys to the GPU memory.
    *   The `GPUEngine::Launch` method then launches the CUDA kernel (`comp_keys`) on the GPU.
    *   The `comp_keys` kernel runs in parallel across many threads on the GPU. Each thread takes its assigned starting public key and iteratively computes the public keys for subsequent private keys within its assigned range using highly optimized elliptic curve operations (point addition and doubling) implemented in CUDA.
    *   For each computed public key, the kernel calculates the corresponding Bitcoin address hash (RIPEMD160 of the SHA256 of the public key) and checks if the initial part of the hash (or the full hash for full address search) matches any of the target addresses/prefixes stored in the GPU's lookup table.
    *   If a match is found, the relevant information (thread ID, increment, endomorphism flag, hash) is stored in an output buffer on the GPU.
    *   After the kernel completes its execution for a batch, the results from the output buffer are transferred back to the CPU.

4.  **Result Processing:**
    *   The CPU code (`VanitySearch::FindKeyGPU`) processes the found items from the GPU's output buffer.
    *   For each potential match, the full private key is reconstructed, and the corresponding public key and address are verified using the `checkPrivKey` function.
    *   If the address is confirmed to be a match, the private key (in WIF and HEX format), public address, and public key are printed to the console and saved to the output file (`VanitySearch::output`).
    *   Search statistics are updated and displayed periodically (`VanitySearch::PrintStats`).

5.  **Pause/Resume:**
    *   A separate thread monitors keyboard input. Pressing 'p' toggles the `Pause` flag.
    *   When `Pause` is true, the GPU engine is temporarily freed, and the search pauses. Pressing 'p' again resumes the search by re-initializing the GPU engine and continuing from the last processed key.

6.  **Completion:**
    *   The search continues until the entire key space range has been scanned or the user stops the program.
    *   If the `-stop` flag is used and all target addresses are found, the program will also stop early.

## Specific Modifications in this Version

*   **`GPU/defs.h`**: Modified the `PNT_GROUP_CNT` macro to 1024, increasing the number of parallel threads used per block in the CUDA kernel for enhanced performance.
*   **Optimized Starting Key Generation (in `Vanity.cpp`)**: Implemented a significantly faster method for calculating the initial public keys for each GPU thread using ECC addition and a single batch modular inverse operation. This replaces a slower method that would compute each starting key individually.
*   **`Vanity.cpp`**: Cleaned up extraneous code and ensured correct definitions for `checkPrivKey` and `updateFound` functions to resolve compilation and linking errors encountered during previous development stages. These fixes are crucial for the project to build and run correctly.
*   **Git Origin**: Updated the remote origin URL for this repository to `https://github.com/eleonoraandrea/vanity.git`.

## Git Repository

The origin git repository for this project has been updated to:
`https://github.com/eleonoraandrea/vanity.git`

## Usage

VanitySeacrh [-v] [-gpuId] [-i inputfile] [-o outputfile] [-start HEX] [-range] [-m] [-stop]

 -v: Print version

 -gpuId: GPU to use, default is 0

 -batchSize: Batch size for GPU processing (affects memory usage and performance, default is 8)

 -i inputfile: Get list of addresses/prefixes to search from specified file

 -o outputfile: Output results to the specified file

 -start start Private Key HEX

 -range bit range dimension (start -> start + 2^range)

 -m: Max number of prefixes found per kernel call (default: 262144, use multiples of 65536)

 -stop: Stop when all prefixes are found


If you want to search for multiple addresses or prefixes, insert them into the input file, one address/prefix per line.

Be careful, if you are looking for many prefixes or very long prefixes, it may be necessary to increase MaxFound using "-m". Use multiples of 65536. Increasing this value might slightly decrease the speed but can prevent found addresses from being lost.

## Examples:

Windows:

```./VanitySearch.exe -gpuId 0 -i input.txt -o output.txt -start 3BA89530000000000 -range 40```

```./VanitySearch.exe -gpuId 1 -o output.txt -start 3BA89530000000000 -range 42 1MVDYgVaSN6iKKEsbzRUAYFrYadLYZvvZ```

```./VanitySearch.exe -gpuId 0 -start 3BA89530000000000 -range 41 1MVDYgVaSN6iKKEsbzRUAYFrYJadLYZvvZ ```

Linux

```./vanitysearch -gpuId 0 -i input.txt -o output.txt -start 3BA89530000000000 -range 40```

## License

VanitySearch-Bitrack is licensed under GPLv3.

## Support

If you have any luck using this software and would like to be generous, you can send a donation to the following Bitcoin address:

`bc1qu0mcrmdwhkkwds77tz5dsdr5p0uhqw6apua90t`

Remember to pump [Kotia.cash](https://kotia.cash)!

A big thank you to JLP for the first version of VanitySearch: [https://github.com/JeanLucPons/VanitySearch](https://github.com/JeanLucPons/VanitySearch)
