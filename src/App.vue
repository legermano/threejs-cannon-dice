<script setup>
import { onMounted } from "vue";
import {
  IcosahedronGeometry,
  Mesh,
  MeshPhysicalMaterial,
  DodecahedronGeometry,
  BoxGeometry,
  OctahedronGeometry,
  TetrahedronGeometry,
  AxesHelper,
  AmbientLight,
  DirectionalLight,
  DirectionalLightHelper,
  PlaneGeometry,
  BufferGeometry,
} from "three";
import { mergeVertices } from "three/examples/jsm/utils/BufferGeometryUtils";
import { Vec3, ConvexPolyhedron, Body, Box, World, Plane } from "cannon-es";
import { GUI } from "dat.gui";
import CannonDebugger from "cannon-es-debugger";
import SceneInit from "./lib/SceneInit";

const defaultMass = 1;
const updateObjectsPositionFunctions = [];
let speed = 0;
let maxSpeed = 0.1;
let acceleration = 0.0025;
let angle = 0;

const getPolyhedronShape = (mesh) => {
  let geometry = new BufferGeometry();
  geometry.setAttribute("position", mesh.geometry.getAttribute("position"));

  geometry = mergeVertices(geometry);

  const position = geometry.attributes.position.array;
  const index = geometry.index.array;

  const points = [];
  for (let i = 0; i < position.length; i += 3) {
    points.push(new Vec3(position[i], position[i + 1], position[i + 2]));
  }
  const faces = [];
  for (let i = 0; i < index.length; i += 3) {
    faces.push([index[i], index[i + 1], index[i + 2]]);
  }

  return new ConvexPolyhedron({ vertices: points, faces });
};

const createPolyhedronPhysic = (polyhedronMesh) => {
  // Cast shadow
  polyhedronMesh.castShadow = true;

  // Create physics object
  const shape = getPolyhedronShape(polyhedronMesh);

  const polyhedronBody = new Body({ mass: defaultMass, shape });

  const updatePolyhedronPosition = () => {
    polyhedronMesh.position.copy(polyhedronBody.position);
    polyhedronMesh.quaternion.copy(polyhedronBody.quaternion);
  };

  return {
    body: polyhedronBody,
    mesh: polyhedronMesh,
    updatePosition: updatePolyhedronPosition,
  };
};

const createIcosahedron = () => {
  // Create render object
  const icosahedronGeometry = new IcosahedronGeometry(1, 0);
  const icosahedronMesh = new Mesh(
    icosahedronGeometry,
    new MeshPhysicalMaterial({ color: 0xff0000 })
  );

  return createPolyhedronPhysic(icosahedronMesh);
};

const createDedecahedron = () => {
  // Create render object
  const dodecahedronGeometry = new DodecahedronGeometry(1, 0);
  const dodecahedronMesh = new Mesh(
    dodecahedronGeometry,
    new MeshPhysicalMaterial({ color: 0xffff00 })
  );

  return createPolyhedronPhysic(dodecahedronMesh);
};

const createOctahedron = () => {
  // Create render object
  const octahedronGeometry = new OctahedronGeometry(1, 0);
  const octahedronMesh = new Mesh(
    octahedronGeometry,
    new MeshPhysicalMaterial({ color: 0x00ff00 })
  );

  return createPolyhedronPhysic(octahedronMesh);
};

const createTetrahedron = () => {
  // Create render object
  const tetrahedronGeometry = new TetrahedronGeometry(1, 0);
  const tetrahedronMesh = new Mesh(
    tetrahedronGeometry,
    new MeshPhysicalMaterial({ color: 0x9900ff })
  );

  return createPolyhedronPhysic(tetrahedronMesh);
};

const createCube = () => {
  const cubeGeometry = new BoxGeometry(1.5, 1.5, 1.5);
  const cubeMesh = new Mesh(
    cubeGeometry,
    new MeshPhysicalMaterial({ color: 0x0000ff })
  );
  cubeMesh.castShadow = true;

  // Create physic obeject
  const cubeBody = new Body({
    mass: defaultMass,
    shape: new Box(new Vec3(0.75, 0.75, 0.75)),
  });

  const updateCubePosition = () => {
    cubeMesh.position.copy(cubeBody.position);
    cubeMesh.quaternion.copy(cubeBody.quaternion);
  };

  return {
    body: cubeBody,
    mesh: cubeMesh,
    updatePosition: updateCubePosition,
  };
};

