# MiniCraft-index.html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>MiniCraft</title>
  <style>
    html, body {
      margin: 0;
      overflow: hidden;
      background: #87ceeb;
      font-family: Arial, sans-serif;
    }

    #info {
      position: fixed;
      top: 12px;
      left: 12px;
      z-index: 10;
      color: white;
      background: rgba(0,0,0,.55);
      padding: 10px 14px;
      border-radius: 8px;
      line-height: 1.5;
    }

    #crosshair {
      position: fixed;
      left: 50%;
      top: 50%;
      transform: translate(-50%, -50%);
      z-index: 10;
      color: white;
      font-size: 24px;
      pointer-events: none;
    }

    #start {
      position: fixed;
      inset: 0;
      z-index: 20;
      display: flex;
      align-items: center;
      justify-content: center;
      background: rgba(0,0,0,.55);
      color: white;
      cursor: pointer;
      font-size: 24px;
    }
  </style>
</head>

<body>
  <div id="info">
    WASD = Move<br>
    Space = Jump<br>
    Left Click = Break block<br>
    Right Click = Place block<br>
    1 = Grass &nbsp; 2 = Stone &nbsp; 3 = Wood
  </div>

  <div id="crosshair">+</div>
  <div id="start">Click to Start</div>

  <script type="module">
    import * as THREE from
      "https://cdn.jsdelivr.net/npm/three@0.180.0/build/three.module.js";

    import { PointerLockControls } from
      "https://cdn.jsdelivr.net/npm/three@0.180.0/examples/jsm/controls/PointerLockControls.js";

    // -------------------------
    // Scene
    // -------------------------
    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x87ceeb);

    scene.fog = new THREE.Fog(0x87ceeb, 30, 100);

    // -------------------------
    // Camera
    // -------------------------
    const camera = new THREE.PerspectiveCamera(
      75,
      innerWidth / innerHeight,
      0.1,
      200
    );

    camera.position.set(0, 3, 6);

    // -------------------------
    // Renderer
    // -------------------------
    const renderer = new THREE.WebGLRenderer({
      antialias: true
    });

    renderer.setSize(innerWidth, innerHeight);
    renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
    renderer.shadowMap.enabled = true;

    document.body.appendChild(renderer.domElement);

    // -------------------------
    // Lights
    // -------------------------
    scene.add(new THREE.HemisphereLight(
      0xffffff,
      0x555555,
      2
    ));

    const sun = new THREE.DirectionalLight(
      0xffffff,
      2
    );

    sun.position.set(30, 50, 20);
    sun.castShadow = true;
    scene.add(sun);

    // -------------------------
    // Controls
    // -------------------------
    const controls = new PointerLockControls(
      camera,
      document.body
    );

    const start = document.getElementById("start");

    start.addEventListener("click", () => {
      controls.lock();
    });

    controls.addEventListener("lock", () => {
      start.style.display = "none";
    });

    controls.addEventListener("unlock", () => {
      start.style.display = "flex";
    });

    // -------------------------
    // Blocks
    // -------------------------
    const blocks = [];

    const geometry = new THREE.BoxGeometry(1, 1, 1);

    const materials = {
      grass: new THREE.MeshLambertMaterial({
        color: 0x55aa35
      }),

      dirt: new THREE.MeshLambertMaterial({
        color: 0x8b5a2b
      }),

      stone: new THREE.MeshLambertMaterial({
        color: 0x888888
      }),

      wood: new THREE.MeshLambertMaterial({
        color: 0x9b642f
      }),

      leaves: new THREE.MeshLambertMaterial({
        color: 0x2f8f2f
      })
    };

    let selectedMaterial = materials.grass;

    function addBlock(x, y, z, material) {
      const block = new THREE.Mesh(
        geometry,
        material
      );

      block.position.set(x, y, z);
      block.castShadow = true;
      block.receiveShadow = true;

      scene.add(block);
      blocks.push(block);

      return block;
    }

    // -------------------------
    // World
    // -------------------------
    const size = 20;

    for (let x = -size / 2; x < size / 2; x++) {
      for (let z = -size / 2; z < size / 2; z++) {

        addBlock(x, -1, z, materials.dirt);
        addBlock(x, 0, z, materials.grass);

        // small hills
        if (
          Math.random() < 0.08 &&
          Math.abs(x) < 8 &&
          Math.abs(z) < 8
        ) {
          addBlock(x, 1, z, materials.grass);
        }
      }
    }

    // -------------------------
    // Trees
    // -------------------------
    function makeTree(x, z) {

      const height = 3 + Math.floor(Math.random() * 2);

      for (let y = 1; y <= height; y++) {
        addBlock(x, y, z, materials.wood);
      }

      for (let dx = -2; dx <= 2; dx++) {
        for (let dz = -2; dz <= 2; dz++) {

          if (Math.abs(dx) + Math.abs(dz) <= 3) {
            addBlock(
              x + dx,
              height + 1,
              z + dz,
              materials.leaves
            );
          }
        }
      }
    }

    makeTree(-5, -4);
    makeTree(5, -2);
    makeTree(2, 6);

    // -------------------------
    // Player
    // -------------------------
    const player = {
      velocity: new THREE.Vector3(),
      canJump: false
    };

    const keys = {};

    document.addEventListener("keydown", e => {
      keys[e.code] = true;

      if (e.code === "Digit1") {
        selectedMaterial = materials.grass;
      }

      if (e.code === "Digit2") {
        selectedMaterial = materials.stone;
      }

      if (e.code === "Digit3") {
        selectedMaterial = materials.wood;
      }

      if (
        e.code === "Space" &&
        player.canJump
      ) {
        player.velocity.y = 8;
        player.canJump = false;
      }
    });

    document.addEventListener("keyup", e => {
      keys[e.code] = false;
    });

    // -------------------------
    // Raycaster
    // -------------------------
    const raycaster = new THREE.Raycaster();

    function getTargetBlock() {

      raycaster.setFromCamera(
        new THREE.Vector2(0, 0),
        camera
      );

      const hits = raycaster.intersectObjects(
        blocks,
        false
      );

      if (hits.length > 0 && hits[0].distance < 8) {
        return hits[0];
      }

      return null;
    }

    // Break / place
    document.addEventListener("mousedown", e => {

      if (!controls.isLocked) return;

      const hit = getTargetBlock();

      if (!hit) return;

      // Left click = break
      if (e.button === 0) {

        scene.remove(hit.object);

        const index = blocks.indexOf(hit.object);

        if (index !== -1) {
          blocks.splice(index, 1);
        }
      }

      // Right click = place
      if (e.button === 2) {

        const normal = hit.face.normal;

        const position =
          hit.object.position.clone().add(normal);

        addBlock(
          Math.round(position.x),
          Math.round(position.y),
          Math.round(position.z),
          selectedMaterial
        );
      }
    });

    document.addEventListener("contextmenu", e => {
      e.preventDefault();
    });

    // -------------------------
    // Movement
    // -------------------------
    const clock = new THREE.Clock();

    function updatePlayer(delta) {

      const speed = 7;

      const direction = new THREE.Vector3();

      if (keys["KeyW"]) direction.z -= 1;
      if (keys["KeyS"]) direction.z += 1;
      if (keys["KeyA"]) direction.x -= 1;
      if (keys["KeyD"]) direction.x += 1;

      if (direction.length() > 0) {
        direction.normalize();
        direction.multiplyScalar(speed * delta);

        controls.moveRight(direction.x);
        controls.moveForward(-direction.z);
      }

      // Gravity
      player.velocity.y -= 20 * delta;

      camera.position.y += player.velocity.y * delta;

      // Ground
      if (camera.position.y < 2.2) {
        camera.position.y = 2.2;
        player.velocity.y = 0;
        player.canJump = true;
      }
    }

    // -------------------------
    // Resize
    // -------------------------
    window.addEventListener("resize", () => {

      camera.aspect =
        innerWidth / innerHeight;

      camera.updateProjectionMatrix();

      renderer.setSize(
        innerWidth,
        innerHeight
      );
    });

    // -------------------------
    // Game Loop
    // -------------------------
    function animate() {

      requestAnimationFrame(animate);

      const delta =
        Math.min(clock.getDelta(), 0.05);

      if (controls.isLocked) {
        updatePlayer(delta);
      }

      renderer.render(
        scene,
        camera
      );
    }

    animate();
  </script>
</body>
</html>
