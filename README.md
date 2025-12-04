# IG-S11
La idea que se me ocurrió, fue desarrollar un juego que consiste en varias fichas colocadas contiguamente, como en el juego del dominó. 
El objetivo del juego es tirar todas las fichas fuera del suelo con un solo bolazo que impacte en todas ellas. 
Si queda alguna ficha en el tablero sea de la forma que sea no se consigue victoria. 

En el siguiente [video de prueba](https://youtu.be/-esUHufccsE) grabado podemos observar varios lanzamientos consecutivos hasta llegar al lanzamiento ganador que está al final del video.

Antes de nada comentar que todo el código de la práctica fue implementado en el CodeSandbox -> [IG-ENTREGA-S11](https://codesandbox.io/p/devbox/ig2526-s10-forked-tq66nv?workspaceId=ws_R33WtefLGa8Mx8YH5rgWSi)

# Desarrollo de la práctica

## 1. Descripción general del prototipo

Este prototipo implementa una escena 3D interactiva con three.js y el motor de física Ammo.js:

- Se crea un suelo con textura.

- Encima del suelo se genera una fila de fichas de dominó (cajas delgadas con cuerpo rígido).

- Al hacer clic con el ratón, se dispara una bola física en la dirección de la cámara.

- Si la bola impacta en las fichas, la gravedad y las colisiones gestionadas por Ammo.js hacen que caigan de forma realista.

Es un ejemplo de integración de gráficos (three.js) y simulación física (Ammo.js) aplicado al efecto dominó.

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

## 6. Creación de objetos de la escena
### 6.1 Creación de Objetos - `createObjects`
```js
function createObjects() {
  // Suelo
  pos.set(0, -0.5, 0);
  quat.set(0, 0, 0, 1);
  const suelo = createBoxWithPhysics(
    40,
    1,
    40,
    0, // masa 0 => estático
    pos,
    quat,
    new THREE.MeshPhongMaterial({ color: 0xffffff })
  );
  // Se le aplica una textura de rejilla
  ...
  // Línea de dominó
  createDominoLine();
}
```
- Se crea un suelo como caja grande (40×1×40) con masa 0 → cuerpo estático.

- Se le asocia una textura repetida para el aspecto visual.

- Luego se llama a la función `createDominoLine()` para generar la fila de fichas.

### 6.2 Fila de dominó - `createDominoLine`
```js
function createDominoLine() {
  const totalLength = (DOMINO_COUNT - 1) * DOMINO_SPACING;
  const zStart = -totalLength * 0.5;

  for (let i = 0; i < DOMINO_COUNT; i++) {
    const z = zStart + i * DOMINO_SPACING;

    pos.set(0, DOMINO_HEIGHT * 0.5, z);
    quat.set(0, 0, 0, 1);

    const mat = createMaterial();
    const domino = createBoxWithPhysics(
      DOMINO_THICKNESS,   // ancho
      DOMINO_HEIGHT,      // alto
      DOMINO_WIDTH,       // fondo
      DOMINO_MASS,
      pos,
      quat,
      mat
    );
    domino.castShadow = true;
    domino.receiveShadow = true;
  }
}
```
- Se calcula la posición inicial de la fila y se distribuyen las fichas en el eje Z.

- Cada ficha:

    - Es una caja delgada y alta, como una ficha de dominó.

    - Tiene una masa pequeña, por lo que es un cuerpo dinámico que puede caerse y chocar.

    - Se crea con `createBoxWithPhysics`, que automatiza la parte three.js + Ammo.

## 7. Enlace gráfico-físico
### 7.1 `createBoxWithPhysics`
```js
function createBoxWithPhysics(sx, sy, sz, mass, pos, quat, material) {
  const object = new THREE.Mesh(
    new THREE.BoxGeometry(sx, sy, sz, 1, 1, 1),
    material
  );
  const shape = new Ammo.btBoxShape(
    new Ammo.btVector3(sx * 0.5, sy * 0.5, sz * 0.5)
  );
  shape.setMargin(margin);

  createRigidBody(object, shape, mass, pos, quat);

  return object;
}
```
- Crea:

    - Una malla de three.js (la parte visual).

    - Una forma de colisión de Ammo (`btBoxShape`) con la mitad de las dimensiones (Ammo trabaja con semiejes).

- Llama a `createRigidBody` para:

    - Registrar el cuerpo en el mundo físico.

    - Vincular el `Mesh` de three.js con el `RigidBody` de Ammo.
 
### 7.2 `createRigidBody`
```js
function createRigidBody(object, physicsShape, mass, pos, quat, vel, angVel) {
  // Posición y rotación iniciales
  ...
  const transform = new Ammo.btTransform();
  transform.setIdentity();
  transform.setOrigin(new Ammo.btVector3(pos.x, pos.y, pos.z));
  transform.setRotation(new Ammo.btQuaternion(quat.x, quat.y, quat.z, quat.w));
  const motionState = new Ammo.btDefaultMotionState(transform);

  const localInertia = new Ammo.btVector3(0, 0, 0);
  if (mass > 0) {
    physicsShape.calculateLocalInertia(mass, localInertia);
  }

  const rbInfo = new Ammo.btRigidBodyConstructionInfo(
    mass,
    motionState,
    physicsShape,
    localInertia
  );
  const body = new Ammo.btRigidBody(rbInfo);

  body.setFriction(0.5);
  ...
  object.userData.physicsBody = body;
  ...
  scene.add(object);
  if (mass > 0) {
    rigidBodies.push(object);
    body.setActivationState(4); // disable deactivation
  }
  physicsWorld.addRigidBody(body);

  return body;
}
```
- Crea el cuerpo rígido de Ammo con:

    - masa, forma de colisión, inercia.

    - posición y orientación inicial.

- Enlaza:

    - `object.userData.physicsBody = body` → permite, más tarde, actualizar la malla con los datos físicos.

- Añade:

    - El objeto a la escena de three.js.

    - El cuerpo al mundo físico de Ammo.

    - Si la masa > 0, se añade también a `rigidBodies` para ser actualizado en cada frame.
 
## 8. Interacción - `initInput()` y disparo de bolas
```js
function initInput() {
  window.addEventListener("pointerdown", function (event) {
    mouseCoords.set(
      (event.clientX / window.innerWidth) * 2 - 1,
      -(event.clientY / window.innerHeight) * 2 + 1
    );

    raycaster.setFromCamera(mouseCoords, camera);

    const ballMass = 35;
    const ballRadius = 0.4;
    const ball = new THREE.Mesh(
      new THREE.SphereGeometry(ballRadius, 14, 10),
      ballMaterial
    );
    ...
    const ballShape = new Ammo.btSphereShape(ballRadius);
    ballShape.setMargin(margin);

    pos.copy(raycaster.ray.direction);
    pos.add(raycaster.ray.origin);
    quat.set(0, 0, 0, 1);
    const ballBody = createRigidBody(ball, ballShape, ballMass, pos, quat);

    pos.copy(raycaster.ray.direction);
    pos.multiplyScalar(24);
    ballBody.setLinearVelocity(new Ammo.btVector3(pos.x, pos.y, pos.z));
  });
}
```
- Al hacer clic:

    - Se calcula la dirección del rayo de la cámara (`raycaster`).

    - Se crea una esfera (malla + cuerpo físico).

    - Se coloca un poco delante de la cámara (`ray.origin + direction`).

    - Se le da una velocidad lineal fuerte en la dirección del rayo.

- Resultado: una bala física que avanza y, si impacta contra las fichas, las tira.

## 9. Bucle de animación y actualización de la física

### 9.1. `animationLoop()`

```js
function animationLoop() {
  requestAnimationFrame(animationLoop);

  const deltaTime = clock.getDelta();
  updatePhysics(deltaTime);

  renderer.render(scene, camera);
}
```
- Es el bucle principal:

    - Calcula el tiempo transcurrido entre frames.

    - Llama a updatePhysics para avanzar la simulación.

    - Renderiza la escena con la cámara.

### 9.2. `updatePhysics(deltaTime)`

```js
function updatePhysics(deltaTime) {
  physicsWorld.stepSimulation(deltaTime, 10);

  for (let i = 0, il = rigidBodies.length; i < il; i++) {
    const objThree = rigidBodies[i];
    const objPhys = objThree.userData.physicsBody;
    const ms = objPhys.getMotionState();
    if (ms) {
      ms.getWorldTransform(transformAux1);
      const p = transformAux1.getOrigin();
      const q = transformAux1.getRotation();
      objThree.position.set(p.x(), p.y(), p.z());
      objThree.quaternion.set(q.x(), q.y(), q.z(), q.w());
      objThree.userData.collided = false;
    }
  }
}
```
- `physicsWorld.stepSimulation(deltaTime, 10)`:

    - Avanza la simulación física en función del tiempo.

- Para cada objeto de `rigidBodies`:

    - Se obtiene la transformación del cuerpo físico (posición + rotación).

    - Se actualiza la malla de three.js para que coincida con la simulación.

- De esta manera, lo que ves en pantalla es la representación gráfica del mundo físico.

## 10. Resumen
Este prototipo implementa una escena 3D interactiva con three.js y Ammo.js que simula una fila de fichas de dominó.
El suelo y las fichas se crean como mallas de three.js a las que se asocia un cuerpo rígido de Ammo (caja con masa para las fichas y estática para el suelo).

El usuario puede disparar esferas físicas en la dirección de la cámara.
Cuando la bola impacta en la primera ficha, el resto caen de forma encadenada reproduciendo el efecto dominó. 
El objetivo del juego es tirar todas las fichas fuera del suelo.

