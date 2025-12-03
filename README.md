# IG-S11
La idea que se me ocurrió, fue desarrollar un juego que consiste en varias fichas colocadas seguidas del estilo dominó. 
El objetivo del juego es tirar todas las fichas fuera del tablero con un solo bolazo que impacte en todas ellas. 
Si queda alguna ficha en el tablero sea de la forma que sea no se consigue victoria. 

En el video grabado podemos observar varios lanzamientos consecutivos hasta llegar al lanzamiento ganador que está al final del video. El video fue entregado en el campus virtual con el enlace de este github.

Antes de nada comentar que todo el código de la práctica fue implementado en el CodeSandbox -> [IG-ENTREGA-S11](https://codesandbox.io/p/devbox/ig2526-s10-forked-tq66nv?workspaceId=ws_R33WtefLGa8Mx8YH5rgWSi)

# Desarrollo de la práctica
## 1. Descripción general del prototipo

Este prototipo implementa una escena 3D interactiva con three.js y el motor de física Ammo.js:

- Se crea un suelo con textura.

- Encima del suelo se genera una fila de fichas de dominó (cajas delgadas con cuerpo rígido).

- Al hacer clic con el ratón, se dispara una bola física en la dirección de la cámara.

- Si la bola impacta en las fichas, la gravedad y las colisiones gestionadas por Ammo.js hacen que caigan de forma realista.

Es un ejemplo de integración de gráficos (three.js) y simulación física (Ammo.js) aplicado al clásico efecto dominó.

## 2. Tecnologías utilizadas

- three.js

    - Renderizado 3D en WebGL.

    - Cámara, luces, materiales, geometrías y OrbitControls para mover la cámara con el ratón.

- Ammo.js (ammojs-typed)

  - Motor de física basado en Bullet.

  - Manejo de cuerpos rígidos, gravedad, colisiones e integración temporal del mundo físico.

## 3. Estructura general del código

Ahora pasaré a explicar el código implementado en el [domino_ammo.js](https://github.com/guillecab7/IG-S11/blob/main/domino_ammo.js) y sus utilidades.

### 3.1 Importaciones y variables globales
```js
import * as THREE from "three";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls";
import Ammo from "ammojs-typed";

let camera, controls, scene, renderer;
let textureLoader;
const clock = new THREE.Clock();

const mouseCoords = new THREE.Vector2();
const raycaster = new THREE.Raycaster();
const ballMaterial = new THREE.MeshPhongMaterial({ color: 0x202020 });

// Mundo físico con Ammo
let physicsWorld;
const gravityConstant = 7.8;
let collisionConfiguration;
let dispatcher;
let broadphase;
let solver;
const margin = 0.05; //margen colisiones

// Objetos rígidos
const rigidBodies = [];

const pos = new THREE.Vector3();
const quat = new THREE.Quaternion();
//Variables temporales para actualizar transformación en el bucle
let transformAux1;
let tempBtVec3_1;

// --- Parámetros del dominó ---
const DOMINO_COUNT = 25;
const DOMINO_SPACING = 0.8; // separación entre fichas
const DOMINO_THICKNESS = 0.2;
const DOMINO_HEIGHT = 1.0;
const DOMINO_WIDTH = 0.5;
const DOMINO_MASS = 0.4;
```
- Se importan three.js, los controles de órbita y Ammo.

- Después se declaran variables globales para:

    - la escena (`scene`), cámara (`camera`), renderer (`renderer`) y controles (`controls`)

    - el mundo físico (`physicsWorld`) y sus componentes (configuración de colisiones, solver, etc.)

    - una lista `rigidBodies` con todos los objetos gráficos que tienen cuerpo físico asociado.

    - constantes del dominó: número de fichas, medidas y masa.

### 3.2 Carga de Ammo y arranque
```js
Ammo(Ammo).then(start);
```
- Ammo se carga de forma asíncrona.

- Cuando está listo, se llama a `start()`, donde se inicializa todo:

```js
function start() {
  initGraphics();
  initPhysics();
  createObjects();
  initInput();
  animationLoop();
}
```
## 4. Parte gráfica
En `initGraphics()` se configura toda la parte visual
```js
function initGraphics() {
  //Cámara, escena, renderer y control de cámara
  camera = new THREE.PerspectiveCamera(
    60,
    window.innerWidth / window.innerHeight,
    0.2,
    2000
  );
  scene = new THREE.Scene();
  scene.background = new THREE.Color(0xbfd1e5);
  camera.position.set(-14, 8, 16);

  renderer = new THREE.WebGLRenderer();
  renderer.setPixelRatio(window.devicePixelRatio);
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.shadowMap.enabled = true;
  document.body.appendChild(renderer.domElement);

  controls = new OrbitControls(camera, renderer.domElement);
  controls.target.set(0, 2, 0);
  controls.update();

  textureLoader = new THREE.TextureLoader();

  //Luces
  const ambientLight = new THREE.AmbientLight(0x707070);
  scene.add(ambientLight);

  const light = new THREE.DirectionalLight(0xffffff, 1);
  light.position.set(-10, 18, 5);
  light.castShadow = true;
  const d = 14;
  light.shadow.camera.left = -d;
  light.shadow.camera.right = d;
  light.shadow.camera.top = d;
  light.shadow.camera.bottom = -d;

  light.shadow.camera.near = 2;
  light.shadow.camera.far = 50;

  light.shadow.mapSize.x = 1024;
  light.shadow.mapSize.y = 1024;

  scene.add(light);
  //Redimensión de la ventana
  window.addEventListener("resize", onWindowResize);
}
```
- __Cámara__: `PerspectiveCamera` con FOV 60º y posición inicial en `(-14, 8, 16)`.

- __Escena__: fondo de color azul claro.

- __Renderer__: se activa `shadowMap` para que los objetos proyecten sombra y el canvas se añade al `document.body`.

- __OrbitControls__: permiten rotar la cámara alrededor de la escena con el ratón.

- __Luces__:

    - `AmbientLight` para iluminación base.

    - `DirectionalLight` con sombras, ajustando el volumen de cámara de sombras.

Esto prepara todo para poder renderizar los objetos 3D.

## 5. Parte física
```js
function initPhysics() {
  // Configuración Ammo
  // Colisiones
  collisionConfiguration = new Ammo.btDefaultCollisionConfiguration();
  // Gestor de colisiones convexas y cóncavas
  dispatcher = new Ammo.btCollisionDispatcher(collisionConfiguration);
  // Colisión fase amplia
  broadphase = new Ammo.btDbvtBroadphase();
  // Resuelve restricciones de reglas físicas como fuerzas, gravedad, etc.
  solver = new Ammo.btSequentialImpulseConstraintSolver();
  // Crea el mundo físico
  physicsWorld = new Ammo.btDiscreteDynamicsWorld(
    dispatcher,
    broadphase,
    solver,
    collisionConfiguration
  );
  // Establece gravedad
  physicsWorld.setGravity(new Ammo.btVector3(0, -gravityConstant, 0));

  transformAux1 = new Ammo.btTransform();
  tempBtVec3_1 = new Ammo.btVector3(0, 0, 0);
}
```
- Se crea el mundo de física de Ammo:

    - configuración de colisiones,

    - gestor de colisiones,

    - fase amplia para detección de colisiones,

    - solver de restricciones.

- Se fija la gravedad en el eje Y (negativo).

- Se preparan transformaciones temporales (`transformAux1`, `tempBtVec3_1`) que se usan luego para leer la posición/rotación de los cuerpos físicos en cada frame.
```js

```

```js

```

```js

```

```js

```

```js

```

```js

```

```js

```

```js

```

```js

```

```js

```

```js

```

```js

```

```js

```

```js

```

```js

```

```js

```

```js

```


