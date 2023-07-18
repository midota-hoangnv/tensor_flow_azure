<template lang="">
  <div>
    <div>
      <h1>Real-Time Object Detection: Kangaroo</h1>
      <h3>MobileNetV2</h3>
    </div>
    <div class="main-section">
      <div class="camera-wrapper">
        <video
          class="camera-element"
          autoPlay
          playsInline
          muted
          ref="videoRef"
          id="frame"
        />
        <canvas class="detect-area" ref="canvasRef" />
      </div>
      <div>
        <figure>
          <div class="preview-wrapper">
            <img :src="originalImage" alt="" ref="originalCaptureImageRef" />
            <canvas class="preview-detect-area" ref="canvasPreviewRef" />
          </div>
          <figcaption>Capture image</figcaption>
        </figure>
        <figure>
          <img :src="cutImage" alt="" />
          <figcaption>Cut Image</figcaption>
        </figure>
        <div class="action-buttons">
          <button @click="openCamera">Open camera</button
          ><button @click="detectFrame">Capture Image</button>
        </div>
      </div>
      <div>
        <figure>
          <img src="/example.png" alt="" />
          <figcaption>Original Image</figcaption>
        </figure>
      </div>
    </div>
  </div>
</template>
<script>
export default {
  data() {
    return {
      model: null,
      isNewOdModel: null,
      cutImage: null,
      originalImage: null,
      canDetect: false,
      predictions: [],
      animationFixedFrame: false,
      previousTimestamp: 0,
    };
  },
  mounted() {
    // this.openCamera();
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
    openCamera() {
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
            this.canDetect = true;
            this.loadFrame();
          })
          .catch((error) => {
            console.error(error);
          });
      }
    },

    loadFrame() {
      requestAnimationFrame((timestamp) => {
        this.drawFixedFrame(timestamp);
        this.loadFrame();
      });
    },

    drawFixedFrame(timestamp) {
          // Kiểm tra thời gian đã trôi qua 1 giây
      if (timestamp - this.previousTimestamp < 1000) return;
      this.previousTimestamp = timestamp;
      const ctx = this.$refs.canvasRef.getContext("2d");
      ctx.canvas.width = ctx.canvas.clientWidth;
      ctx.canvas.height = ctx.canvas.clientHeight;
      ctx.clearRect(0, 0, ctx.canvas.width, ctx.canvas.height);

      // Draw the bounding box.
      ctx.beginPath();
      // Set line color
      if (this.animationFixedFrame) {
        ctx.strokeStyle = "red";
      } else {
        ctx.strokeStyle = "blue";
      }

      // Set line width
      ctx.lineWidth = 2;
      const centerX = ctx.canvas.width / 2;
      const centerY = ctx.canvas.height / 2;
      const rectangleWidth = 100;
      const rectangleHeight = 100;

      const topLeft = {
        x: centerX - rectangleWidth / 2,
        y: centerY - rectangleHeight / 2,
        yPosition: "top",
        xPosition: "left",
      };
      const topRight = {
        x: centerX + rectangleWidth / 2,
        y: centerY - rectangleHeight / 2,
        yPosition: "top",
        xPosition: "right",
      };
      const bottomLeft = {
        x: centerX - rectangleWidth / 2,
        y: centerY + rectangleHeight / 2,
        yPosition: "bottom",
        xPosition: "left",
      };
      const bottomRight = {
        x: centerX + rectangleWidth / 2,
        y: centerY + rectangleHeight / 2,
        yPosition: "bottom",
        xPosition: "right",
      };

      [topLeft, topRight, bottomLeft, bottomRight].forEach((corner) => {
        drawCorner(corner);
      });
      ctx.stroke();
      function drawCorner(corner) {
        ctx.moveTo(corner.x, corner.y);
        if (corner.xPosition == "left") {
          ctx.lineTo(corner.x + 15, corner.y);
        } else {
          ctx.lineTo(corner.x - 15, corner.y);
        }
        ctx.moveTo(corner.x, corner.y);
        if (corner.yPosition == "top") {
          ctx.lineTo(corner.x, corner.y + 15);
        } else {
          ctx.lineTo(corner.x, corner.y - 15);
        }
      }
      this.animationFixedFrame = !this.animationFixedFrame;
      ctx.closePath();
    },

    getDitectedImageDimension() {
      if (!this.$refs.originalCaptureImageRef) return { width: 0, height: 0 };
      return {
        width: this.$refs.originalCaptureImageRef.clientWidth,
        height: this.$refs.originalCaptureImageRef.clientHeight,
      };
    },

    getFinalObjectDetection() {
      if (!this.predictions?.length) return;
      //Getting predictions
      const imageDimension = this.getDitectedImageDimension();
      const tagName = this.predictions[0].tagName;
      const point = Math.round(
        parseFloat(this.predictions[0].probability) * 100
      );

      let x = this.predictions[0].boundingBox.left * imageDimension.width;
      let y = this.predictions[0].boundingBox.top * imageDimension.height;
      let width = this.predictions[0].boundingBox.width * imageDimension.width;
      let height =
        this.predictions[0].boundingBox.height * imageDimension.height;

      return {
        x,
        y,
        width,
        height,
        tagName,
        point,
      };
    },

    loadDetectionPreviewArea() {
      const finalObjectDetection = this.getFinalObjectDetection();
      if (!finalObjectDetection) return;
      const { x, y, width, height, tagName, point } = finalObjectDetection;
      console.log(finalObjectDetection);
      const ctx = this.$refs.canvasPreviewRef.getContext("2d");
      const imageDimension = this.getDitectedImageDimension();

      ctx.canvas.width = imageDimension.width;
      ctx.canvas.height = imageDimension.height;
      ctx.clearRect(0, 0, ctx.canvas.width, ctx.canvas.height);

      // Font options.
      const font = "16px sans-serif";
      ctx.font = font;
      ctx.textBaseline = "top";

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

    async loadCutImage() {
      const finalObjectDetection = this.getFinalObjectDetection();
      if (finalObjectDetection) {
        const { x, y, width, height } = finalObjectDetection;
        await this.cutImageByCoordinates(
          this.originalImage,
          x,
          y,
          width,
          height
        );
      }
    },
    detectFrame() {
      let video = this.$refs.videoRef;
      let canvas = document.createElement("canvas");
      canvas.width = video.clientWidth;
      canvas.height = video.clientHeight;

      let ctx = canvas.getContext("2d");
      ctx.drawImage(video, 0, 0, canvas.width, canvas.height);

      canvas.toBlob((blob) => {
        console.log("blob", blob);
        if (blob) {
          this.originalImage = URL.createObjectURL(blob);
          this.predictions = [];
          this.detectWithAruze(blob).then((predictions) => {
            console.log(predictions);
            this.predictions = predictions;
            this.loadCutImage();
            this.loadDetectionPreviewArea();
            // requestAnimationFrame(() => {
            // this.detectFrame(video);
            // });
          });
        }
      });
    },
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
          return data.predictions;
        })
        .catch((error) => {
          console.error("Error:", error);
        });
    },
    async cutImageByCoordinates(imageSrc, x, y, width, height) {
      // Create a canvas element
      var canvas = document.createElement("canvas");

      // Set the dimensions of the canvas
      canvas.width = width;
      canvas.height = height;

      // Get the 2D drawing context of the canvas
      var ctx = canvas.getContext("2d");

      const image = await this.getLoadedImage(imageSrc);
      // Draw the portion of the image onto the canvas
      ctx.drawImage(image, x, y, width, height, 0, 0, width, height);

      // Get the resulting image data from the canvas
      this.cutImage = canvas.toDataURL();

      // Use the image data as needed (e.g., display it or save it)
      // imageDataURL contains the data URL of the resulting image
    },
    async getLoadedImage(src) {
      return new Promise((resolve, reject) => {
        let img = new Image();
        img.onload = () => {
          resolve(img);
        };
        img.onerror = reject;
        img.src = src;
      });
    },
  },
};
</script>

<style lang="css">
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

.highlighter {
  background: rgba(0, 255, 0, 0.25);
  border: 1px dashed #fff;
  z-index: 1;
  position: absolute;
}

.camera-wrapper {
  width: 375px;
  height: 667px;
  position: relative;
  border: 1px solid #eee;
}
.camera-element {
  width: 100%;
  height: 100%;
  object-fit: fill;
}
.detect-area {
  position: absolute;
  top: 0;
  right: 0;
  bottom: 0;
  left: 0;
  width: 100%;
  height: 100%;
}
.main-section,
.action-buttons {
  display: flex;
  gap: 10px;
}
.main-section {
  justify-content: space-between;
}
.preview-wrapper {
  position: relative;
}
.preview-detect-area {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  width: 100%;
  height: 100%;
}

button {
  cursor: pointer;
}
figure {
  border: 1px solid #eee;
  padding: 10px;
}
</style>
