## ThreeJS

### Basic threejs scene
```
let container, renderer, scene, camera, controls;

function init() {
  container = document.getElementById("threeContainer");
  renderer = new THREE.WebGLRenderer();
  container.appendChild(renderer.domElement);
  renderer.setSize(container.clientWidth, container.clientHeight);
  scene = new THREE.Scene();
  camera = new THREE.PerspectiveCamera(
    40,
    container.clientWidth / container.clientHeight,
    1,
    10000
  );
  camera.position.set(2300, 2100, 1600);
  controls = new OrbitControls(camera, renderer.domElement);
  scene.add(new THREE.AmbientLight(0x222222));
  let light = new THREE.DirectionalLight(0xffffff, 2);
  light.position.set(3000, 2000, 1000);
  scene.add(light);
  scene.add(new THREE.AxesHelper(20));
  window.addEventListener("resize", onWindowResize, false);
}

function animate() {
  requestAnimationFrame(animate);
  renderer.render(scene, camera);
}

function onWindowResize() {
  camera.aspect = container.clientWidth / container.clientHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(container.clientWidth, container.clientHeight);
}

init();
animate();
```

### Add lights to threejs scene  
```
scene.add(new THREE.AmbientLight(0x222222));
let light = new THREE.DirectionalLight(0xffffff, 2);
light.position.set(3000, 2000, 1000);
scene.add(light);
```

### Add a red cube to the scene  
```
let cube = new THREE.Mesh(
  new THREE.BoxGeometry(1200, 1200, 1200, 1, 1, 1),
  new THREE.MeshStandardMaterial({ color: 0xff0000 })
);
scene.add(cube);
```

### Add particles to scene
```
let R = 0.89;
let G = 0.62;
let B = 0.17;
let size = 4000;
let particleCount = 80000;

let particles = new THREE.Geometry();
let pMaterial = new THREE.PointsMaterial({
  size: 2,
  vertexColors: THREE.VertexColors
});

for (let p = 0; p < particleCount; p++) {
  let pRat = p / particleCount;

  let r = pRat * size;

  let phi = Math.random() * Math.PI,
    theta = Math.random() * Math.PI;
  let pX = r * Math.sin(phi) * Math.cos(theta),
    pZ = r * Math.cos(phi),
    pY = r * Math.sin(phi) * Math.sin(theta),
    particle = new THREE.Vector3(pX, pY, pZ);

  particles.vertices.push(particle);
  let col = new THREE.Color(R, G, B);
  particles.colors.push(col);
}

let particleSystem = new THREE.Points(particles, pMaterial);

scene.add(particleSystem);

```

### Update particles
```


function average(arr) {
  let sum = 0;
  for (let i = 0; i < arr.length; i++) {
    sum += parseInt(arr[i], 10);
  }
  return sum / arr.length;
}

let prevLowsArr;
let prevMidsArr;
let prevHigsArr;

let avgPeriod = 20;
let origin = new THREE.Vector3();

let frequencies = Array(3).fill(0);
let waveLengths;

let amplitudes = Array(3).fill(0);

let sampleDistance;
let waveSampleDistance;
let cumulAmplitude;

let p;
let c;

function updateParticles() {
  if (!audioContext) return;
  if (!prevLowsArr) {
    prevLowsArr = Array(avgPeriod).fill(0);
    prevMidsArr = Array(avgPeriod).fill(0);
    prevHigsArr = Array(avgPeriod).fill(0);
  }

  prevLowsArr.shift();
  prevLowsArr.push(lowFreqArray[0]);

  prevMidsArr.shift();
  prevMidsArr.push(midFreqArray[0]);

  prevHigsArr.shift();
  prevHigsArr.push(higFreqArray[0]);

  frequencies[0] = average(prevLowsArr) / 20;
  frequencies[1] = average(prevMidsArr) / 20;
  frequencies[2] = average(prevHigsArr) / 20;

  waveLengths = frequencies.map((f) => size / f);

  for (let i = 0; i < particleCount; i++) {
    p = particleSystem.geometry.vertices[i];
    waveLengths.forEach((wv, idx) => {
      sampleDistance = p.distanceTo(origin);
      waveSampleDistance = sampleDistance % wv;
      amplitudes[idx] = Math.sin((waveSampleDistance / wv) * Math.PI * 2);
    });
    cumulAmplitude = amplitudes.reduce((a, b) => a + b) / amplitudes.length;

    c = cumulAmplitude > 0.4 ? 1 : 0;
    // c = 1 - cumulAmplitude;
    particleSystem.geometry.colors[i] = new THREE.Color(c * R, c * G, c * B);
  }
  particleSystem.geometry.colorsNeedUpdate = true;
}

```

## WebAudio API

### Play an audio file from URL (async)

```
let audioSrcURL = "<YOUR-AUDIO-SRC-URL>";
let audioContext = new AudioContext();
let resp = await fetch(audioSrcURL, { responseType: "arraybuffer" });
let audio = await resp.arrayBuffer();
let buffer = await audioContext.decodeAudioData(audio);
let audioSouceNode = audioContext.createBufferSource();
audioSouceNode.buffer = buffer;
audioSouceNode.connect(audioContext.destination);
audioSouceNode.start(0);
```

### Play an audio file from URL with a connected analyser node (async)

```
let audioSrcURL = "<YOUR-AUDIO-SRC-URL>";
let audioContext = new AudioContext();
let resp = await fetch(audioSrcURL, { responseType: "arraybuffer" });
let audio = await resp.arrayBuffer();
let buffer = await audioContext.decodeAudioData(audio);
let audioSouceNode = audioContext.createBufferSource();
audioSouceNode.buffer = buffer;
let analyser = audioContext.createAnalyser();
analyser.fftSize = 512;
audioSouceNode.connect(analyser);
analyser.connect(audioContext.destination);
audioSouceNode.start(0);
```

### Sample audio at regular intervals 

```
let ARR_SIZE = 16;
let array = new Uint8Array(ARR_SIZE * ARR_SIZE);

function sampleAudio() {
  if (!audioContext) return;
  analyser.getByteFrequencyData(array);
  console.log(array);
}

setInterval(sampleAudio, 2000);
```

### Process sample data to lows, mids and highs 

```
let ARR_SIZE = 16;
let array = new Uint8Array(ARR_SIZE * ARR_SIZE);
let lowFreqArray = new Uint8Array(ARR_SIZE * ARR_SIZE);
let midFreqArray = new Uint8Array(ARR_SIZE * ARR_SIZE);
let higFreqArray = new Uint8Array(ARR_SIZE * ARR_SIZE);
let firstBrk = 86;
let secondBrk = 170;
let lowIdx, midIdx, higIdx;

function sampleAudio() {
  if (!audioContext) return;
  lowIdx = 0;
  midIdx = 0;
  higIdx = 0;
  analyser.getByteFrequencyData(array);
  console.log(array);
  for (let p = 0; p < array.length; p++) {
    if (array[p] < firstBrk) {
      lowFreqArray[lowIdx] = array[p];
      lowIdx++;
    } else if (array[p] > secondBrk) {
      higFreqArray[higIdx] = array[p];
      higIdx++;
    } else {
      midFreqArray[midIdx] = array[p];
      midIdx++;
    }
  }
}
```

## Misc

### Basic nodejs server  
```
const http = require('http');

const requestListener = function (req, res) {
  res.writeHead(200);
  res.end('Hello, World!');
}

const server = http.createServer(requestListener);
server.listen(8080);
```
