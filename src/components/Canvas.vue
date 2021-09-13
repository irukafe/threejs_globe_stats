<template>
    <div ref="container" id="canvas">
    </div>
</template>

<script>
import * as THREE from 'three';
import { TrackballControls } from 'three/examples/jsm/controls/TrackballControls.js';
import { GUI } from 'three/examples/jsm/libs/dat.gui.module.js';
import ThreeGlobe from 'three-globe';

export default {
  name: 'Canvas',
  data () {
    return {
      camera: null,
      scene: null,
      renderer: null,
      controls: null
    };
  },
  created () {
  },
  destroyed () {
    window.removeEventListener('resize', this.onWindowResize);
  },
  mounted () {
    this.init();
    window.addEventListener('resize', this.onWindowResize);
  },
  methods: {
    init: function () {
      fetch('./json/ne_110m_admin_0_countries_geom.geojson').then(res => res.json()).then(countries => {
        // Function to sort numbers.
        const sortNumbers = data => data.sort((a, b) => a - b);

        // Function to get quantile.
        const quartile = function (data, q) {
          data = sortNumbers(data);

          const pos = ((data.length) - 1) * q;
          const base = Math.floor(pos);
          const rest = pos - base;
          if ((data[base + 1] !== undefined)) {
            return data[base] + rest * (data[base + 1] - data[base]);
          } else {
            return data[base];
          }
        }

        // Function to get cap color.
        const capColor = function (value, quartiles) {
          if (value < 0) return '#333';

          var green = 250;
          quartiles.forEach(function (q) {
            if (value < q) return `rgba(200, ${green}, 0, 0.8)`
            green -= 50;
          })
          return `rgba(200, ${green}, 0, 0.8)`
        }

        // Function to change cap colors.
        const changeCapColor = function () {
          const numArray = countries.features.map(d => d.properties[params.item])
          const quartiles = [0.2, 0.4, 0.6, 0.8].map(q => quartile(numArray, q));
          Globe.polygonCapColor(d => capColor(d.properties[params.item], quartiles));
        }

        const initialItem = 'POP_EST';

        const numArray = countries.features.map(d => d.properties[initialItem])
        const quartiles = [0.2, 0.4, 0.6, 0.8].map(q => quartile(numArray, q));

        // Creates globe object.
        const Globe = new ThreeGlobe()
          .globeImageUrl('./image/NE1_50M_SR_W.jpg')
          .polygonsData(countries.features)
          .polygonAltitude(0.01)
          .polygonStrokeColor(() => 'rgba(200, 200, 200, 0.8)')
          .polygonSideColor(() => 'rgba(0, 200, 0, 0.7)')
          .polygonCapColor(d => capColor(d.properties[initialItem], quartiles));

        // Creates renderer.
        this.renderer = new THREE.WebGLRenderer();
        this.renderer.setSize(window.innerWidth, window.innerHeight);
        this.$refs.container.appendChild(this.renderer.domElement);

        // Creates scene.
        this.scene = new THREE.Scene();
        this.scene.add(Globe);
        this.scene.add(new THREE.AmbientLight(0xbbbbbb));
        this.scene.add(new THREE.DirectionalLight(0xffffff, 0.6));

        // Creates camera.
        this.camera = new THREE.PerspectiveCamera();
        this.camera.aspect = window.innerWidth / window.innerHeight;
        this.camera.updateProjectionMatrix();
        this.camera.position.z = 300;

        // Creates item selecter.
        const params = {
          item: initialItem
        }
        const values = Object.keys(countries.features[0].properties).filter(k => k !== 'SOV_A3')
        const gui = new GUI({ width: 'max-content' });
        gui.domElement.id = 'gui';
        gui.add(params, 'item', values).onChange(function () {
          changeCapColor();
        });

        // Adds camera controls.
        this.controls = new TrackballControls(this.camera, this.renderer.domElement);
        this.controls.minDistance = 101;
        this.controls.rotateSpeed = 2;
        this.controls.zoomSpeed = 0.5;

        this.renderer.setAnimationLoop(this.animation);
      });
    },
    animation: function () {
      this.controls.update();
      this.renderer.render(this.scene, this.camera);
    },
    onWindowResize: function () {
      this.camera.aspect = window.innerHeight / window.innerHeight;
      this.camera.updateProjectionMatrix();
      this.renderer.setPixelRatio(window.devicePixelRatio);
      this.renderer.setSize(window.innerHeight, window.innerHeight);
    }
  }
}
</script>

<style>
body {
  margin: 0;
  background-color: #000;
}

#canvas {
  margin: 0 auto;
  width: max-content;
}

.dg .property-name {
  width: 40px;
}
</style>
