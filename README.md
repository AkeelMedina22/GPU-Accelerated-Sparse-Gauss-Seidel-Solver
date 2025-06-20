# GPU-Accelerated Gauss-Seidel Sparse Iterative Solver using Graph Coloring

This project explores the acceleration of the Gauss-Seidel iterative method for solving sparse linear systems ($Ax = b$) using GPU parallelism, enabled by graph coloring techniques. The primary goal is to leverage the parallel processing capabilities of GPUs to speed up the traditionally sequential Gauss-Seidel algorithm. Different approaches to graph coloring and GPU implementation are investigated and benchmarked using matrices from the SuiteSparse collection, primarily "flowmeter0".

## Overview

The Gauss-Seidel method is an iterative technique used to solve systems of linear equations. In its standard form, it updates components of the solution vector $x$ sequentially, which poses a challenge for parallelization due to data dependencies. Graph coloring can be used to partition the unknowns (nodes in the graph representation of the sparse matrix) into sets (colors) such that unknowns within the same set are independent of each other. This allows for parallel updates of all unknowns belonging to the same color.

This project implements and compares:
1.  A purely sequential CPU-based Gauss-Seidel solver as a baseline.
2.  A CPU-based randomized graph coloring algorithm (inspired by Brooks-Vizing, as used by Vivace and provided by Eric Arneback) combined with a GPU-based Gauss-Seidel kernel.
3.  A GPU-based graph coloring approach using NVIDIA's cuSPARSE library (`cusparseScsrcolor`) followed by a custom GPU Gauss-Seidel kernel.
4.  An experimental approach using cuSPARSE coloring and attempting to decompose the matrix into sub-matrices per color, applying Sparse Matrix-Vector multiplication (SpMV) for updates (noted as incomplete/not fully functional in the notebook).

The "flowmeter0" matrix from the SuiteSparse collection is primarily used for benchmarking.

## Features

* **Sparse Matrix Handling**: Loads sparse matrices in Matrix Market format (`.mtx`) using `ssgetpy`.
* **CPU Gauss-Seidel**: Baseline implementation for comparison.
* **Custom CPU Graph Coloring**: A randomized graph coloring algorithm.
* **cuSPARSE Integration**:
    * Utilizes `cusparseScsrcolor` for efficient graph coloring on the GPU.
    * Converts dense matrices to CSR (Compressed Sparse Row) format using functions like `cusparseSdense2csr` (though deprecated, it was used in the project).
* **GPU Kernels**:
    * Custom CUDA kernel for the parallel Gauss-Seidel update step based on colored partitions.
    * Experimental use of cuSPARSE SpMV for Gauss-Seidel updates.
* **Benchmarking**: Compares the runtime performance of different coloring algorithms and solver implementations.

## Methods Explored

### 1. CPU Gauss-Seidel
A standard sequential implementation to serve as a reference. The matrix is processed row by row.

### 2. CPU Randomized Graph Coloring + GPU Gauss-Seidel
* **Graph Coloring (CPU)**: A custom implementation of a randomized graph coloring algorithm. Nodes (matrix rows/columns) are colored such that no two adjacent nodes (dependencies) share the same color.
* **Gauss-Seidel (GPU)**: After coloring, nodes of the same color are processed in parallel on the GPU. Iterations proceed color by color.

### 3. cuSPARSE Graph Coloring + Custom GPU Gauss-Seidel Kernel
* **Graph Coloring (cuSPARSE)**: Leverages `cusparseScsrcolor` from the NVIDIA cuSPARSE library for fast and efficient graph coloring. The matrix is first converted to CSR format.
* **Gauss-Seidel (GPU)**: A custom CUDA kernel updates the solution vector $x$. Each color forms a parallelizable set of updates. Kernel launches are performed sequentially for each color. The number of threads per block is dynamically adjusted based on matrix size (e.g., by finding the next largest power of two up to a limit like 1024).

