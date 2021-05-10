<template>
  <canvas id="three-canvas"></canvas>
</template>

<script>
import * as THREE from 'three';
// import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';

import * as CANNON from 'cannon-es';

export default {
  name: 'ThreeCanvas',
  data() {
    return {
      timeStep: 1/60,
      objects: [],

      // Player Movement
      controls: {
        moveForward: false,
        moveBackward: false,
        moveLeft: false,
        moveRight: false,
      },
      player: {
        body: undefined,
        mesh: undefined,
      },
      maxPlayerVelocity: 5,
      playerJumpVelocity: 5,
    };
  },
  methods: {
    animate() {
      requestAnimationFrame(this.animate);

      // Step CANNON time forward
      const time = performance.now() / 1000 // seconds
      if (!this.lastCallTime) {
        this.world.step(this.timeStep)
      } else {
        const dt = time - this.lastCallTime
        this.world.step(this.timeStep, dt)
      }
      this.lastCallTime = time

      // Track meshes to CANNON bodies
      for (const obj of this.objects) {
        obj.mesh.position.copy(obj.body.position)
        obj.mesh.quaternion.copy(obj.body.quaternion)
      }

      // Move the player
      this.movePlayer()

      // Update the Camera
      this.camera.position.x = this.player.mesh.position.x;
      this.camera.position.y = this.player.mesh.position.y + 20;
      this.camera.position.z = this.player.mesh.position.z + 50;
      this.camera.lookAt(this.player.mesh.position);

      this.renderer.render(this.scene, this.camera);
    },
    movePlayer() {
      // Apply the forces to move the player
      if (this.controls.moveForward) {
        this.player.body.velocity.z = -Math.abs(this.player.body.velocity.z);
        this.player.body.applyForce(new CANNON.Vec3(0, 0, -200));
      }
      if (this.controls.moveBackward) {
        this.player.body.velocity.z = Math.abs(this.player.body.velocity.z);
        this.player.body.applyForce(new CANNON.Vec3(0, 0, 200));
      }
      if (this.controls.moveLeft) {
        this.player.body.velocity.x = -Math.abs(this.player.body.velocity.x);
        this.player.body.applyForce(new CANNON.Vec3(-200, 0, 0));
      }
      if (this.controls.moveRight) {
        this.player.body.velocity.x = Math.abs(this.player.body.velocity.x);
        this.player.body.applyForce(new CANNON.Vec3(200, 0, 0));
      }

      // Cap player speed
      if (this.player.body.velocity.x > this.maxPlayerVelocity) {
        this.player.body.velocity.x = this.maxPlayerVelocity;
      }
      if (this.player.body.velocity.x < -this.maxPlayerVelocity) {
        this.player.body.velocity.x = -this.maxPlayerVelocity;
      }
      if (this.player.body.velocity.z > this.maxPlayerVelocity) {
        this.player.body.velocity.z = this.maxPlayerVelocity;
      }
      if (this.player.body.velocity.z < -this.maxPlayerVelocity) {
        this.player.body.velocity.z = -this.maxPlayerVelocity;
      }
    },
    initThree() {
      const canvReference = document.getElementById("three-canvas");

      // Init Scene
      this.scene = new THREE.Scene();
      this.scene.fog = new THREE.Fog(0x000000, 500, 1000);

      // Init Camera
      this.camera = new THREE.PerspectiveCamera(30, window.innerWidth / window.innerHeight, 0.5, 1000);
      this.camera.position.set(0, 2, 10);

      // Init Renderer
      this.renderer = new THREE.WebGLRenderer({ canvas: canvReference });
      this.renderer.setSize( window.innerWidth, window.innerHeight );
      this.renderer.setClearColor(this.scene.fog.color)
      this.renderer.outputEncoding = THREE.sRGBEncoding
      this.renderer.shadowMap.enabled = true
      this.renderer.shadowMap.type = THREE.PCFSoftShadowMap

      // OrbitControls to help debug
      // const controls = new OrbitControls(this.camera, canvReference);
      // controls.target.set(0, 0, 0);
      // controls.update();

      // Init Ambient and Directional light
      const light = new THREE.AmbientLight(0x666666);
      this.scene.add(light);

      const directionalLight = new THREE.DirectionalLight(0xffffff, 1.2)
      const distance = 200
      directionalLight.position.set(-distance, distance, distance)
      directionalLight.castShadow = true
      directionalLight.shadow.mapSize.width = 2048
      directionalLight.shadow.mapSize.height = 2048
      directionalLight.shadow.camera.left = -distance
      directionalLight.shadow.camera.right = distance
      directionalLight.shadow.camera.top = distance
      directionalLight.shadow.camera.bottom = -distance
      directionalLight.shadow.camera.far = 3 * distance
      directionalLight.shadow.camera.near = distance
      this.scene.add(directionalLight)
    },
    initCannon() {
      // Init CANNON world
      this.world = new CANNON.World({
        gravity: new CANNON.Vec3(0, -9.82, 0), // m/s²
      });
    },
    setFloor() {
      // Create floor CANNON body
      const groundBody = new CANNON.Body({
        mass: 0,
        shape: new CANNON.Plane(),
        material: new CANNON.Material({ restitution: 0.5 })
      })
      groundBody.quaternion.setFromEuler(-Math.PI / 2, 0, 0) // make it face up
      this.world.addBody(groundBody)

      // Create floor ThreeJS mesh
      const loader = new THREE.TextureLoader();
      const floorGeometry = new THREE.PlaneBufferGeometry(100, 100, 1, 1)
      const floorMaterial = new THREE.MeshBasicMaterial({map: loader.load('https://threejsfundamentals.org/threejs/resources/images/flower-1.jpg')})
      const floor = new THREE.Mesh(floorGeometry, floorMaterial)
      floor.receiveShadow = true
      this.scene.add(floor)

      // Store body/mesh pair
      this.objects.push({ body: groundBody, mesh: floor });
    },
    addBox() {
      // Create box CANNON body
      const radius = 1 // m
      const sphereBody = new CANNON.Body({
        mass: 10, // kg
        shape: new CANNON.Sphere(radius),
        material: new CANNON.Material({ restitution: 1 })
      })
      sphereBody.position.set(0, 10, 0) // m
      this.world.addBody(sphereBody)

      // Create box ThreeJS mesh
      const geometry = new THREE.SphereGeometry(radius)
      const material = new THREE.MeshPhongMaterial({ color: 0x00ff00 })
      const sphereMesh = new THREE.Mesh(geometry, material)
      sphereMesh.castShadow = true;
      this.scene.add(sphereMesh)

      // Store body/mesh pair
      this.objects.push({ body: sphereBody, mesh: sphereMesh });

      // Set the player (Temporary)
      this.player.body = sphereBody;
      this.player.mesh = sphereMesh;
    },
    onKeyDown(event) {
      // Allow WASD or arrow keys for movement
      switch (event.code) {
        case 'KeyW':
        case 'ArrowUp':
          this.controls.moveForward = true
          break
        case 'KeyA':
        case 'ArrowLeft':
          this.controls.moveLeft = true
          break
        case 'KeyS':
        case 'ArrowDown':
          this.controls.moveBackward = true
          break
        case 'KeyD':
        case 'ArrowRight':
          this.controls.moveRight = true
          break
        case 'Space':
          this.player.body.velocity.y = this.playerJumpVelocity
          break
      }
    },
    onKeyUp(event) {
      // Allow WASD or arrow keys for movement
      switch (event.code) {
        case 'KeyW':
        case 'ArrowUp':
          this.controls.moveForward = false
          break
        case 'KeyA':
        case 'ArrowLeft':
          this.controls.moveLeft = false
          break
        case 'KeyS':
        case 'ArrowDown':
          this.controls.moveBackward = false
          break
        case 'KeyD':
        case 'ArrowRight':
          this.controls.moveRight = false
          break
      }
    },
  },
  mounted() {
    this.initThree();
    this.initCannon();
    this.addBox();
    this.setFloor();
    this.animate();
    document.addEventListener('keydown', this.onKeyDown)
    document.addEventListener('keyup', this.onKeyUp)
  }
}
</script>

<style scoped>
#three-canvas {
  height: 100%;
  width: 100%;
}
</style>
