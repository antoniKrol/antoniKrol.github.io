<template>
  <VCol>
    <VRow class="mx-2 mt-4">
      <VCard class="me-1 pa-1" color="primary">
        <div ref="rendererContainer" />
      </VCard>
      <VCard color="primary" class="pa-1">
        <div class="content-container">
          <video class="input_video" ref="videoElement" playsinline></video>
          <canvas id="drawing" ref="canvas"></canvas>
        </div>
      </VCard>
    </VRow>

    <VContainer fluid class="d-flex justify-center align-center">
      <VIcon class="mt-5" size="36" :icon="currentIcon" :color="currentColor" />
      <br />
      <p id="coord" ref="coordElement"></p>
    </VContainer>
  </VCol>
</template>

<script>
import { Point } from "../../lib/point";
import * as THREE from "three";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls";
import { Camera } from "@mediapipe/camera_utils";
import { Hands } from "@mediapipe/hands";
import { ConvexHull } from "three-stdlib";
import {
  arePointsTouching,
  mergeClosePoints,
} from "@/utils/utils";
export default {
  name: "HandRecognition",
  data() {
    return {
      currentIcon: "mdi-thumb-down",
      currentColor: "red",
    };
  },
  mounted() {
    const width = 700;
    const height = 400;
    let isCameraLooking = false;
    const addShape = (camera) => {
      const scale = 100; // Increase the scale if needed
      const points = pointsArray.map(
        (point) =>
          new THREE.Vector3(
            point.x * scale,
            point.y * scale,
            Math.abs(point.z * scale * 10)
          )
      );
      const convexHull = new ConvexHull().setFromPoints(points);

      // Create geometry from the convex hull
      const geometry = new THREE.BufferGeometry();
      const vertices = [];
      const faces = convexHull.faces;

      faces.forEach((face) => {
        const a = face.edge.next.vertex.point;
        const b = face.edge.prev.vertex.point;
        const c = face.edge.vertex.point;
        vertices.push(a.x, a.y, a.z, b.x, b.y, b.z, c.x, c.y, c.z);
      });

      geometry.setAttribute(
        "position",
        new THREE.BufferAttribute(new Float32Array(vertices), 3)
      );

      const material = new THREE.MeshBasicMaterial({
        color: 0x00ffee,
        side: THREE.DoubleSide,
      });

      mesh = new THREE.Mesh(geometry, material);
      if (!isCameraLooking) {
        // Calculate the center of the input points
        const center = new THREE.Vector3(0, 0, 0);
        pointsArray.forEach((point) => center.add(point));
        center.divideScalar(pointsArray.length);

        // Update camera position and target
        camera.position.set(center.x, center.y, center.z + 100);
        camera.lookAt(center);
        isCameraLooking = true;
      }
      scene.add(mesh);
    };

    let shapeAdded = false;
    // Pobierz element <video> o klasie "input_video"
    const videoElement = this.$refs.videoElement;

    // Tablica do przechowywania punktów
    let pointsArray = [];

    // Zmienna do przechowywania aktualnego indeksu w tablicy temp
    //let i = -1;
    let isPointAdded = false;



    //Funkcja wywoływana przy każdym nowym wyniku rozpoznawania dłoni
    //d = ((x2 - x1)2 + (y2 - y1)2 + (z2 - z1)2)1/2
    const onResults = (results) => {
      // Jeśli została znaleziona co najmniej jedna dłoń
      if (results.multiHandLandmarks.length > 0) {
        // Stwórz nowe obiekty Point dla dwóch punktów: palca wskazującego i kciuka
        let p1 = new Point(
          results.multiHandLandmarks[0][8].x,
          results.multiHandLandmarks[0][8].y,
          results.multiHandLandmarks[0][8].z
        );
        let p2 = new Point(
          results.multiHandLandmarks[0][4].x,
          results.multiHandLandmarks[0][4].y,
          results.multiHandLandmarks[0][4].z
        );
        if (pointsArray.length >= 3 && !shapeAdded) {
          addShape(cameraT);
          //shapeAdded=true;
        }
        if (!isPointAdded && arePointsTouching(p1, p2)) {
          pointsArray = mergeClosePoints(pointsArray, p1, 0.02);
          isPointAdded = true;
        } else if (!arePointsTouching(p1, p2)) {
          this.$refs.coordElement.innerHTML = pointsArray.length;
          this.currentIcon = "mdi-thumb-up";
          this.currentColor = "green";
          isPointAdded = false;
          if (pointsArray.length >= 3) {
            //draw(pointsArray);
          }
        } else {
          this.$refs.coordElement.innerHTML =
            "Geting coordinates from your hands";
        }
      } else {
        this.currentIcon = "mdi-thumb-down";
        this.currentColor = "red";
      }
    };
    // Stwórz nowy obiekt Hands z użyciem biblioteki @mediapipe/hands
    const hands = new Hands({
      locateFile: (file) => {
        return `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}`;
      },
    });

    // Ustaw opcje dla obiektu Hands
    hands.setOptions({
      maxNumHands: 1,
      modelComplexity: 1,
      minDetectionConfidence: 0.7,
      minTrackingConfidence: 0.7,
    });

    hands.onResults(onResults);

    const camera = new Camera(videoElement, {
      onFrame: async () => {
        await hands.send({ image: videoElement });
      },
      width: width,
      height: height,
    });
    camera.start();

    // Załaduj bibliotekę Three.js
    var scene, cameraT, renderer, mesh;
    var controls;
    const canvas = this.$refs.canvas;
    canvas.width = width;
    canvas.height = height;
    // Utwórz scenę
    scene = new THREE.Scene();

    // Utwórz kamerę
    cameraT = new THREE.PerspectiveCamera(
      775,
      canvas.width / canvas.height,
      0.1,
      1000
    );
    cameraT.position.z = 5;

    // Utwórz renderer
    renderer = new THREE.WebGLRenderer({ canvas: canvas });
    renderer.setSize(canvas.width, canvas.height);
    this.$refs.rendererContainer.appendChild(renderer.domElement);

    // Utwórz kontrolery orbity
    controls = new OrbitControls(cameraT, renderer.domElement);

    // Funkcja aktualizująca
    var render = function () {
      requestAnimationFrame(render);

      controls.update();
      renderer.render(scene, cameraT);
    };

    // Rozpocznij renderowanie
    render();
  },
};
</script>

<style scoped>
.input_video {
  transform: scaleX(-1);
  /* This will flip the video horizontally */
}
</style>
