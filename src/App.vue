<template>
  <div id="app"></div>
</template>

<script>
import * as THREE from "three/build/three.module.js";
import { FlyControls } from "three/examples/jsm/controls/FlyControls.js";

export default {
  name: "App",
  mounted() {
    const OrbitControls = require("three-orbit-controls")(THREE);
    const segmentHeight = 8;
    const segmentCount = 4;
    const height = segmentHeight * segmentCount;
    const halfHeight = height * 0.5;

    const sizing = {
      segmentHeight: segmentHeight,
      segmentCount: segmentCount,
      height: height,
      halfHeight
    };

    const scene = new THREE.Scene();
    const camera = new THREE.PerspectiveCamera(
      75,
      window.innerWidth / window.innerHeight,
      0.1,
      1000
    );

    // 3. Добавить источник пассивного звучания Audio.
    const listener = new THREE.AudioListener();
    camera.add(listener);

    const sound = new THREE.Audio(listener);
    const audioLoader = new THREE.AudioLoader();
    audioLoader.load(require("../public/audio/music.ogg"), function (buffer) {
      sound.setBuffer(buffer);
      sound.setLoop(true);
      sound.setVolume(0.5);
      sound.play();
    });

    // 5. Добавить элемент точечного звучания PositionalAudio, прикрепив его к одному из созданных элементов.
    const sound2 = new THREE.PositionalAudio(listener);
    const audioLoader2 = new THREE.AudioLoader();
    audioLoader2.load(require("../public/audio/music.wav"), function (buffer) {
      sound2.setBuffer(buffer);
      sound2.setLoop(true);
      sound2.setRefDistance(20);
      sound2.play();
    });

    const renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    document.body.appendChild(renderer.domElement);

    let controls2 = new FlyControls(camera, renderer.domElement);

    controls2.movementSpeed = 1000;
    controls2.domElement = renderer.domElement;
    controls2.rollSpeed = Math.PI / 24;
    controls2.autoForward = false;
    controls2.dragToLook = false;

    camera.position.z = 30;

    // 6. Добавить возможность перемещения по сцены и возможность "смотреть по сторонам".
    const controls = new OrbitControls(camera, renderer.domElement);
    controls.update();

    // 2. Добавить Skybox к сцене
    const loader = new THREE.CubeTextureLoader();
    const texture = loader.load([
      require("./assets/arid_ft.jpg"),
      require("./assets/arid_bk.jpg"),
      require("./assets/arid_up.jpg"),
      require("./assets/arid_dn.jpg"),
      require("./assets/arid_rt.jpg"),
      require("./assets/arid_lf.jpg")
    ]);
    scene.background = texture;

    const light = new THREE.HemisphereLight(0xffffbb, 0x080820, 1);
    // 1. Добавить тени к элементам из ЛР4, использовав атрибут CastShadow: true;
    light.castShadow = true;
    scene.add(light);

    const light2 = new THREE.PointLight(0xff0000, 5, 120);
    light2.position.set(50, 20, 0);
    scene.add(light2);

    const geometry2 = new THREE.TorusKnotGeometry(10, 3, 100, 16);
    const material2 = new THREE.MeshBasicMaterial({
      color: 0xffff00,
      wireframe: true
    });
    const torusKnot = new THREE.Mesh(geometry2, material2);
    torusKnot.position.set(50, 0, 0);
    scene.add(torusKnot);

    const geometry3 = new THREE.IcosahedronGeometry(10, 3, 100, 16);
    const material3 = new THREE.MeshPhysicalMaterial({
      color: 0x1f2236
    });
    const icosahedron = new THREE.Mesh(geometry3, material3);
    icosahedron.position.set(0, 0, 50);
    scene.add(icosahedron);

    const points = [];
    for (let i = 0; i < 10; i++) {
      points.push(new THREE.Vector2(Math.sin(i * 0.2) * 10 + 5, (i - 5) * 2));
    }
    const geometry4 = new THREE.LatheGeometry(points);
    const material4 = new THREE.MeshNormalMaterial({
      color: 0xffff00,
      side: THREE.DoubleSide
    });
    const lathe = new THREE.Mesh(geometry4, material4);
    lathe.position.set(50, 0, 50);
    scene.add(lathe);

    var bones = [];
    var prevBone = new THREE.Bone();
    prevBone.position.y = -sizing.halfHeight;

    for (let i = 0; i <= sizing.segmentCount; i++) {
      const bone = new THREE.Bone();
      bone.position.y = sizing.segmentHeight;
      bones.push(bone);
      prevBone.add(bone);
      prevBone = bone;
    }

    const geometry = new THREE.ConeGeometry(
      5,
      sizing.height,
      8,
      sizing.segmentCount * 3,
      true
    );

    const position = geometry.attributes.position;

    const vertex = new THREE.Vector3();

    const skinIndices = [];
    const skinWeights = [];

    for (let i = 0; i < position.count; i++) {
      vertex.fromBufferAttribute(position, i);

      const y = vertex.y + sizing.halfHeight;

      const skinIndex = Math.floor(y / sizing.segmentHeight);
      const skinWeight = (y % sizing.segmentHeight) / sizing.segmentHeight;

      skinIndices.push(skinIndex, skinIndex + 1, 0, 0);
      skinWeights.push(1 - skinWeight, skinWeight, 0, 0);
    }

    geometry.setAttribute(
      "skinIndex",
      new THREE.Uint16BufferAttribute(skinIndices, 4)
    );
    geometry.setAttribute(
      "skinWeight",
      new THREE.Float32BufferAttribute(skinWeights, 4)
    );

    const material = new THREE.MeshStandardMaterial({
      skinning: true,
      side: THREE.DoubleSide
    });
    const mesh = new THREE.SkinnedMesh(geometry, material);

    const skeleton = new THREE.Skeleton(bones);

    bones[0].position.y = -sizing.halfHeight;

    mesh.add(bones[0]);

    mesh.bind(skeleton);

    let helper = new THREE.SkeletonHelper(mesh);
    helper.material.linewidth = 2;
    scene.add(helper);

    mesh.scale.multiplyScalar(1);
    scene.add(mesh);
    mesh.add(sound2);

    let forward = true;
    let stepSize = 0.01;
    const animate = function () {
      requestAnimationFrame(animate);

      if (forward) {
        bones[1].rotation.z += stepSize;
        bones[2].rotation.z += stepSize;
        bones[3].rotation.z += stepSize / 2;

        icosahedron.scale.x += stepSize;
        icosahedron.scale.y += stepSize;
        icosahedron.scale.z += stepSize;
        if (bones[2].rotation.z >= 1) {
          forward = false;
        }
      } else {
        bones[1].rotation.z -= stepSize;
        bones[2].rotation.z -= stepSize;
        bones[3].rotation.z -= stepSize / 2;

        icosahedron.scale.x -= stepSize;
        icosahedron.scale.y -= stepSize;
        icosahedron.scale.z -= stepSize;
        if (bones[2].rotation.z <= -1) {
          forward = true;
        }
      }

      // camera.rotation.y += 0.01;

      torusKnot.rotation.y += stepSize;
      renderer.render(scene, camera);
    };

    animate();

    window.addEventListener(
      "resize",
      function () {
        camera.aspect = window.innerWidth / window.innerHeight;
        camera.updateProjectionMatrix();

        renderer.setSize(window.innerWidth, window.innerHeight);
      },
      false
    );
  }
};
</script>

<style lang="scss">
body {
  margin: 0;
  padding: 0;
}
</style>
