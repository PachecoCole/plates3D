# 3D Gaussian splatting for Three.js

This repository contains a Three.js-based implemetation of a renderer for [3D Gaussian Splatting for Real-Time Radiance Field Rendering](https://repo-sam.inria.fr/fungraph/3d-gaussian-splatting/), a technique for generating 3D scenes from 2D images. Their project is CUDA-based and needs to run natively on your machine, but I wanted to build a viewer that was accessible via the web.

The 3D scenes are stored in a format similar to point clouds and can be viewed, navigated, and interacted with in real-time. This renderer will work with both the `.ply` files generated by the INRIA project, or my own custom `.splat` files, which are a trimmed-down and compressed version of those files.

When I started, web-based viewers were already available -- A WebGL-based viewer from [antimatter15](https://github.com/antimatter15/splat) and a WebGPU viewer from [cvlab-epfl](https://github.com/cvlab-epfl/gaussian-splatting-web) -- However no Three.js version existed. I used those versions as a starting point for my initial implementation, but as of now this project contains all my own code.
<br>
<br>
## Highlights

- Rendering is done entirely through Three.js
- Code is organized into modern ES modules
- Built-in viewer is self-contained so very little code is necessary to load and view a scene
- Allows user to import `.ply` files for conversion to custom compressed `.splat` file format
- Allows a Three.js scene or object group to be rendered along with the splats

## Known issues

- Splat sort runs on the CPU – would be great to figure out a GPU-based approach
- Artifacts are visible when you move or rotate too fast (due to CPU-based splat sort)
- Sub-optimal performance on mobile devices
- Custom `.splat` file format still needs work, especially around compression

## Future work
This is still very much a work in progress! There are several things that still need to be done:
  - Improve the method by which splat data is stored in textures
  - Properly incorporate spherical harmonics data to achieve view dependent lighting effects
  - Continue optimizing CPU-based splat sort - maybe try an incremental sort of some kind?
  - Add editing mode, allowing users to modify scene and export changes
  - Add WebXR compatibility
  - Support very large scenes and/or multiple splat files

## Online demo
[https://projects.markkellogg.org/threejs/demo_gaussian_splats_3d.php](https://projects.markkellogg.org/threejs/demo_gaussian_splats_3d.php)

## Controls
Mouse
- Left click to set the focal point
- Left click and drag to orbit around the focal point
- Right click and drag to pan the camera and focal point
  
Keyboard
- `C` Toggles the mesh cursor, showing the intersection point of a mouse-projected ray and the splat mesh

- `I` Toggles an info panel that displays debugging info:
  - Camera position
  - Camera focal point/look-at point
  - Camera up vector
  - Mesh cursor position
  - Current FPS
  - Renderer window size
  - Ratio of rendered splats to total splats
  - Last splat sort duration

- `P` Toggles a debug object that shows the orientation of the camera controls. It includes a green arrow representing the camera's orbital axis and a white square representing the plane at which the camera's elevation angle is 0.

- `Left arrow` Rotate the camera's up vector counter-clockwise

- `Right arrow` Rotate the camera's up vector clockwise

<br>

## Building and running locally
Navigate to the code directory and run
```
npm install
```
Next run the build. For Linux & Mac OS systems run:
```
npm run build
```
For Windows I have added a Windows-compatible version of the build command:
```
npm run build-windows
```
To view the demo scenes locally run
```
npm run demo
```
The demo will be accessible locally at [http://127.0.0.1:8080/index.html](http://127.0.0.1:8080/index.html). You will need to download the data for the demo scenes and extract them into 
```
<code directory>/build/demo/assets/data
```
The demo scene data is available here: [https://projects.markkellogg.org/downloads/gaussian_splat_data.zip](https://projects.markkellogg.org/downloads/gaussian_splat_data.zip)
<br>
<br>

## Basic Usage

To run the built-in viewer:

```javascript
const viewer = new GaussianSplats3D.Viewer({
    'cameraUp': [0, -1, -0.6],
    'initialCameraPosition': [-1, -4, 6],
    'initialCameraLookAt': [0, 4, 0],
    'ignoreDevicePixelRatio': false,
    'gpuAcceleratedSort': true
});
viewer.loadFile('<path to .ply or .splat file>', {
    'splatAlphaRemovalThreshold': 5, // out of 255
    'halfPrecisionCovariancesOnGPU': true
})
.then(() => {
    viewer.start();
});
```
| Parameter | Purpose
| --- | ---
| `cameraUp` | The natural 'up' vector for viewing the scene. Determines the scene's orientation relative to the camera and serves as the axis around which the camera will orbit.
| `initialCameraPosition` | The camera's initial position.
| `initialCameraLookAt` | The initial focal point of the camera and center of the camera's orbit.
| `ignoreDevicePixelRatio` | Tells the viewer to pretend the device pixel ratio is 1, which can boost performance on devices where it is larger, at a small cost to visual quality. Defaults to `false`.
| `gpuAcceleratedSort` | Tells the viewer to use a partially GPU-accelerated approach to sorting splats. Currently this means pre-computing splat distances is done on the GPU. Defaults to `true`.
| `splatAlphaRemovalThreshold` | Tells `loadFile()` to ignore any splats with an alpha less than the specified value. Defaults to `1`.
| `halfPrecisionCovariancesOnGPU` |  Tells the viewer to use 16-bit floating point values for each element of a splat's 3D covariance matrix, instead of 32-bit. Defaults to `true`.

<br>


As an alternative to using `cameraUp` to adjust to the scene's natural orientation, you can pass an orientation (and/or position) to the `loadFile()` method to transform the entire scene:
```javascript
const viewer = new GaussianSplats3D.Viewer({
    'initialCameraPosition': [-1, -4, 6],
    'initialCameraLookAt': [0, 4, 0]
});
const orientation = new THREE.Quaternion();
orientation.setFromUnitVectors(new THREE.Vector3(0, -1, -0.6).normalize(), new THREE.Vector3(0, 1, 0));
viewer.loadFile('<path to .ply or .splat file>', {
    'splatAlphaRemovalThreshold': 5, // out of 255
    'halfPrecisionCovariancesOnGPU': true,
    'position': [0, 0, 0],
    'orientation': orientation.toArray(),
})
.then(() => {
    viewer.start();
});
```

The `loadFile()` method will accept the original `.ply` files as well as my custom `.splat` files.
<br>
<br>
### Creating SPLAT files
To convert a `.ply` file into the stripped-down `.splat` format (currently only compatible with this viewer), there are several options. The easiest method is to use the UI in the main demo page at [http://127.0.0.1:8080/index.html](http://127.0.0.1:8080/index.html). If you want to run the conversion programatically, run the following in a browser:

```javascript
const compressionLevel = 1;
const splatAlphaRemovalThreshold = 5; // out of 255
const plyLoader = new GaussianSplats3D.PlyLoader();
plyLoader.loadFromURL('<URL for .ply file>', compressionLevel, splatAlphaRemovalThreshold)
.then((splatBuffer) => {
    new GaussianSplats3D.SplatLoader(splatBuffer).downloadFile('converted_file.splat');
});
```
Both of the above methods will prompt your browser to automatically start downloading the converted `.splat` file.

The third option is to use the included nodejs script:

```
node util/create-splat.js [path to .PLY] [output file] [compression level = 0] [alpha removal threshold = 1]
```

Currently supported values for `compressionLevel` are `0` or `1`. `0` means no compression, `1` means compression of scale, rotation, and position values from 32-bit to 16-bit.
<br>
<br>
### Integrating THREE.js scenes
You can integrate your own Three.js scene into the viewer if you want rendering to be handled for you. Just pass a Three.js scene object as the 'scene' parameter to the constructor:
```javascript
const scene = new THREE.Scene();

const boxColor = 0xBBBBBB;
const boxGeometry = new THREE.BoxGeometry(2, 2, 2);
const boxMesh = new THREE.Mesh(boxGeometry, new THREE.MeshBasicMaterial({'color': boxColor}));
scene.add(boxMesh);
boxMesh.position.set(3, 2, 2);

const viewer = new GaussianSplats3D.Viewer({
    'scene': scene,
    'cameraUp': [0, -1, -0.6],
    'initialCameraPosition': [-1, -4, 6],
    'initialCameraLookAt': [0, 4, -0]
});
viewer.loadFile('<path to .ply or .splat file>')
.then(() => {
    viewer.start();
});
```
Currently this will only work for objects that write to the depth buffer (e.g. standard opaque objects). Supporting transparent objects will be more challenging :)
<br>
<br>
### Custom options
The viewer allows for various levels of customization via constructor parameters. You can control when its `update()` and `render()` methods are called by passing `false` for the `selfDrivenMode` parameter and then calling those methods whenever/wherever you decide is appropriate. You can tell the viewer to not use its built-in camera controls by passing `false` for the `useBuiltInControls` parameter. You can also use your own Three.js renderer and camera by passing those values to the viewer's constructor. The sample below shows all of these options:

```javascript
const renderWidth = 800;
const renderHeight = 600;

const rootElement = document.createElement('div');
rootElement.style.width = renderWidth + 'px';
rootElement.style.height = renderHeight + 'px';
document.body.appendChild(rootElement);

const renderer = new THREE.WebGLRenderer({
    antialias: false
});
renderer.setSize(renderWidth, renderHeight);
rootElement.appendChild(renderer.domElement);

const camera = new THREE.PerspectiveCamera(65, renderWidth / renderHeight, 0.1, 500);
camera.position.copy(new THREE.Vector3().fromArray([-1, -4, 6]));
camera.lookAt(new THREE.Vector3().fromArray([0, 4, -0]));
camera.up = new THREE.Vector3().fromArray([0, -1, -0.6]).normalize();

const viewer = new GaussianSplats3D.Viewer({
    'selfDrivenMode': false,
    'renderer': renderer,
    'camera': camera,
    'useBuiltInControls': false
});
viewer.loadFile('<path to .ply or .splat file>')
.then(() => {
    requestAnimationFrame(update);
});
```
Since `selfDrivenMode` is false, it is up to the developer to call the `update()` and `render()` methods on the `Viewer` class:
```javascript
function update() {
    requestAnimationFrame(update);
    viewer.update();
    viewer.render();
}
```
<br>

### CORS issues and SharedArrayBuffer
The `Viewer` class uses shared memory (via a typed array backed by a SharedArrayBufffer) to communicate with the web worker that sorts the splats. This mechanism presents a potential security issue that is outlined here: https://web.dev/articles/cross-origin-isolation-guide.

To resolve this issue, a couple of extra CORS HTTP headers need to be present in the response from the server that is sent when loading the application. Without those headers set, you might see an error like the following in the debug console:

```
"DOMException: Failed to execute 'postMessage' on 'DedicatedWorkerGlobalScope': SharedArrayBuffer transfer requires self.crossOriginIsolated."
```

For the local demo I created a simple HTTP server (util/server.js) that sets those headers:

```
response.setHeader("Cross-Origin-Opener-Policy", "same-origin");
response.setHeader("Cross-Origin-Embedder-Policy", "require-corp");
```

If you're using Apache, you can edit the `.htaccess` file to do that by adding the lines:

```
Header add Cross-Origin-Opener-Policy "same-origin"
Header add Cross-Origin-Embedder-Policy "require-corp"
```

For other web servers, these headers most likely can be set in a similar fashion.

Additionally you may need to require a secure connection to your server by redirecting all access via `http://` to `https://`. In Apache this can be done by updating the `.htaccess` file with the following lines:

```
RewriteEngine On
RewriteCond %{HTTPS} off
RewriteRule (.*) https://%{HTTP_HOST}%{REQUEST_URI} [R,L]
```