onMounted(() => {
  const test = new SceneInit("physicsTest");
  test.initialize();
  test.animate();

  // Initialize gui
  const gui = new GUI();

  // Add axes helper
  const axesHelper = new AxesHelper(8);
  test.scene.add(axesHelper);

  // Set up ambient light
  const ambientLight = new AmbientLight(0xffffff, 0.5);
  test.scene.add(ambientLight);

  // Set up ambient light gui
  const alFolder = gui.addFolder("ambient light");
  const alSettings = { color: ambientLight.color.getHex() };
  alFolder.add(ambientLight, "visible");
  alFolder.add(ambientLight, "intensity", 0, 1, 0.1);
  alFolder
    .addColor(alSettings, "color")
    .onChange((value) => ambientLight.color.set.value(value));
  alFolder.open();

  // Set up directional light + helper
  const directionalLight = new DirectionalLight(0xff0000, 0.5);
  directionalLight.position.set(0, 10, 0);
  directionalLight.castShadow = true;
  const directionalLightHelper = new DirectionalLightHelper(
    directionalLight,
    3
  );
  directionalLightHelper.visible = false;
  test.scene.add(directionalLight);
  test.scene.add(directionalLightHelper);

  // Set up directional light gui
  const dlSettings = { visible: false, color: directionalLight.color.getHex() };
  const dlFolder = gui.addFolder("directional light");
  dlFolder.add(dlSettings, "visible").onChange((value) => {
    directionalLight.visable = value;
    directionalLightHelper.visible = value;
  });
  dlFolder.add(directionalLight, "intensity", 0, 1, 0.25);
  dlFolder.add(directionalLight.position, "x", -20, 20, 0.5);
  dlFolder.add(directionalLight.position, "y", -20, 20, 0.5);
  dlFolder.add(directionalLight.position, "z", -20, 20, 0.5);
  dlFolder.add(directionalLight, "castShadow");
  dlFolder
    .addColor(dlSettings, "color")
    .onChange((value) => directionalLight.color.set(value));
  dlFolder.open();

  //  Set up world physics with a few objects
  const physicsWorld = new World({
    gravity: new Vec3(0, -9.82, 0),
  });

  const groundBody = new Body({
    type: Body.STATIC,
    // Infinite geometric plane
    shape: new Plane(),
  });

  // Rotate ground by 90 degrees
  groundBody.quaternion.setFromEuler(-Math.PI / 2, 0, 0);
  physicsWorld.addBody(groundBody);

  const groundGeometry = new PlaneGeometry(25, 25);
  const groundMaterial = new MeshPhysicalMaterial({ color: 0xfafafa });
  const groundMesh = new Mesh(groundGeometry, groundMaterial);
  groundMesh.receiveShadow = true;

  // Sync ground position and quaternion to physic world
  groundMesh.position.copy(groundBody.position);
  groundMesh.quaternion.copy(groundBody.quaternion);

  test.scene.add(groundMesh);

  const {
    body: icoBody1,
    mesh: icoMesh1,
    updatePosition: updateIcoPos1,
  } = createIcosahedron();
  updateObjectsPositionFunctions.push(updateIcoPos1);
  icoBody1.position.set(0, 10, 0);
  test.scene.add(icoMesh1);
  physicsWorld.addBody(icoBody1);

  const {
    body: dodeBody1,
    mesh: dodeMesh1,
    updatePosition: updateDodePos1,
  } = createDedecahedron();
  updateObjectsPositionFunctions.push(updateDodePos1);
  dodeBody1.position.set(1, 7, 0);
  test.scene.add(dodeMesh1);
  physicsWorld.addBody(dodeBody1);

  const {
    body: octaBody1,
    mesh: octaMesh1,
    updatePosition: updateOctaPos1,
  } = createOctahedron();
  updateObjectsPositionFunctions.push(updateOctaPos1);
  octaBody1.position.set(2, 8, 0);
  test.scene.add(octaMesh1);
  physicsWorld.addBody(octaBody1);

  const {
    body: cubeBody1,
    mesh: cubeMesh1,
    updatePosition: updateCubePosition1,
  } = createCube();
  updateObjectsPositionFunctions.push(updateCubePosition1);
  cubeBody1.position.set(2, 5, 0);
  test.scene.add(cubeMesh1);
  physicsWorld.addBody(cubeBody1);

  const {
    body: tetraBody1,
    mesh: tetraMesh1,
    updatePosition: updateTetraPosition1,
  } = createTetrahedron();
  updateObjectsPositionFunctions.push(updateTetraPosition1);
  tetraBody1.position.set(-1, 11, 0);
  test.scene.add(tetraMesh1);
  physicsWorld.addBody(tetraBody1);

  // Key controls of icosahedron
  document.onkeydown = (event) => {
    switch (event.key) {
      case "a":
      case "ArrowLeft":
        angle += Math.PI / 180;
        break;
      case "d":
      case "ArrowRight":
        angle -= Math.PI / 180;
        break;
      case "w":
      case "ArrowUp":
        speed += acceleration;
        break;
      case "s":
      case "ArrowDown":
        speed -= acceleration;
        break;
    }
  };

  const moveIcosahedron = () => {
    // Set max speed forward
    if (speed > maxSpeed) {
      speed = maxSpeed;
    }

    // Set max speed backward
    if (speed < -maxSpeed) {
      speed = -maxSpeed;
    }

    // Update position based on unit circle
    icoBody1.position.x += speed * Math.sin(angle);
    icoBody1.position.z += speed * Math.cos(angle);
  };

  const cannonDebugger = new CannonDebugger(test.scene, physicsWorld);

  const animate = () => {
    physicsWorld.fixedStep();
    // cannonDebugger.update();
    moveIcosahedron();
    updateObjectsPositionFunctions.forEach((f) => f());
    window.requestAnimationFrame(animate);
  };

  animate();
});
</script>

<template>
  <div>
    <canvas id="physicsTest" />
  </div>
</template>
