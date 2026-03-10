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
pip install pyvista>=0.40.0
pip install SimpleITK>=2.2.0
pip install vtk>=9.1.0
pip install numpy>=1.24.0
pip install scipy>=1.8.0
pip install meshpy>=2025.1.0
pip install meshlib>=2025.1.0
```

### Dependencies

- **PyVista** (≥0.40.0): 3D visualization, mesh manipulation, and UnstructuredGrid operations
- **SimpleITK** (≥2.2.0): Medical image processing and image resampling
- **VTK** (≥9.1.0): Visualization toolkit with tetrahedral element support
- **NumPy** (≥1.24.0): Numerical computing and array operations
- **SciPy** (≥1.8.0): Scientific computing, image processing, and coordinate interpolation
- **MeshPy** (≥2025.1.0): Constrained Delaunay triangulation and tetrahedral meshing
- **MeshLib** (≥2025.1.0): Advanced 3D mesh processing and surface extraction

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
   - (Optional) Provide a segmentation mask file (e.g., `Bone.seg.nrrd`) for pre-segmented data

2. **Configure parameters** (v1.2): Edit the main processing cell with your settings:
```python
input_nrrd = "microCT_volume_preview.nrrd"  # Input CT file
mask_nrrd = "Bone.seg.nrrd"                 # Pre-segmented mask ([] for intensity-based segmentation)
iso_value = 23103                           # Intensity threshold for segmentation (if mask_nrrd = [])
downsample_factor = 2                       # Downsampling factor for speed
roi = [0,0,0,0,0,0]                         # ROI [x0,x1,y0,y1,z0,z1], [0,0,0,0,0,0] = full volume
size_blocks = 0.0                           # Support block thickness in voxels (0.0 = no blocks)
rotation = True                             # Enable/disable volume alignment
```

3. **Run the notebook**: Execute cells sequentially in Jupyter:
   - Function definitions
   - Data import and optional alignment
   - Mesh generation with quality control
   - Material binning
   - Export to Abaqus format

4. **Output files**:
   - `*_TETmesh.stl`: Triangulated surface mesh
   - `*_TETmesh_rot.vtk`: Tetrahedral mesh with element-wise material properties (HU, density, Young's modulus)
   - `*_TETmesh_binned_E.inp`: Abaqus input file with binned material definitions

### Key Parameters

- **`input_nrrd`**: Path to input µCT volume file (NRRD format)
- **`mask_nrrd`**: Path to pre-computed segmentation mask (NRRD format), or `[]` to use iso_value-based segmentation
- **`iso_value`**: CT intensity threshold for bone segmentation (tune based on your scanner/protocol)
- **`downsample_factor`**: Integer factor to reduce mesh resolution (higher = faster but coarser; default: 2-4)
- **`roi`**: Region of interest as `[x0,x1,y0,y1,z0,z1]` in voxel indices; `[0,0,0,0,0,0]` uses full volume
- **`size_blocks`**: Thickness in voxels for support/load blocks at specimen ends (0.0 = no blocks, typically 5-10)
- **`rotation`**: Boolean flag to enable/disable automatic volume alignment (default: True)
- **`fraction`**: Fraction of volume ends used for centroid alignment (default: 0.1)
- **`axis`**: Alignment axis - "X", "Y", or "Z" (default: "Z")
- **`tolerance`**: Material grouping tolerance (decimal fraction, e.g., 0.01 = 1% Young's modulus variation)

## Workflow Details

### 1. Volume Preprocessing
- **Data Input**: Loads NRRD files (CT volumes or pre-computed segmentation masks)
- **Alignment** (optional): Computes centroids at specimen ends and rotates to align with chosen axis using Rodrigues' formula
- **Downsampling**: Reduces voxel count by integer factor to speed up meshing while preserving physical extent
- **Segmentation**: Binary thresholding at user-defined CT intensity (if not using a pre-made mask)

### 2. Mask Processing
- **Island Removal**: Eliminates small disconnected components (artifacts) using connected-component labeling
- **ROI Masking**: Restricts processing to specified region if provided
- **Support Block Generation**: Adds rigid support regions (value 2) at specimen ends for boundary conditions

### 3. Surface Extraction & Mesh Generation
- **Surface Extraction**: Converts binary mask to triangulated surface mesh (STL) using MeshLib dense grid
- **Surface Smoothing**: Applies Laplacian smoothing (100 iterations) for improved mesh quality
- **Tetrahedralization**: Generates volume mesh using MeshPy with quality controls
  - Quality metric: `pq1.2a1.0` (perturb/quality mixed optimization)
  - Produces C3D4 (linear tetrahedral) elements suitable for FEA

### 4. Material Assignment
- **Centroid Sampling**: Interpolates CT intensity (HU values) at each element centroid
- **Density Conversion**: Applies empirical formula: `ρ = 2.2×10⁻⁷ × HU² - 6.4×10⁻⁶ × HU` (g/cm³)
- **Young's Modulus**: Computes from density: `E = 1.4×10⁴ × ρ²` (MPa)
- **Material Binning**: Groups elements with similar moduli using greedy clustering
  - Reduces material count while preserving property variation
  - Improves FEA solver efficiency

### 5. Export to Abaqus
- **Node Coordinates**: Writes all mesh nodes with spatial coordinates
- **Element Definitions**: Exports C3D4 tetrahedral elements with 1-based node numbering
- **Material Definitions**: Groups elements into sets by binned material properties
- **Section Assignments**: Links element sets to corresponding material definitions

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

### v1.2 (Current, Recommended)
- **Optional Volume Alignment**: Toggle-able centroid-based alignment with Rodrigues rotation
- **Segmentation Mask Support**: Import pre-computed segmentation masks (e.g., from Slicer, ITK-SNAP)
- **Flexible Support Blocks**: Variable block thickness; skip blocks entirely if not needed
- **Improved Surface Mesh Quality**: Added Laplacian smoothing (100 iterations) for better element quality
- **Mesh Quality Control**: Integrated MeshPy Options for optimized tetrahedral generation
- **Enhanced Material Binning**: Greedy clustering algorithm groups similar E values efficiently
- **Better Parameter Control**: Separate control of masking method, rotation, and block generation

### v1.1
- Enhanced ROI handling and more flexible alignment parameters
- Support for island removal and mask processing
- Abaqus export with material grouping

### v1.0
- Initial implementation with basic workflow
- Volume preprocessing, meshing, and material assignment
- Abaqus-compatible output files

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
   - Or use `mask_nrrd` to provide pre-segmented data

2. **Memory errors**
   - Increase `downsample_factor` (try 6 or 8)
   - Reduce ROI size
   - Decrease `size_blocks` if using support regions

3. **Poor mesh quality or alignment failures**
   - Adjust `fraction` parameter (try 0.05 or 0.15) for centroid computation
   - Verify ROI contains sufficient bone at specimen ends
   - Set `rotation = False` if alignment causes issues
   - Increase smoothing iterations or adjust mesh quality parameters

4. **Mask file issues**
   - Ensure segmentation mask has same dimensions as input CT volume
   - Verify mask values: 0 = background, >0 = foreground
   - Check NRRD header for spacing/origin metadata

5. **Abaqus file import errors**
   - Verify node IDs are 1-indexed (they should be automatically)
   - Check that element node references are valid
   - Ensure material names and element sets are properly linked

## Citation

If you use this pipeline in your research, please cite:

```
Borgiani, E. (2026). 3D µCT Finite Element Reconstruction Pipeline.
GitHub repository: https://github.com/edoborgiani/napari_env-3D_uCT_finite_element_reconstruction
```

## Features by Version

| Feature | v1.0 | v1.1 | v1.2 |
|---------|------|------|------|
| Basic µCT → FEA workflow | ✓ | ✓ | ✓ |
| Volume alignment | ✓ | ✓ | ✓ (optional) |
| Island removal | ✗ | ✓ | ✓ |
| ROI masking | ✗ | ✓ | ✓ |
| Support blocks | ✗ | ✓ | ✓ (configurable) |
| Segmentation mask import | ✗ | ✗ | ✓ |
| Surface mesh smoothing | ✗ | ✗ | ✓ |
| Mesh quality control | ✗ | ✗ | ✓ |
| Material binning | ✗ | ✓ | ✓ (improved) |
| Abaqus `.inp` export | ✓ | ✓ | ✓ |

## Advanced Usage

### Using Pre-Segmented Data

If you have a segmentation mask from external software (e.g., Slicer, ITK-SNAP):

```python
mask_nrrd = "my_segmentation_mask.seg.nrrd"  # Provide filename
iso_value = 23103                             # Ignored if mask_nrrd is specified
```

The pipeline will use your mask directly, skipping intensity-based segmentation.

### Disabling Volume Alignment

For specimens that are already aligned or when ROI-based alignment fails:

```python
rotation = False  # Skip centroid-based alignment
```

### Customizing Material Groups

Adjust the tolerance parameter to control material binning granularity:

```python
tolerance = 0.01   # Group materials within 1% Young's modulus (default)
tolerance = 0.05   # Coarser binning (fewer materials)
tolerance = 0.001  # Finer binning (more materials, slower solver)
```

## License

*[Add your license information here]*

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## Contact

For questions or issues, please open an issue on GitHub or contact the repository owner.

## Acknowledgments

This pipeline was developed for biomechanical analysis of bone specimens using clinical/preclinical µCT data and finite element simulation.