### 4. cuSPARSE Graph Coloring + SpMV-based GPU Gauss-Seidel (Experimental)
* **Graph Coloring (cuSPARSE)**: Same as above.
* **Gauss-Seidel (GPU)**: This approach attempts to decompose the reordered (colored) matrix into several sparse sub-matrices, one for each color. The update for the variables corresponding to a color is then performed using cuSPARSE SpMV. This method was noted as incomplete and not fully validated in the notebook.

## Dependencies

* Python 3
* `ssgetpy` (for downloading SuiteSparse matrices)
* `nvcc4jupyter` (for compiling CUDA code in Jupyter Notebook)
* NVIDIA CUDA Toolkit (for `nvcc` compiler and cuSPARSE library)
* A C++ compiler (e.g., `g++`)

You can install the Python dependencies using pip:
```bash
pip install ssgetpy --quiet
pip install git+[https://github.com/andreinechaev/nvcc4jupyter.git](https://github.com/andreinechaev/nvcc4jupyter.git) --quiet
```
Ensure the CUDA Toolkit is installed and nvcc is in your system's PATH.

## Usage

The project is structured as a Jupyter Notebook (`multicolored_gauss_seidel (3).ipynb`).
1.  Ensure all dependencies are installed and the CUDA environment is configured.
2.  Open and run the Jupyter Notebook.
3.  The notebook cells will:
    * Install necessary Python packages.
    * Load the `nvcc_plugin`.
    * Download a test sparse matrix (e.g., `flowmeter0`) from the SuiteSparse collection.
    * Compile and run the C++/CUDA code for each implemented method.

Each major section in the notebook corresponds to one of the methods described above. Output including timing and number of colors/partitions will be displayed below the respective code cells.

## Results Summary (Illustrative based on notebook outputs)

* **CPU Gauss-Seidel**: For the `flowmeter0` matrix (N=9669), the CPU version took approximately 30396 ms for 20 iterations.
* **CPU Graph Coloring (Custom)**: For a different matrix (`t2dal_a`, N=4257), the custom graph coloring execution was manually interrupted in the notebook, indicating potential performance challenges for larger matrices.
* **cuSPARSE Graph Coloring (`cusparseScsrcolor`)**:
    * For `flowmeter0` (N=9669), this coloring method took around 430 ms to 517 ms.
    * It produced 16 colors for the `flowmeter0` matrix.
* **GPU Gauss-Seidel (with cuSPARSE coloring)**:
    * The custom kernel version for `flowmeter0` (N=9669) completed 9 iterations in approximately 3483.52 ms.
    * The SpMV-based experimental version also ran for 9 iterations in a similar timeframe (around 3466.36 ms), though its correctness was noted as a concern.

These results demonstrate the significant speedup achieved by GPU acceleration and the efficiency of cuSPARSE for the graph coloring preprocessing step compared to the custom CPU coloring for the tested matrices.

## Notes & Future Work

As mentioned in the notebook:
* Under-relaxation was applied to select SuiteSparse matrices because the absence of an explicit $b$ vector prevented convergence; with extremely large coefficients in the matrix, iterative methods would otherwise diverge in the absence of a stabilizing factor.
* Contribute to an open source simulation repository that may use Preconditioned Conjugate Gradient or Jacobi or serial Gauss Seidel, and use a multicoloring approach. Note that grid based approaches do not benefit from multicoloring, as at most there would be 2 colors, known as red-black coloring. Thus, this is suited to some physics simulations. While there may not be an inner product at every update, there may be a constraint update such as in Position Based Dynamics. Thus, a constraint graph must be generated to parallelize it.
* Use METIS library to partition into super nodes, color the super nodes and keep the processing of the super nodes, or color those individually. Note that this is only suitable for extremely large graphs with many SCC's.
* Genetic algorithms on the GPU to create the coloring may also be worth exploring, especially if when initially testing it out, it outperforms CSRColor by producing less colors and a valid configuration.
* The SpMV-based Gauss-Seidel update requires further development and validation for correctness.

## Acknowledgements

* The CPU Gauss-Seidel implementation was based on a gist by Eric Arneback.
