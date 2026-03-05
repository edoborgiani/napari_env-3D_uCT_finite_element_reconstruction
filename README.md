# 3D µCT Finite Element Reconstruction

A Python pipeline for converting micro-computed tomography (µCT) medical imaging data into finite element analysis (FEA) models with material properties derived from bone density.

## Overview

This project provides a complete workflow to transform 3D medical imaging data (NRRD format) into Abaqus-compatible finite element meshes with spatially-varying material properties. The pipeline automatically:

- Aligns and preprocesses µCT volumes
- Segments bone/tissue from CT intensity values
- Generates high-quality tetrahedral meshes
- Computes element-wise material properties (density, Young's modulus) from CT data
- Groups materials efficiently for FEA simulation
- Exports models in Abaqus `.inp` format

## Features

- **Automatic Volume Alignment**: Aligns specimens by centroid analysis along any axis
- **Adaptive Downsampling**: Reduces computation time while preserving geometric features
- **Island Removal**: Cleans segmentation artifacts and isolated components
- **ROI Support**: Restricts analysis to user-defined regions of interest
- **Support Block Generation**: Adds load application regions for mechanical testing simulation
- **Density-Based Material Assignment**: Converts CT intensities to bone density and Young's modulus using empirical relationships
- **Material Binning**: Groups elements with similar properties to reduce model complexity
- **Multiple Versions**: Includes v1.0, v1.1, and v1.2 with incremental improvements

## Requirements

### Python Packages

```bash
pip install pyvista
pip install SimpleITK
pip install vtk
pip install numpy
pip install scipy
pip install meshpy
pip install meshlib
```

### Dependencies

- **PyVista**: 3D visualization and mesh manipulation
- **SimpleITK**: Medical image processing
- **VTK**: Visualization toolkit
- **NumPy**: Numerical computing
- **SciPy**: Scientific computing (image processing)
- **MeshPy**: Tetrahedral mesh generation
- **MeshLib**: Advanced meshing utilities

## Installation

1. Clone the repository:
```bash
git clone https://github.com/edoborgiani/napari_env-3D_uCT_finite_element_reconstruction.git
cd napari_env-3D_uCT_finite_element_reconstruction
```

2. Install dependencies:
```bash
pip install -r requirements.txt
```
   *(Note: Create a requirements.txt with the packages listed above if needed)*

## Usage

### Basic Workflow

1. **Prepare your data**: Place your µCT volume in NRRD format in the working directory

2. **Configure parameters**: Edit the main processing cell with your settings:
```python
input_nrrd = "microCT_volume_preview.nrrd"  # Input CT file
iso_value = 23103                           # Intensity threshold for segmentation
downsample_factor = 4                       # Downsampling factor for speed
roi = [0,0,0,0,0,0]                        # ROI [x0,x1,y0,y1,z0,z1], [0,0,0,0,0,0] = full volume
```

3. **Run the notebook**: Execute cells sequentially in Jupyter:
   - Function definitions
   - Data import and alignment
   - Mesh generation
   - Material binning
   - Export to Abaqus format

4. **Output files**:
   - `*_TETmesh.stl`: Surface mesh
   - `*_TETmesh_rot.vtk`: Tetrahedral mesh with material properties
   - `*_TETmesh_binned_E.inp`: Abaqus input file

### Key Parameters

- **`iso_value`**: CT intensity threshold for bone segmentation (tune based on your scanner/protocol)
- **`downsample_factor`**: Integer factor to reduce mesh resolution (higher = faster but coarser)
- **`roi`**: Region of interest as `[x0,x1,y0,y1,z0,z1]` in voxel indices
- **`fraction`**: Fraction of volume ends used for centroid alignment (default: 0.1)
- **`axis`**: Alignment axis - "X", "Y", or "Z"
- **`max_volume`**: Controls tetrahedral element size (smaller = finer mesh)
- **`tolerance`**: Material grouping tolerance (1% = group materials within 1% Young's modulus)

## Workflow Details

### 1. Volume Preprocessing
- **Alignment**: Computes centroids at specimen ends and rotates to align with chosen axis
- **Downsampling**: Reduces voxel count by integer factor to speed up meshing
- **Segmentation**: Binary thresholding at user-defined CT intensity

### 2. Mask Processing
- **Island Removal**: Eliminates small disconnected components (artifacts)
- **ROI Masking**: Restricts processing to specified region
- **Block Creation**: Adds support regions (value 2) at specimen ends for boundary conditions

### 3. Mesh Generation
- **Surface Extraction**: Converts binary mask to triangulated surface mesh (STL)
- **Decimation**: Reduces triangle count for efficiency
- **Tetrahedralization**: Generates volume mesh using constrained Delaunay triangulation

### 4. Material Assignment
- **HU Sampling**: Interpolates CT intensity at each element centroid
- **Density Conversion**: `ρ = 2.2×10⁻⁷ × HU² - 6.4×10⁻⁶ × HU`
- **Young's Modulus**: `E = 1.4×10⁴ × ρ²` (MPa)
- **Material Binning**: Groups similar moduli to reduce material count

### 5. Export
- **Abaqus Format**: Writes `.inp` file with:
  - Node coordinates
  - C3D4 tetrahedral elements
  - Material definitions per binned group
  - Element sets and section assignments

## Input Files

### Required
- **`microCT_volume_preview.nrrd`**: µCT volume (or your specified filename)

### Template Files (for advanced Abaqus setup)
- **`inp_header.txt`**: Abaqus header template
- **`inp_footer.txt`**: Abaqus footer template (assembly, boundary conditions)

## Output Files

- **`*_TETmesh.stl`**: Triangulated surface mesh
- **`*_TETmesh_rot.vtk`**: Tetrahedral mesh with HU, density, and Young's modulus per element
- **`*_TETmesh_binned_E.inp`**: Complete Abaqus input deck with materials

## Versions

- **v1.0**: Initial implementation
- **v1.1**: Enhanced ROI handling and alignment
- **v1.2**: Improved material binning and export options (recommended)

## Example Material Properties

The pipeline uses bone-specific empirical relationships:

| HU Value | Density (g/cm³) | Young's Modulus (MPa) |
|----------|-----------------|------------------------|
| 10,000   | ~1.6            | ~35,000                |
| 20,000   | ~8.2            | ~940,000               |
| Support  | 5.0             | 1,000,000              |

*Note: Support blocks receive fixed properties to represent rigid test fixtures.*

## Troubleshooting

### Common Issues

1. **No voxels above threshold**
   - Reduce `iso_value` until segmentation captures bone

2. **Memory errors**
   - Increase `downsample_factor` (try 6 or 8)
   - Reduce ROI size

3. **Mesh quality issues**
   - Adjust `max_volume` parameter in `build()` function
   - Check surface decimation settings

4. **Alignment failures**
   - Verify ROI contains sufficient bone at specimen ends
   - Adjust `fraction` parameter (try 0.05 or 0.15)

## Citation

If you use this pipeline in your research, please cite:

```
Borgiani, E. (2026). 3D µCT Finite Element Reconstruction Pipeline.
GitHub repository: https://github.com/edoborgiani/napari_env-3D_uCT_finite_element_reconstruction
```

## License

*[Add your license information here]*

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## Contact

For questions or issues, please open an issue on GitHub or contact the repository owner.

## Acknowledgments

This pipeline was developed for biomechanical analysis of bone specimens using clinical/preclinical µCT data and finite element simulation.
