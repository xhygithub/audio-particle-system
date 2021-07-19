<template>
  <div class="audio-particles">
    <audio class="audio-particles__audio" src="../assets/test.mp3" ref="audio" loop />
    <div class="particles-canvas" ref="canvasContainer"></div>
  </div>
</template>

<script>
import * as THREE from 'three';

export default {
  name: "AudioParticles",
  data: function () {
    return {
      audioBufferLength: 128,
      SEPARATION: 20,
      AMOUNTX: 170,
      AMOUNTY: 30,
      canvasHeight: 600,
      defaultWaveHeight: 80,
      PI2: Math.PI * 2,
      movingSpeed:  0,
      fov: 120,
      particleColor: 0x99999,
      firstRendering: true,
      renderer: new THREE.WebGLRenderer(),
      scene: new THREE.Scene(),
    }
  },
  mounted: function () {
    var AudioContext =
        window.AudioContext || // Default
        window.webkitAudioContext || // Safari and old versions of Chrome
        false;
    if (AudioContext) {
      this.audioCtx = new AudioContext();
      this.analyser = this.audioCtx.createAnalyser(); //频域和时域分析器
      this.analyser.fftSize = this.audioBufferLength * 2; //用于计算频域信号时使用的 FFT
      this.$nextTick(() => {
        this.initCanvas();
        this.visualize();
        this.listenVideoPlay();
      });
    }
  },
  computed: {

  },
  methods: {
    listenVideoPlay: function () {
      const myAudio = this.$refs.audio;
      myAudio &&
      myAudio.addEventListener('play', () => {
        this.audioCtx.resume();
        if (!this.source) {
          this.source = this.audioCtx.createMediaElementSource(myAudio);
        }
        let gainNode = this.audioCtx.createGain();
        this.source.connect(this.analyser);
        this.analyser.connect(gainNode);
        gainNode.connect(this.audioCtx.destination);
      });
    },

    visualize: function () {
      let bufferLength = this.analyser.frequencyBinCount; //half of the fftSize： 128
      let frequencyArr = new Uint8Array(bufferLength); // // 缓冲区:进行数据的缓冲处理，转换成二进制数据
      //告诉浏览器——你希望执行一个动画，并且要求浏览器在下次重绘之前调用指定的回调函数更新动画
      let requestAnimationFrame = window.requestAnimationFrame || window.webkitRequestAnimationFrame;
      let animate = this.animation;
      let analyser = this.analyser;

      function v() {
        analyser.getByteFrequencyData(frequencyArr); //频域数据拷贝进数组里
        animate(frequencyArr);
        requestAnimationFrame(v);
      }

      requestAnimationFrame(v);
    },

    initCanvas: function () {
      this.calculateCanvasParams();
      this.camera = new THREE.PerspectiveCamera(
          this.fov,
          this.aspect, // based on chart width/height (window.innerWidth + window.innerWidth * 0.2) / this.canvasHeight,
          1,
          10000
      ); // 视野角度（FOV），长宽比（aspect ratio），近截面（near）和远截面（far
      this.camera.position.set(0, 700, 200); // set camera position: x, y, z
      this.camera.rotation.x = -0.5;

      const numPoints = this.AMOUNTX * this.AMOUNTY;
      const positions = new Float32Array(numPoints * 3);
      let k = 0;
      for (let ix = 0; ix < this.AMOUNTX; ix++) {
        for (let iy = 0; iy < this.AMOUNTY; iy++) {
          let x = ix * this.SEPARATION - (this.AMOUNTX * this.SEPARATION) / 2;
          let z = iy * this.SEPARATION - (this.AMOUNTY * this.SEPARATION - 10);
          let y = 0;
          positions[3 * k] = x;
          positions[3 * k + 1] = y;
          positions[3 * k + 2] = z;
          k++;
        }
      }
      this.geometry = new THREE.BufferGeometry();
      this.geometry.setAttribute('position', new THREE.Float32BufferAttribute(positions, 3));
      const material = new THREE.PointsMaterial({
        size: 1,
        color: this.particleColor,
        sizeAttenuation: false,
      });

      let points = new THREE.Points(this.geometry, material);
      this.scene.add(points);
      this.renderer.setSize(window.innerWidth + window.innerWidth * 0.2, this.canvasHeight);
      this.renderer.setClearColor(0x000000, 1);
      const container = this.$refs.canvasContainer;
      container && container.appendChild(this.renderer.domElement);
      window.addEventListener('resize', this.onWindowResize, false);
    },

    calculateCanvasParams: function () {
      this.fov = 100;
      let ratio = window.innerWidth / this.canvasHeight;
      if (window.innerWidth / window.innerHeight < 1) {
        this.aspect = ratio;
        this.fov = 140;
      }
      else {
        let max = 3;
        let min = 2;
        if (ratio > max) {
          this.aspect = max;
        } else if (ratio < min) {
          this.aspect = min;
        } else {
          this.aspect = ratio;
        }
      }

      if (window.innerWidth < 768) {
        this.canvasHeight = window.innerHeight - 368 + 124; // image height + padding top
        this.particleColor = 0x666666;
      } else {
        this.canvasHeight = window.innerHeight - 12.4 * window.innerWidth / 100 - 259 + 40;
        this.particleColor = 0x666666;
      }
    },
    onWindowResize: function () {
      this.calculateCanvasParams();
      this.camera.aspect = this.aspect; //(window.innerWidth + window.innerWidth * 0.2) / this.canvasHeight;
      this.camera.fov = this.fov;
      this.camera.updateProjectionMatrix();
      this.renderer.setSize(window.innerWidth + window.innerWidth * 0.2, this.canvasHeight);
    },

    animation: function (arr) {
      let data = arr || [];
      let sum = data.reduce((a, b) => a + b, 0);
      let waveH, movingSpeedIncreasing;
      if (sum > 0) {
        movingSpeedIncreasing = 0.8;
        waveH = sum / 20 < this.defaultWaveHeight ? this.defaultWaveHeight : sum / 20;
      } else {
        waveH = this.defaultWaveHeight;
        movingSpeedIncreasing = 0.1;
      }
      let i = 0;
      let waveCount = 0.3;
      const positions = this.geometry.attributes.position;
      for (let ix = 0; ix < this.AMOUNTX; ix++) {
        for (let iy = 0; iy < this.AMOUNTY; iy++) {
          if (this.firstRendering) {
            positions.setY(
                i,
                80 * Math.cos(0.2 * iy - this.movingSpeed * 0.5) +
                80 * Math.cos(0.1 * ix - this.movingSpeed * 0.5 - 10)
            );
          } else {
            positions.setY(
                i,
                waveH * Math.random() * Math.cos(waveCount * iy - this.movingSpeed * 0.5) +
                waveH *
                Math.random() *
                Math.cos(0.1 * ix - this.movingSpeed * 0.6 - 200)
            );
          }
          i++;
        }
      }
      positions.needsUpdate = true;
      this.renderer.render(this.scene, this.camera);
      this.movingSpeed += movingSpeedIncreasing; // define the moving speed
    },

    audioPlay: function () {
      this.$refs.audio.load();
      this.$refs.audio.play();
      this.firstRendering = false;
    },

    audioStop: function () {
      this.$refs.audio.pause();
      this.$refs.audio.currentTime = 0;
      this.firstRendering = true;
    },

  }
}
</script>

<style scoped>
.audio-particles__audio {
  visibility: hidden;
}
</style>