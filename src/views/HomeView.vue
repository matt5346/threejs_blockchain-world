<template>
  <div class="home">
    <div id="globeMap" class="globe"></div>
    <div id="blocker">
      <div id="instructions">
        <p style="font-size:36px">
          Click to play
        </p>
        <p>
          Move: WASD<br/>
          Jump: SPACE<br/>
          Look: MOUSE
        </p>
      </div>
    </div>
  </div>
</template>

<script>
// import Globe from 'globe.gl'
import * as THREE from 'three'

import { PointerLockControls } from '../controls/PointerLockControls.js'

// const TILE_MARGIN = 0.35 // degrees

// Gen random data
const GRID_SIZE = [30, 10]
const COLORS = ['red', 'green', 'yellow', 'blue', 'orange', 'pink', 'brown', 'purple', 'magenta']

const materials = COLORS.map(color => new THREE.MeshLambertMaterial({ color, opacity: 0.6, transparent: true }))
const tileWidth = 360 / GRID_SIZE[0]
const tileHeight = 180 / GRID_SIZE[1]

export default {
  name: 'HomeView',

  data: () => ({
    tilesData: []
  }),

  methods: {
    getTiles () {
      [...Array(GRID_SIZE[0]).keys()].forEach(lngIdx =>
        [...Array(GRID_SIZE[1]).keys()].forEach(latIdx =>
          this.tilesData.push({
            lng: -180 + lngIdx * tileWidth,
            lat: -90 + (latIdx + 0.5) * tileHeight,
            material: materials[Math.floor(Math.random() * materials.length)]
          })
        )
      )
    }
  },

  mounted () {
    // this.getTiles()

    // Globe()
    //   .tilesData(this.tilesData)
    //   .tileWidth(tileWidth - TILE_MARGIN)
    //   .tileHeight(tileHeight - TILE_MARGIN)
    //   .tileMaterial('material')(document.getElementById('globeMap'))

    let camera, scene, renderer, controls

    const objects = []

    let raycaster

    let moveForward = false
    let moveBackward = false
    let moveLeft = false
    let moveRight = false
    let canJump = false

    let prevTime = performance.now()
    const velocity = new THREE.Vector3()
    const direction = new THREE.Vector3()
    const vertex = new THREE.Vector3()
    const color = new THREE.Color()

    init()
    animate()

    function init () {
      camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 1, 1000)
      camera.position.y = 10

      scene = new THREE.Scene()
      scene.background = new THREE.Color(0xffffff)
      scene.fog = new THREE.Fog(0xffffff, 0, 750)

      const light = new THREE.HemisphereLight(0xeeeeff, 0x777788, 0.75)
      light.position.set(0.5, 1, 0.75)
      scene.add(light)

      controls = new PointerLockControls(camera, document.body)

      const blocker = document.getElementById('blocker')
      const instructions = document.getElementById('instructions')

      instructions.addEventListener('click', function () {
        controls.lock()
      })

      controls.addEventListener('lock', function () {
        instructions.style.display = 'none'
        blocker.style.display = 'none'
      })

      controls.addEventListener('unlock', function () {
        blocker.style.display = 'block'
        instructions.style.display = ''
      })

      scene.add(controls.getObject())

      const onKeyDown = function (event) {
        switch (event.code) {
          case 'ArrowUp':
          case 'KeyW':
            moveForward = true
            break

          case 'ArrowLeft':
          case 'KeyA':
            moveLeft = true
            break

          case 'ArrowDown':
          case 'KeyS':
            moveBackward = true
            break

          case 'ArrowRight':
          case 'KeyD':
            moveRight = true
            break

          case 'Space':
            if (canJump === true) velocity.y += 900
            canJump = false
            break
        }
      }

      const onKeyUp = function (event) {
        switch (event.code) {
          case 'ArrowUp':
          case 'KeyW':
            moveForward = false
            break

          case 'ArrowLeft':
          case 'KeyA':
            moveLeft = false
            break

          case 'ArrowDown':
          case 'KeyS':
            moveBackward = false
            break

          case 'ArrowRight':
          case 'KeyD':
            moveRight = false
            break
        }
      }

      document.addEventListener('keydown', onKeyDown)
      document.addEventListener('keyup', onKeyUp)

      raycaster = new THREE.Raycaster(new THREE.Vector3(), new THREE.Vector3(0, -1, 0), 0, 10)

      // floor

      let floorGeometry = new THREE.PlaneGeometry(2000, 2000, 100, 100)
      floorGeometry.rotateX(-Math.PI / 2)

      // vertex displacement

      let position = floorGeometry.attributes.position

      for (let i = 0, l = position.count; i < l; i++) {
        vertex.fromBufferAttribute(position, i)

        vertex.x += Math.random() * 20 - 10
        vertex.y += Math.random() * 2
        vertex.z += Math.random() * 20 - 10

        position.setXYZ(i, vertex.x, vertex.y, vertex.z)
      }

      floorGeometry = floorGeometry.toNonIndexed() // ensure each face has unique vertices

      position = floorGeometry.attributes.position
      const colorsFloor = []

      for (let i = 0, l = position.count; i < l; i++) {
        color.setHSL(Math.random() * 0.3 + 0.5, 0.75, Math.random() * 0.25 + 0.75)
        colorsFloor.push(color.r, color.g, color.b)
      }

      floorGeometry.setAttribute('color', new THREE.Float32BufferAttribute(colorsFloor, 3))

      const floorMaterial = new THREE.MeshBasicMaterial({ vertexColors: true })

      const floor = new THREE.Mesh(floorGeometry, floorMaterial)
      scene.add(floor)

      // objects

      const boxGeometry = new THREE.BoxGeometry(20, 20, 20).toNonIndexed()

      position = boxGeometry.attributes.position
      const colorsBox = []

      for (let i = 0, l = position.count; i < l; i++) {
        color.setHSL(Math.random() * 0.3 + 0.5, 0.75, Math.random() * 0.25 + 0.75)
        colorsBox.push(color.r, color.g, color.b)
      }

      boxGeometry.setAttribute('color', new THREE.Float32BufferAttribute(colorsBox, 3))

      for (let i = 0; i < 50; i++) {
        const boxMaterial = new THREE.MeshPhongMaterial({ specular: 0xffffff, flatShading: true, vertexColors: true })
        boxMaterial.color.setHSL(Math.random() * 0.2 + 0.5, 0.75, Math.random() * 0.25 + 0.75)

        const box = new THREE.Mesh(boxGeometry, boxMaterial)
        box.position.x = Math.floor(Math.random() * 20 - 10) * 20
        box.position.y = Math.floor(Math.random() * 20) * 20 + 10
        box.position.z = Math.floor(Math.random() * 20 - 10) * 20

        scene.add(box)
        objects.push(box)
      }

      const geometry = new THREE.SphereGeometry(50, 32, 16)
      // const images = []

      const loader = new THREE.TextureLoader()
      loader.load('/img/monkey1.png', function (texture) {
        console.log(texture, 'TEXTURE')
        const geometryImage = new THREE.SphereGeometry(65, 20, 20)

        const material = new THREE.MeshBasicMaterial({ map: texture, overdraw: 0.5 })
        const mesh = new THREE.Mesh(geometryImage, material)
        mesh.position.x = Math.floor(30) * 20
        mesh.position.y = Math.floor(100)
        scene.add(mesh)
      })
      const material = new THREE.MeshBasicMaterial({ color: 0x00ffaa, wireframe: true })
      const sphere = new THREE.Mesh(geometry, material)
      sphere.position.x = Math.floor(10) * 20
      sphere.position.y = Math.floor(70)

      const geometry2 = new THREE.SphereGeometry(35, 32, 16)
      const sphere2 = new THREE.Mesh(geometry2, material)
      sphere2.position.x = Math.floor(15) * 20
      sphere2.position.y = Math.floor(70)
      sphere2.position.z = Math.floor(100)

      const geometry3 = new THREE.SphereGeometry(50, 32, 16)
      const sphere3 = new THREE.Mesh(geometry3, material)
      sphere3.position.x = Math.floor(20) * 20
      sphere3.position.y = Math.floor(70)

      scene.add(sphere)
      scene.add(sphere2)
      scene.add(sphere3)

      renderer = new THREE.WebGLRenderer({ antialias: true })
      renderer.setPixelRatio(window.devicePixelRatio)
      renderer.setSize(window.innerWidth, window.innerHeight)
      document.body.appendChild(renderer.domElement)

      //

      window.addEventListener('resize', onWindowResize)
    }

    function onWindowResize () {
      camera.aspect = window.innerWidth / window.innerHeight
      camera.updateProjectionMatrix()

      renderer.setSize(window.innerWidth, window.innerHeight)
    }

    function animate () {
      requestAnimationFrame(animate)

      const time = performance.now()

      if (controls.isLocked === true) {
        raycaster.ray.origin.copy(controls.getObject().position)
        raycaster.ray.origin.y -= 10

        const intersections = raycaster.intersectObjects(objects, false)

        const onObject = intersections.length > 0

        const delta = (time - prevTime) / 1000

        velocity.x -= velocity.x * 1.0 * delta
        velocity.z -= velocity.z * 1.0 * delta

        velocity.y -= 9.8 * 100.0 * delta // 100.0 = mass

        direction.z = Number(moveForward) - Number(moveBackward)
        direction.x = Number(moveRight) - Number(moveLeft)
        direction.normalize() // this ensures consistent movements in all directions

        if (moveForward || moveBackward) velocity.z -= direction.z * 400.0 * delta
        if (moveLeft || moveRight) velocity.x -= direction.x * 400.0 * delta

        if (onObject === true) {
          velocity.y = Math.max(0, velocity.y)
          canJump = true
        }

        controls.moveRight(-velocity.x * delta)
        controls.moveForward(-velocity.z * delta)

        controls.getObject().position.y += (velocity.y * delta) // new behavior

        if (controls.getObject().position.y < 10) {
          velocity.y = 0
          controls.getObject().position.y = 10

          canJump = true
        }
      }

      prevTime = time

      renderer.render(scene, camera)
    }
  }
}
</script>

