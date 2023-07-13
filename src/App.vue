<template lang="">
  <div>
    <!-- <div>Tensorflow</div>
    <input
      type="file"
      id="inputFile"
      accept="image/*"
      @change="handleFileChange"
    />
    <span id="imageOverlay" class="imageOverlay"
      ><img src="" alt="" id="imgInvisible" ref="imgInvisible"
    /></span>
    <img :src="cutImage" alt="" /> -->
    <div>
      <h1>Real-Time Object Detection: Kangaroo</h1>
      <h3>MobileNetV2</h3>
      <video
        :style="{ height: '600px', width: '500px' }"
        class="size"
        autoPlay
        playsInline
        muted
        ref="videoRef"
        width="600"
        height="500"
        id="frame"
      />
      <canvas class="size" ref="canvasRef" width="600" height="500" />
    </div>
  </div>
</template>
<script>
import "@tensorflow/tfjs-backend-cpu";
import "@tensorflow/tfjs-backend-webgl";
import * as cvstfjs from "@microsoft/customvision-tfjs";
// import * as tf from "@tensorflow/tfjs";
// const ANCHORS = [0.573, 0.677, 1.87, 2.06, 3.34, 5.47, 7.88, 3.53, 9.77, 9.17];
// const NEW_OD_OUTPUT_TENSORS = [
//   "detected_boxes",
//   "detected_scores",
//   "detected_classes",
// ];
// const TARGET_CLASSES = {
//   0: "Rectangle",
// };
export default {
  data() {
    return {
      model: null,
      isNewOdModel: null,
      cutImage: null,
    };
  },
  mounted() {
    if (navigator.mediaDevices && navigator.mediaDevices.getUserMedia) {
      const webCamPromise = navigator.mediaDevices
        .getUserMedia({
          audio: false,
          video: {
            facingMode: "user",
          },
        })
        .then((stream) => {
          window.stream = stream;
          this.$refs.videoRef.srcObject = stream;
          return new Promise((resolve) => {
            this.$refs.videoRef.onloadedmetadata = () => {
              resolve();
            };
          });
        });

      Promise.all([webCamPromise])
        .then(() => {
          this.detectFrame(this.$refs.videoRef);
        })
        .catch((error) => {
          console.error(error);
        });
    }
  },
  methods: {
    // _logistic(x) {
    //   if (x > 0) {
    //     return 1 / (1 + Math.exp(-x));
    //   } else {
    //     const e = Math.exp(x);
    //     return e / (1 + e);
    //   }
    // },

    renderPredictions(predictions) {
      const ctx = this.$refs.canvasRef.getContext("2d");
      ctx.clearRect(0, 0, ctx.canvas.width, ctx.canvas.height);

      // Font options.
      const font = "16px sans-serif";
      ctx.font = font;
      ctx.textBaseline = "top";

      //Getting predictions
      const tagName = predictions[0].tagName;
      const point = Math.round(parseFloat(predictions[0].probability) * 100);

      let x = predictions[0].boundingBox.left * this.$refs.videoRef.width;
      let y = predictions[0].boundingBox.top * this.$refs.videoRef.height;
      let width = predictions[0].boundingBox.width * this.$refs.videoRef.width;
      let height =
        predictions[0].boundingBox.height * this.$refs.videoRef.height;

      // Draw the bounding box.
      ctx.strokeStyle = "#00FFFF";
      ctx.lineWidth = 4;
      ctx.strokeRect(x, y, width, height);

      // Draw the label background.
      ctx.fillStyle = "#00FFFF";
      const textWidth = ctx.measureText(tagName + " " + point + "%").width;
      const textHeight = parseInt(font, 10); // base 10
      ctx.fillRect(x, y, textWidth + 4, textHeight + 4);

      // Draw the text last to ensure it's on top.
      ctx.fillStyle = "#000000";
      ctx.fillText(tagName + " " + point + "%", x, y);
    },
    detectFrame(video) {
      let canvas = document.createElement("canvas");

      canvas.width = video.width;
      canvas.height = video.height;

      let ctx = canvas.getContext("2d");
      ctx.drawImage(video, 0, 0, canvas.width, canvas.height);

      canvas.toBlob((blob) => {
        console.log('blob', blob);
        this.detectWithAruze(blob).then((predictions) => {
          console.log(predictions);
          this.renderPredictions(predictions);
          requestAnimationFrame(() => {
            this.detectFrame(video);
          });
        });
      })
    },
    async loadModel() {
      console.log("Loading model...");
      this.model = new cvstfjs.ObjectDetectionModel();
      return await this.model.loadModelAsync("/model/model.json");
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
          // _this.detectImage();
        };
        imageTag.src = reader.result;
      };
      this.detectWithAruze(e.target.files[0]);
      reader.readAsDataURL(e.target.files[0]);
    },
    // async loadImage() {
    //   console.log("Pre-processing image...");
    //   const image = document.getElementById("imgInvisible");

    //   // Pre-process the image
    //   const inputSize = this.model.inputs[0].shape[1];
    //   tf.zeros([image.width, image.height]).print();
    //   console.log("🚀 ~ file: App.vue:66 ~ loadImage ~ inputSize:", inputSize);
    //   let imagePixels = tf.browser.fromPixels(image, 3);
    //   console.log("🚀 ~ file: App.vue:82 ~ loadImage ~ imagePixels:", imagePixels)
    //   imagePixels = tf.image.resizeBilinear(
    //     imagePixels.expandDims().toFloat(),
    //     [inputSize, inputSize]
    //   );
    //   console.log("🚀 ~ file: App.vue:87 ~ loadImage ~ imagePixels:", imagePixels)
    //   if (this.isNewOdModel) {
    //     console.log("Object Detection Model V2 detected.");
    //     imagePixels = this.isNewOdModel ? imagePixels : image.reverse(-1); // RGB->BGR for old models
    //   }

    //   return imagePixels;
    // },
    // async predictRectangle(inputs) {
    //   console.log("Running predictions...");
    //   const outputs = await this.model.executeAsync(
    //     inputs,
    //     this.isNewOdModel ? NEW_OD_OUTPUT_TENSORS : null
    //   );

    //   const arrays = !Array.isArray(outputs)
    //     ? outputs.array()
    //     : Promise.all(outputs.map((t) => t.array()));
    //   let predictions = await arrays;

    //   console.log("🚀 ~ file: App.vue:102 ~ predictRectangle ~ predictions:", predictions)

    //   // Post processing for old models.
    //   if (predictions.length != 3) {
    //     console.log("Post processing...");
    //     const num_anchor = ANCHORS.length / 2;
    //     const channels = predictions[0][0][0].length;
    //     const height = predictions[0].length;
    //     const width = predictions[0][0].length;

    //     const num_class = channels / num_anchor - 5;

    //     let boxes = [];
    //     let scores = [];
    //     let classes = [];

    //     for (var grid_y = 0; grid_y < height; grid_y++) {
    //       for (var grid_x = 0; grid_x < width; grid_x++) {
    //         let offset = 0;

    //         for (var i = 0; i < num_anchor; i++) {
    //           let x =
    //             (this._logistic(predictions[0][grid_y][grid_x][offset++]) +
    //               grid_x) /
    //             width;
    //           let y =
    //             (this._logistic(predictions[0][grid_y][grid_x][offset++]) +
    //               grid_y) /
    //             height;
    //           let w =
    //             (Math.exp(predictions[0][grid_y][grid_x][offset++]) *
    //               ANCHORS[i * 2]) /
    //             width;
    //           let h =
    //             (Math.exp(predictions[0][grid_y][grid_x][offset++]) *
    //               ANCHORS[i * 2 + 1]) /
    //             height;

    //           let objectness = tf.scalar(
    //             this._logistic(predictions[0][grid_y][grid_x][offset++])
    //           );
    //           let class_probabilities = tf
    //             .tensor1d(
    //               predictions[0][grid_y][grid_x].slice(
    //                 offset,
    //                 offset + num_class
    //               )
    //             )
    //             .softmax();
    //           offset += num_class;

    //           class_probabilities = class_probabilities.mul(objectness);
    //           let max_index = class_probabilities.argMax();
    //           boxes.push([x - w / 2, y - h / 2, x + w / 2, y + h / 2]);
    //           scores.push(class_probabilities.max().dataSync()[0]);
    //           classes.push(max_index.dataSync()[0]);
    //         }
    //       }
    //     }

    //     boxes = tf.tensor2d(boxes);
    //     scores = tf.tensor1d(scores);
    //     classes = tf.tensor1d(classes);

    //     const selected_indices = await tf.image.nonMaxSuppressionAsync(
    //       boxes,
    //       scores,
    //       10
    //     );
    //     predictions = [
    //       await boxes.gather(selected_indices).array(),
    //       await scores.gather(selected_indices).array(),
    //       await classes.gather(selected_indices).array(),
    //     ];
    //   }

    //   return predictions;
    // },

    // async highlightResults(predictions) {
    //   console.log(predictions);
    //   console.log("Highlighting results...");
    //   const selectedImage = document.getElementById("imgInvisible");
    //   const imageOverlay = document.getElementById("imageOverlay");

    //   for (let n = 0; n < predictions[0].length; n++) {
    //     // Check scores
    //     console.log("🚀 ~ file: App.vue:187 ~ highlightResults ~ predictions[1][n] :", predictions[1][n] )
    //     if (predictions[1][n] > 0.3) {

    //       const p = document.createElement("p");
    //       p.innerText =
    //         TARGET_CLASSES[predictions[2][n]] +
    //         ": " +
    //         Math.round(parseFloat(predictions[1][n]) * 100) +
    //         "%";

    //       let bboxLeft = predictions[0][n][0] * selectedImage.width + 10;
    //       let bboxTop = predictions[0][n][1] * selectedImage.height - 10;
    //       let bboxWidth =
    //         predictions[0][n][2] * selectedImage.width - bboxLeft + 20;
    //       let bboxHeight =
    //         predictions[0][n][3] * selectedImage.height - bboxTop + 10;

    //       p.style =
    //         "margin-left: " +
    //         bboxLeft +
    //         "px; margin-top: " +
    //         (bboxTop - 10) +
    //         "px; width: " +
    //         bboxWidth +
    //         "px; top: 0; left: 0;";
    //       const highlighter = document.createElement("div");
    //       highlighter.setAttribute("class", "highlighter");
    //       highlighter.style =
    //         "left: " +
    //         bboxLeft +
    //         "px; top: " +
    //         bboxTop +
    //         "px; width: " +
    //         bboxWidth +
    //         "px; height: " +
    //         bboxHeight +
    //         "px;";
    //       imageOverlay.appendChild(highlighter);
    //       imageOverlay.appendChild(p);
    //       // children.push(highlighter);
    //       // children.push(p);
    //       // document.body.append(p)
    //     }
    //   }
    // },
    async detectWithAruze(file) {
      return fetch(
        "https://tensorflowapp-prediction.cognitiveservices.azure.com/customvision/v3.0/Prediction/da302476-9603-4307-92c6-f3e9243853fa/detect/iterations/Iteration1/image",
        {
          method: "POST",
          headers: {
            "Content-Type": "application/octet-stream",
            "Prediction-Key": "3f9b6fd6e4fb4f6e841063b2eb64a47b",
          },
          body: file,
        }
      )
        .then((response) => response.json())
        .then((data) => {
          console.log("Success:", data);
          // this.highlightResults(data.predictions);
          return data.predictions;
        })
        .catch((error) => {
          console.error("Error:", error);
        });
    },
    async highlightResults(predictions) {
      console.log(predictions);
      console.log("Highlighting results...");
      const selectedImage = document.getElementById("imgInvisible");
      const imageOverlay = document.getElementById("imageOverlay");

      const p = document.createElement("p");
      p.innerText =
        predictions[0].tagName +
        ": " +
        Math.round(parseFloat(predictions[0].probability) * 100) +
        "%";

      let bboxLeft = predictions[0].boundingBox.left * selectedImage.width;
      let bboxTop = predictions[0].boundingBox.top * selectedImage.height;
      let bboxWidth = predictions[0].boundingBox.width * selectedImage.width;
      let bboxHeight = predictions[0].boundingBox.height * selectedImage.height;

      p.style =
        "margin-left: " +
        bboxLeft +
        "px; margin-top: " +
        bboxTop +
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
      this.cutImageByCoordinates(
        selectedImage,
        bboxLeft,
        bboxTop,
        bboxWidth,
        bboxHeight
      );
      imageOverlay.appendChild(highlighter);
      imageOverlay.appendChild(p);
    },
    cutImageByCoordinates(image, x, y, width, height) {
      // Create a canvas element
      var canvas = document.createElement("canvas");

      // Set the dimensions of the canvas
      canvas.width = width;
      canvas.height = height;

      // Get the 2D drawing context of the canvas
      var ctx = canvas.getContext("2d");

      // Draw the portion of the image onto the canvas
      ctx.drawImage(image, x, y, width, height, 0, 0, width, height);

      // Get the resulting image data from the canvas
      this.cutImage = canvas.toDataURL();

      // Use the image data as needed (e.g., display it or save it)
      // imageDataURL contains the data URL of the resulting image
      console.log(this.cutImage);
    },

    async detectImage() {
      if (!this.model) return;
      const image = document.getElementById("imgInvisible");
      const result = await this.model.executeAsync(image);
      console.log("🚀 ~ file: App.vue:247 ~ detectImage ~ result:", result);
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
  color: #fff;
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

.size {
  position: fixed;
  top: 0;
  left: 0;
}
</style>
