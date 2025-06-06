**core\_toolbox\_python**
A lightweight Python toolbox providing utilities for camera intrinsics, Plücker‐line representations, and 3D transformation matrices. This package is organized into three submodules:

* **Camera.Intrinsics**: Classes for intrinsic camera matrices (Matlab/OpenCV conventions), radial distortion, ray generation, and JSON serialization.
* **Plucker.Line**: A `Line` class to represent 3D lines (start/end points or Plücker coordinates), intersection computations, line fitting, and basic plotting utilities.
* **Transformation.TransformationMatrix**: A 4×4 rigid‐body transformation class with support for Euler angles (radians/degrees), quaternions, Bundler‐format I/O, JSON serialization, inversion, chaining, and plotting (matplotlib/Open3D).

---

## Table of Contents

1. [Features](#features)
2. [Requirements](#requirements)
3. [Installation](#installation)
4. [Module Overview](#module-overview)

   * [Camera.Intrinsics](#cameraintrinsics)
   * [Plucker.Line](#pluckerline)
   * [Transformation.TransformationMatrix](#transformationtransformationmatrix)
5. [Usage Examples](#usage-examples)
6. [Development & Contributing](#development--contributing)
7. [License](#license)

---

## Features

* **Intrinsics & Distortion**

  * Create and manipulate camera intrinsic matrices in both Matlab and OpenCV formats.
  * Store and serialize radial‐distortion coefficients.
  * Compute focal length in millimeters (if pixel size is known).
  * Compute perspective (field‐of‐view) angles.
  * Generate per‐pixel rays as Plücker‐line objects.
  * Save/load intrinsic parameters to/from JSON.

* **Plücker‐Line Representation**

  * Represent a set of 3D rays or line segments via Plücker coordinates.
  * Compute shortest‐distance intersections between two sets of lines.
  * Fit a line to a cloud of 3D points (including placeholder methods for RANSAC, to be implemented).
  * Compute angles between two lines.
  * Basic 3D plotting of lines (matplotlib).

* **Transformation Matrices**

  * Encapsulate a 4×4 rigid transformation (rotation + translation).
  * Get/set translation (`.T`) and rotation (`.R`) as 3×3 matrix.
  * Get/set Euler angles in radians (`.angles`) or degrees (`.angles_degree`) via SciPy.
  * Get/set quaternion (`.quaternion`) for the rotation.
  * Apply transformation to point clouds.
  * Invert transformations, chain multiple transformations with `@`.
  * Save/load transformations in JSON.
  * Save/load Bundler v0.3 camera entries for MeshLab (single‐camera mode).
  * Plot coordinate frames in 3D (matplotlib, or Open3D if available).

---

## Requirements

* Python ≥ 3.7
* NumPy
* Matplotlib
* SciPy (especially `scipy.spatial.transform.Rotation`)
* scikit‐learn (for any future line‐fitting routines)

*(All dependencies are declared in `pyproject.toml` or `setup.py` under `dependencies`.)*

---

## Installation

1. **Clone the repository**

   ```bash
   git clone https://github.com/yourusername/core_toolbox_python.git
   cd core_toolbox_python
   ```

2. **Build a wheel (PEP 517)**

   ```bash
   python -m pip install --upgrade pip
   pip install build
   python -m build --wheel
   ```

   A `.whl` file will appear under `dist/`.

3. **Install from the local wheel**

   ```bash
   pip install dist/core_toolbox_python-0.1.0-py3-none-any.whl
   ```

4. **Or install in editable/development mode**

   ```bash
   pip install -e .
   ```

   This lets you modify source code and have changes reflected immediately.

---

## Module Overview

### Camera.Intrinsics

**File**: `core_toolbox_python/Camera/Intrinsics.py`

* **Class `RadialDistortion`**

  * Holds distortion coefficients `k1, k2, k3`.
  * `set_from_list([k1, k2, k3])`: assign three‐element coefficient list.

* **Class `IntrinsicMatrix`**

  * Attributes:

    * `fx, fy, cx, cy, s` (standard pinhole‐camera parameters).
    * `width, height` (image resolution).
    * `pixel_size` (in millimeters, e.g. sensor pixel pitch).
    * `RadialDistortion`: an instance of `RadialDistortion`.
    * `.info`: optional metadata (e.g. camera/lens ID).

  * **Properties**:

    * `.MatlabIntrinsics` (getter/setter): 3×3 matrix in Matlab convention (⎡fx s 0; 0 fy 0; cx cy 1⎤).
    * `.OpenCVIntrinsics` (getter/setter): 3×3 matrix in OpenCV convention (⎡fx 0 cx; 0 fy cy; 0 0 1⎤).
    * `.focal_length_mm`: returns `(fx ⋅ pixel_size, fy ⋅ pixel_size)`.
    * `.PerspectiveAngle` (getter/setter): horizontal or vertical field‐of‐view (degrees) based on `width/height` vs `fx,fy`.

  * **Methods**:

    * `.CameraParams2Intrinsics(CameraParams)`: load intrinsics from an external camera‐parameters object (e.g. if you have a `CameraParams.IntrinsicMatrix` & `CameraParams.ImageSize`).
    * `.Intrinsics2CameraParams()`: return a dictionary `{IntrinsicMatrix: […], ImageSize: […], RadialDistortion: …}`.
    * `.ScaleIntrinsics(s)`: multiply `fx, fy, cx, cy, width, height` by scale ` s`.
    * `.generate_rays() → Line`: produce a `Line` object where each row corresponds to a 3D ray originating from pixel centers; uses radial‐undistortion (if defined).
    * `.save_intrinsics_to_json(filename)`: write a JSON file containing OpenCV intrinsics, distortion, resolution, pixel size, and `info`.
    * `.load_intrinsics_from_json(filename)`: read JSON file and populate intrinsics, distortion, `width, height, pixel_size, info`.

  * **Example** (at bottom of file):

    ```python
    if __name__ == "__main__":
        I = IntrinsicMatrix()
        I.info = "testCamera"
        I.fx = I.fy = 1770
        I.width, I.height = 1440, 1080
        I.cx, I.cy = 685, 492
        I.RadialDistortion.set_from_list([-0.5, 0.18, 0])
        I.save_intrinsics_to_json("test.json")
        rays = I.generate_rays()  # Plücker‐line set
        I2 = IntrinsicMatrix().load_intrinsics_from_json("test.json")
        # … compute intersections, etc.
    ```

---

### Plucker.Line

**File**: `core_toolbox_python/Plucker/Line.py`

* **Function `intersection_between_2_lines(L1, L2)`**

  * Computes closest‐point midpoints and shortest distances between each corresponding pair of rays in two `Line` objects.
  * Inputs:

    * `L1`, `L2`: each a `Line` instance with `Ps` (start points) and `V` (direction vectors).
  * Returns:

    * `Points`: an `(N, 3)` array of midpoints between ray *i* from `L1` and ray *i* from `L2`.
    * `distances`: an `(N,)` array of shortest distances.

* **Class `Line`**

  * **Attributes**:

    * `Ps`: `(N, 3)` array of start (origin) points of each line/ray.
    * `Pe`: `(N, 3)` array of end points (so direction = `Pe − Ps`).

  * **Properties**:

    * `.V` (getter): normalized direction vectors for each ray (`(Pe − Ps)` normalized row‐wise).
    * `.V` (setter): sets `Pe = Ps + new_direction`.
    * `.Plucker` (getter): concatenates direction `V` and moment `U=Ps×(Ps+V)` into a `(N,6)` array.
    * `.Plucker` (setter): given a `(N,6)` array, recovers `Ps` and `V` via cross‐product inversion.
    * `.Plucker2` (alternative Plücker ordering): stores `(moment = Ps×Pe ∥ direction=Pe−Ps)`.

  * **Methods**:

    * `.GetAngle()`: returns the angle (in degrees) between each ray and the world‐Z unit vector.
    * `.TransformLines(H)`: applies a `TransformationMatrix` `H` to both `Ps` and `Pe`.
    * `.plot(limits=None, colors=None, …)`: wide‐ranging helper that draws as many lines as you like in 3D (within bounds).
    * `.PlotLine(colori='g', linewidth=2)`: simpler per‐line plotting (downsamples if >500 rays).
    * `.FindXYZNearestLine(XYZ)`: given a single 3D point cloud `XYZ`, returns the index of the ray that is closest.
    * `.FitLine(XYZ)`: placeholder for least‐squares fit to 3D points (calls `_fitline3d`).
    * `.FitLineRansac(XYZ, t=10)`: placeholder for RANSAC line fit (calls `_ransac_fit_line`).
    * `.NormaliseLine()`: project all line origins so that `z=0`.
    * `.DistanceLinePoint(XYZ)`: shortest distance from each line to each query point in `XYZ`.
    * `.Lenght()`: length of each line segment (`‖Pe−Ps‖`).
    * `@staticmethod FromStartEnd(start, end)`: build a `Line` from start/end points.
    * `@staticmethod FromPlucker(VU)`: build a `Line` given a `(N,6)` Plücker array.
    * **Internal helpers**: `_normalize_vectors`, `_is_within_bounds`, `_downsample`, `_fitline3d`, `_ransac_fit_line`, `_homogeneous_transform`, etc. (some are stubs for future extension).
    * `.AngleBetweenLines(L1, L2)`: returns angle (radians, degrees) between two `Line` objects (single‐ray version).
    * `.GenerateRay(I, uv)`: generate rays passing through pixel coordinates `uv` using intrinsics `I`.

  * **Example** (at bottom of file):

    ```python
    if __name__ == "__main__":
        L = Line()
        L.Ps = np.array([[1,1,0]])
        L.Pe = np.array([[2,1,0]])
        print(L.V)  # direction vector
        L.PlotLine()
        L2 = Line()
        L2.Ps, L2.Pe = np.array([[0,0,0]]), np.array([[20,20,0]])
        _, hoek = L.AngleBetweenLines(L, L2)
        print("Angle between lines:", hoek)
    ```

---

### Transformation.TransformationMatrix

**File**: `core_toolbox_python/Transformation/TransformationMatrix.py`

* **Class `TransformationMatrix`**

  * Internally stores a 4×4 homogeneous transform `self.H` (initialized to identity).

  * **Attributes**:

    * `.H`: 4×4 NumPy array.
    * `.info`: a two‐element list of arbitrary metadata (e.g. camera ID, timestamp).
    * `.units`: string indicating units (default `"mm"`).

  * **Properties**:

    * `.T` (getter/setter): get/set the translation vector (3×1).
    * `.R` (getter/setter): get/set the 3×3 rotation submatrix.
    * `.angles` (getter/setter): Euler angles in radians (XYZ convention) via `scipy.spatial.transform.Rotation`.
    * `.angles_degree` (getter/setter): Euler angles in degrees.
    * `.quaternion` (getter/setter): quaternion `[x, y, z, w]` representation of the rotation.

  * **Methods**:

    * `.transform(points)`: apply the 4×4 transform to an `(N,3)` or `(3,)` array of 3D points, returning transformed `(N,3)`.
    * `.invert()`: invert the transformation in‐place, swap and invert `H`, and reverse the `info` list.
    * `.save_bundler_file(output_file, intrinsics=None)`: write a Bundler v0.3‐style camera entry (single camera, zero points) to a text file—storing focal length, distortion (set to zero), rotation rows, and translation vector. If `intrinsics` is `None`, a default intrinsic matrix is used (example values).
    * `.load_bundler_file(filename)`: read a Bundler file (ignore first three lines), load rotation (3×3) and translation (3×1) back into `H`.
    * `.plot(scale=1.0)`: visualize this transformation as a 3D coordinate frame (matplotlib).
    * `.plot_open3d(scale=1.0)`: visualize using Open3D’s `TriangleMesh.create_coordinate_frame`; requires `open3d` installed.
    * `.copy()`: return a deep copy of this `TransformationMatrix`.
    * `.load_from_json(filename)`: read `H`, `info`, `units` from a JSON file.
    * `.save_to_json(filename)`: write `H`, `info`, `units` to JSON.
    * `__matmul__(self, other)`: allow chaining two transformations `T_combined = T1 @ T2` (i.e. matrix multiply). The combined `info` is taken as `[self.info[0], other.info[-1]]` by default.
    * `__repr__`: printable representation of the 4×4 matrix.

  * **Example** (at bottom of file):

    ```python
    if __name__ == "__main__":
        T1 = TransformationMatrix()
        T1.T = [0, 10, 0]
        T1.angles_degree = [0, 30, 0]
        T1.save_bundler_file("test.out")
        print("T1:\n", T1)
        print("Quaternion:", T1.quaternion)
        T1.plot()

        T2 = T1.copy()
        T2.invert()
        T2.plot()
        T_combined = T1 @ T2
        print("Combined:\n", T_combined)
    ```

---

## Usage Examples

Below are some minimal snippets illustrating how to import and use the package once installed.

### 1. Reading/Writing Intrinsics

```python
from core_toolbox_python.Camera.Intrinsics import IntrinsicMatrix, RadialDistortion

# Create an intrinsic matrix
I = IntrinsicMatrix()
I.fx = 1200
I.fy = 1200
I.cx = 640
I.cy = 360
I.width = 1280
I.height = 720
I.pixel_size = 0.0034  # e.g. 3.4 µm
I.RadialDistortion.set_from_list([0.01, -0.001, 0.0])

# Compute OpenCV format
K_opencv = I.OpenCVIntrinsics
print("OpenCV Intrinsics:\n", K_opencv)

# Save to JSON
I.save_intrinsics_to_json("camera_intrinsics.json")

# Load back
I2 = IntrinsicMatrix().load_intrinsics_from_json("camera_intrinsics.json")
print("Loaded fx, fy:", I2.fx, I2.fy)
```

### 2. Generating Rays & Line Intersections

```python
from core_toolbox_python.Camera.Intrinsics import IntrinsicMatrix
from core_toolbox_python.Plucker.Line import intersection_between_2_lines

# Suppose we have two camera poses, project rays, and compute their closest‐point intersections

# Camera 1 intrinsics
I1 = IntrinsicMatrix()
I1.fx = I1.fy = 1000
I1.cx, I1.cy = 320, 240
I1.width, I1.height = 640, 480
I1.pixel_size = 0.0025
# … set radial distortion if needed …

# Camera 2 intrinsics (shifted horizontally by 1 unit)
I2 = IntrinsicMatrix()
I2.fx = I2.fy = 1000
I2.cx, I2.cy = 320, 240
I2.width, I2.height = 640, 480
I2.pixel_size = 0.0025

# Generate full‐image rays from each camera (Plücker‐line sets)
rays1 = I1.generate_rays()
rays2 = I2.generate_rays()

# Compute midpoint & distances between corresponding rays
points_mid, distances = intersection_between_2_lines(rays1, rays2)
print("Mean distance between ray pairs:", distances.mean())
```

### 3. Creating & Transforming 3D Geometry

```python
from core_toolbox_python.Transformation.TransformationMatrix import TransformationMatrix
import numpy as np

# Define a transformation: translate by [1,2,3], rotate 45° about Z
T = TransformationMatrix()
T.T = [1, 2, 3]
T.angles_degree = [0, 0, 45]

# Transform a set of points
points = np.array([[0,0,0], [1,0,0], [0,1,0]])
points_transformed = T.transform(points)
print("Transformed points:\n", points_transformed)

# Inverse transform
T_inv = T.copy()
T_inv.invert()
restored = T_inv.transform(points_transformed)
print("Restored (should match original):\n", restored)

# Save to JSON
T.save_to_json("transform.json")
T2 = TransformationMatrix().load_from_json("transform.json")

# Chain transformations
T_comb = T @ T2  # (applies T first, then T2)
```

### 4. Visualization

```python
import matplotlib.pyplot as plt
from core_toolbox_python.Plucker.Line import Line

# Plotting a single ray
L = Line()
L.Ps = np.array([[0, 0, 0]])
L.Pe = np.array([[1, 1, 1]])
L.PlotLine(colori='r')

# Plot a coordinate frame
from core_toolbox_python.Transformation.TransformationMatrix import TransformationMatrix
T = TransformationMatrix()
T.T = [0, 0, 0]
T.angles_degree = [30, 45, 60]
T.plot(scale=1.0)
plt.show()
```

---

### Development & Contributing

1. **Clone & install in “editable” mode**  
   ```bash
   git clone https://github.com/yourusername/core_toolbox_python.git
   cd core_toolbox_python
   pip install -e .
    ```

2. **Make changes on a feature branch**

   * Create a new branch off `main` (or `develop`).

     ```bash
     git checkout -b feature/my_update
     ```
   * Implement or update functionality as needed (e.g., fill in placeholder methods, add examples, fix bugs).

3. **Run tests & verify locally**

   * If you add new functionality, include or update any unit tests.
   * Make sure existing examples and import statements continue to work.

4. **Tag-based release workflow**

   * CI is configured to build wheels **only when a Git tag is pushed**.
   * Once your branch is reviewed and merged into `main`, create a new lightweight or annotated tag following semantic versioning:

     ```bash
     git checkout main
     git pull origin main
     git tag -a vX.Y.Z -m "Release vX.Y.Z"
     git push origin vX.Y.Z
     ```
   * Pushing that tag will trigger the GitHub Actions workflow to build wheels for all platforms and upload them as artifacts.

5. **Submit a Pull Request**

   * Push your feature branch to the remote repository.

     ```bash
     git push origin feature/my_update
     ```
   * Open a Pull Request against `main`, describing your changes. Once approved and merged, follow the tag‐based release step above.

6. **After a successful tag build**

   * Download platform‐specific wheel artifacts from the “Artifacts” section in the GitHub Actions run.
   * Optionally, publish wheels to PyPI (you can use `twine upload dist/*` after downloading and verifying).

Thank you for contributing! If you have questions or need assistance, please open an issue or reach out directly.\`\`\`
---

## License

This project is distributed under the MIT License. See [LICENSE](LICENSE) for details.
