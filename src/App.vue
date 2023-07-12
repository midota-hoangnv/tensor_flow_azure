<template lang="">
  <div>
    <div>Tensorflow</div>
    <input
      type="file"
      id="inputFile"
      accept="image/*"
      @change="handleFileChange"
    />
    <span id="imageOverlay" class="imageOverlay"><img src="" alt="" id="imgInvisible" ref="imgInvisible" /></span>
  </div>
</template>
<script>
import "@tensorflow/tfjs-backend-cpu";
import "@tensorflow/tfjs-backend-webgl";
import * as tf from "@tensorflow/tfjs";
const ANCHORS = [0.573, 0.677, 1.87, 2.06, 3.34, 5.47, 7.88, 3.53, 9.77, 9.17];
const NEW_OD_OUTPUT_TENSORS = [
  "detected_boxes",
  "detected_scores",
  "detected_classes",
];
const TARGET_CLASSES = {
  0: "Rectangle",
};
export default {
  data() {
    return {
      model: null,
      isNewOdModel: null,
    };
  },
  created() {
    this.loadModel();
  },
  methods: {
    _logistic(x) {
      if (x > 0) {
        return 1 / (1 + Math.exp(-x));
      } else {
        const e = Math.exp(x);
        return e / (1 + e);
      }
    },
    async loadModel() {
      console.log("Loading model...");
      this.model = await tf.loadGraphModel("/model/model.json");
      console.log("🚀 ~ file: App.vue:48 ~ loadModel ~ this.model:", this.model)
      this.isNewOdModel = this.model.inputs.length == 3;
      console.log(
      "🚀 ~ file: App.vue:40 ~ loadModel ~ this.isNewOdModel:",
      this.isNewOdModel
      );
      console.log("Model loaded.");
    },
    handleFileChange(e) {
      if (e.target.files < 1) return;
      const reader = new FileReader();
      const _this = this;
      const image = document.getElementById("imgInvisible");
      reader.onload = function () {
        _this.$refs["imgInvisible"].src = reader.result;
        const imageTag = new window.Image();

        imageTag.onload = function () {
          image.setAttribute("width", `${imageTag.width}px`);
          image.setAttribute("height", `${imageTag.height}px`);
          _this.detectImage();
        };
        imageTag.src = reader.result;
      };
      reader.readAsDataURL(e.target.files[0]);
    },
    async loadImage() {
      console.log("Pre-processing image...");
      const image = document.getElementById("imgInvisible");

      // Pre-process the image
      const inputSize = this.model.inputs[0].shape[1];
      tf.zeros([image.width, image.height]).print();
      console.log("🚀 ~ file: App.vue:66 ~ loadImage ~ inputSize:", inputSize);
      let imagePixels = tf.browser.fromPixels(image, 3);
      console.log("🚀 ~ file: App.vue:82 ~ loadImage ~ imagePixels:", imagePixels)
      imagePixels = tf.image.resizeBilinear(
        imagePixels.expandDims().toFloat(),
        [inputSize, inputSize]
      );
      console.log("🚀 ~ file: App.vue:87 ~ loadImage ~ imagePixels:", imagePixels)
      if (this.isNewOdModel) {
        console.log("Object Detection Model V2 detected.");
        imagePixels = this.isNewOdModel ? imagePixels : image.reverse(-1); // RGB->BGR for old models
      }

      return imagePixels;
    },
    async predictRectangle(inputs) {
      console.log("Running predictions...");
      const outputs = await this.model.executeAsync(
        inputs,
        this.isNewOdModel ? NEW_OD_OUTPUT_TENSORS : null
      );

      const arrays = !Array.isArray(outputs)
        ? outputs.array()
        : Promise.all(outputs.map((t) => t.array()));
      let predictions = await arrays;

      console.log("🚀 ~ file: App.vue:102 ~ predictRectangle ~ predictions:", predictions)


      // Post processing for old models.
      if (predictions.length != 3) {
        console.log("Post processing...");
        const num_anchor = ANCHORS.length / 2;
        const channels = predictions[0][0][0].length;
        const height = predictions[0].length;
        const width = predictions[0][0].length;

        const num_class = channels / num_anchor - 5;

        let boxes = [];
        let scores = [];
        let classes = [];

        for (var grid_y = 0; grid_y < height; grid_y++) {
          for (var grid_x = 0; grid_x < width; grid_x++) {
            let offset = 0;

            for (var i = 0; i < num_anchor; i++) {
              let x =
                (this._logistic(predictions[0][grid_y][grid_x][offset++]) +
                  grid_x) /
                width;
              let y =
                (this._logistic(predictions[0][grid_y][grid_x][offset++]) +
                  grid_y) /
                height;
              let w =
                (Math.exp(predictions[0][grid_y][grid_x][offset++]) *
                  ANCHORS[i * 2]) /
                width;
              let h =
                (Math.exp(predictions[0][grid_y][grid_x][offset++]) *
                  ANCHORS[i * 2 + 1]) /
                height;

              let objectness = tf.scalar(
                this._logistic(predictions[0][grid_y][grid_x][offset++])
              );
              let class_probabilities = tf
                .tensor1d(
                  predictions[0][grid_y][grid_x].slice(
                    offset,
                    offset + num_class
                  )
                )
                .softmax();
              offset += num_class;

              class_probabilities = class_probabilities.mul(objectness);
              let max_index = class_probabilities.argMax();
              boxes.push([x - w / 2, y - h / 2, x + w / 2, y + h / 2]);
              scores.push(class_probabilities.max().dataSync()[0]);
              classes.push(max_index.dataSync()[0]);
            }
          }
        }

        boxes = tf.tensor2d(boxes);
        scores = tf.tensor1d(scores);
        classes = tf.tensor1d(classes);

        const selected_indices = await tf.image.nonMaxSuppressionAsync(
          boxes,
          scores,
          10
        );
        predictions = [
          await boxes.gather(selected_indices).array(),
          await scores.gather(selected_indices).array(),
          await classes.gather(selected_indices).array(),
        ];
      }

      return predictions;
    },

    async highlightResults(predictions) {
      console.log(predictions);
      console.log("Highlighting results...");
      const selectedImage = document.getElementById("imgInvisible");
      const imageOverlay = document.getElementById("imageOverlay");

      for (let n = 0; n < predictions[0].length; n++) {
        // Check scores
        console.log("🚀 ~ file: App.vue:187 ~ highlightResults ~ predictions[1][n] :", predictions[1][n] )
        if (predictions[1][n] > 0.3) {

          const p = document.createElement("p");
          p.innerText =
            TARGET_CLASSES[predictions[2][n]] +
            ": " +
            Math.round(parseFloat(predictions[1][n]) * 100) +
            "%";

          let bboxLeft = predictions[0][n][0] * selectedImage.width + 10;
          let bboxTop = predictions[0][n][1] * selectedImage.height - 10;
          let bboxWidth =
            predictions[0][n][2] * selectedImage.width - bboxLeft + 20;
          let bboxHeight =
            predictions[0][n][3] * selectedImage.height - bboxTop + 10;

          p.style =
            "margin-left: " +
            bboxLeft +
            "px; margin-top: " +
            (bboxTop - 10) +
            "px; width: " +
            bboxWidth +
            "px; top: 0; left: 0;";
          const highlighter = document.createElement("div");
          highlighter.setAttribute("class", "highlighter");
          highlighter.style =
            "left: " +
            bboxLeft +
            "px; top: " +
            bboxTop +
            "px; width: " +
            bboxWidth +
            "px; height: " +
            bboxHeight +
            "px;";
          imageOverlay.appendChild(highlighter);
          imageOverlay.appendChild(p);
          // children.push(highlighter);
          // children.push(p);
          // document.body.append(p)
        }
      }
    },
    async detectImage() {
      if (!this.model) return;
      const image = await this.loadImage();
      console.log("before detection");
      // let predictions = await this.model.predict(image).data();
      let predictions = await this.predictRectangle(image);
      console.log((predictions));
      this.highlightResults(predictions);
      console.log(predictions);
    },
  },
};
</script>


<style>
.removed {
    display: none;
}

.invisible {
    opacity: 0.2;
}

.imageOverlay {
    position: relative;
    cursor: pointer;
    display: inline-block;
}

.imageOverlay p {
    position: absolute;
    padding: 5px;
    background-color: rgba(255, 111, 0, 0.85);
    color: #FFF;
    border: 1px dashed rgba(255, 255, 255, 0.7);
    z-index: 2;
    font-size: 10px;
}

.highlighter {
    background: rgba(0, 255, 0, 0.25);
    border: 1px dashed #fff;
    z-index: 1;
    position: absolute;
}
</style>
