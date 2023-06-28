<template lang="">
  <div>
    <div>Tensorflow</div>
    <input type="file" id="inputFile" accept="image/*" @change="handleFileChange"/>
    <img src="" alt="" id="imgInvisible" ref="imgInvisible">
  </div>
</template>
<script>
import * as cocoSsd from '@tensorflow-models/coco-ssd';
import '@tensorflow/tfjs-backend-cpu';
import '@tensorflow/tfjs-backend-webgl';
export default {
  created() {
    this.initModel();
  },
  methods: {
    async initModel() {
      try {
        this.cocoModel = await cocoSsd.load();
        console.log(this.cocoModel);
      } catch (error) {
        console.log(error);
      }
    },
    handleFileChange(e) {
      if (e.target.files < 1) return;
      const reader = new FileReader();
      const _this = this;
      const image = document.getElementById('imgInvisible');
      reader.onload = function () {
        _this.$refs['imgInvisible'].src = reader.result;
        const imageTag = new window.Image();

        imageTag.onload = function () {
          image.setAttribute('width', `${imageTag.width}px`);
          image.setAttribute('height', `${imageTag.height}px`);
          _this.detectImage();
        }
        imageTag.src = reader.result;
      }
      reader.readAsDataURL(e.target.files[0]);
    },
    async detectImage() {
      const image = document.getElementById('imgInvisible');
      const result = await this.cocoModel.detect(image);
      console.log(result);
    }
  },
};
</script>