<style scoped>
  body {
  margin: 0;
  background-color: #000;
  color: #fff;
  font-family: Monospace;
  font-size: 13px;
  line-height: 24px;
  overscroll-behavior: none;
}

a {
  color: #ff0;
  text-decoration: none;
}

a:hover {
  text-decoration: underline;
}

button {
  cursor: pointer;
  text-transform: uppercase;
}

#info {
  position: absolute;
  top: 0px;
  width: 100%;
  padding: 10px;
  box-sizing: border-box;
  text-align: center;
  -moz-user-select: none;
  -webkit-user-select: none;
  -ms-user-select: none;
  user-select: none;
  pointer-events: none;
  z-index: 1; /* TODO Solve this in HTML */
}

a, button, input, select {
  pointer-events: auto;
}

.lil-gui {
  z-index: 2 !important; /* TODO Solve this in HTML */
}

@media all and ( max-width: 640px ) {
.lil-gui.root {
  right: auto;
  top: auto;
  max-height: 50%;
  max-width: 80%;
  bottom: 0;
  left: 0;
}
}

#overlay {
  position: absolute;
  font-size: 16px;
  z-index: 2;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-direction: column;
  background: rgba(0,0,0,0.7);
}

#overlay button {
  background: transparent;
  border: 0;
  border: 1px solid rgb(255, 255, 255);
  border-radius: 4px;
  color: #ffffff;
  padding: 12px 18px;
  text-transform: uppercase;
  cursor: pointer;
}

#notSupported {
  width: 50%;
  margin: auto;
  background-color: #00ffaa;
  margin-top: 20px;
  padding: 10px;
}
</style>
