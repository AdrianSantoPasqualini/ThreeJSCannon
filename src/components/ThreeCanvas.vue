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

      this.renderer.render(this.scene, this.camera);
    },
    initThree() {
      const canvReference = document.getElementById("three-canvas");

      this.scene = new THREE.Scene();
      this.scene.fog = new THREE.Fog(0x000000, 500, 1000);

      this.camera = new THREE.PerspectiveCamera(30, window.innerWidth / window.innerHeight, 0.5, 1000);
      this.camera.position.set(0, 2, 10);

      this.renderer = new THREE.WebGLRenderer({ canvas: canvReference });
      this.renderer.setSize( window.innerWidth, window.innerHeight );
      this.renderer.setClearColor(this.scene.fog.color)
      this.renderer.outputEncoding = THREE.sRGBEncoding
      this.renderer.shadowMap.enabled = true
      this.renderer.shadowMap.type = THREE.PCFSoftShadowMap

      // const controls = new OrbitControls(this.camera, canvReference);
      // controls.target.set(0, 0, 0);
      // controls.update();

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
      this.world = new CANNON.World({
        gravity: new CANNON.Vec3(0, -9.82, 0), // m/s²
      });
    },
    setFloor() {
      const groundBody = new CANNON.Body({
        mass: 0,
        shape: new CANNON.Plane(),
        material: new CANNON.Material({ restitution: 0.5 })
      })
      groundBody.quaternion.setFromEuler(-Math.PI / 2, 0, 0) // make it face up
      this.world.addBody(groundBody)

      const floorGeometry = new THREE.PlaneBufferGeometry(100, 100, 1, 1)
      const floorMaterial = new THREE.MeshLambertMaterial({ color: 0xff00ff })
      const floor = new THREE.Mesh(floorGeometry, floorMaterial)
      floor.receiveShadow = true
      this.scene.add(floor)

      this.objects.push({ body: groundBody, mesh: floor });
    },
    addBox() {
      const radius = 1 // m
      const sphereBody = new CANNON.Body({
        mass: 10, // kg
        shape: new CANNON.Sphere(radius),
        material: new CANNON.Material({ restitution: 1 })
      })
      sphereBody.position.set(0, 10, 0) // m
      this.world.addBody(sphereBody)

      const geometry = new THREE.SphereGeometry(radius)
      const material = new THREE.MeshPhongMaterial({ color: 0x00ff00 })
      const sphereMesh = new THREE.Mesh(geometry, material)
      sphereMesh.castShadow = true;
      this.scene.add(sphereMesh)

      this.objects.push({ body: sphereBody, mesh: sphereMesh });
    }
  },
  mounted() {
    this.initThree();
    this.initCannon();
    this.addBox();
    this.setFloor();
    this.animate();
  }
}
</script>

<style scoped>
#three-canvas {
  height: 100%;
  width: 100%;
}
</style>
