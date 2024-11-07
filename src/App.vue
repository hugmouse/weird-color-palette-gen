<script lang="ts">
  import {onMounted, Ref, ref, watch} from "vue";

  export default {
    setup() {
      // Variables to manage image sources and canvas data
      const imageSrc: Ref<string | null> = ref(null);
      const pixelColors: Ref<string[]> = ref([]);
      const processedCanvasData: Ref<string | null> = ref(null);
      const resizedCanvasData: Ref<string | null> = ref(null);

      // Card color
      const cardBackgroundColor: Ref<string> = ref('');
      const buttonColor: Ref<string> = ref('');
      const textColor: Ref<string> = ref('');
      const buttonTextColor: Ref<string> = ref('');

      // References to different canvases used for drawing
      const processedCanvas = ref<HTMLCanvasElement | null>(null);
      const currentPixelCanvas = ref<HTMLCanvasElement | null>(null);

      const onFileChange = (e: Event) => {
        const target = e.target as HTMLInputElement;
        const file = target.files ? target.files[0] : null;

        if (file && file.type.startsWith("image/")) {
          const reader = new FileReader();
          reader.onload = (event) => {
            imageSrc.value = event.target?.result as string;
            const img = new Image();
            img.crossOrigin = "Anonymous";
            img.onload = () => {
              processImage(img);
            };
            img.src = imageSrc.value;
          };
          reader.readAsDataURL(file);
        }
      };

      const processImage = (img: HTMLImageElement) => {
        const canvas = document.createElement("canvas");
        const context = canvas.getContext("2d")!;
        canvas.width = img.width;
        canvas.height = img.height;
        context.drawImage(img, 0, 0);

        const filteredPixels: number[] = [];
        const imgData = context.getImageData(0, 0, canvas.width, canvas.height);
        const data = imgData.data;

        let i = 0;
        let ii = 0;
        const pixelsPerInterval = 1332;

        const processNextPixels = () => {
          const end = Math.min(i + pixelsPerInterval * 4, data.length);
          for (; i < end; i += 4) {
            const r = data[i];
            const g = data[i + 1];
            const b = data[i + 2];
            const a = data[i + 3];

            updateCurrentPixelCanvas(r, g, b, a, i, canvas.width);

            if (!isBlackWhiteTransparent(r, g, b, a)) {
              filteredPixels.push(r, g, b, a);
              ii += 4;
              drawProcessedPixel(r, g, b, a, ii, canvas.width, canvas.height);
            } else {
              drawProcessedPixel(0, 0, 0, 255, i, canvas.width, canvas.height);
            }
          }

          if (i < data.length) {
            requestAnimationFrame(processNextPixels);
          } else {
            handleFilteredPixels(filteredPixels);
          }
        };

        processNextPixels();
      };

      const updateCurrentPixelCanvas = (r: number, g: number, b: number, a: number, index: number, width: number) => {
        if (currentPixelCanvas.value) {
          const ctx = currentPixelCanvas.value.getContext("2d")!;
          ctx.clearRect(0, 0, currentPixelCanvas.value.width, currentPixelCanvas.value.height);
          ctx.fillStyle = `rgba(${r}, ${g}, ${b}, ${a / 255})`;
          ctx.fillRect(0, 0, 50, 50);
          ctx.fillStyle = "white";
          ctx.font = "16px Monospace";
          ctx.fillText(`Index: ${index / 4}`, 60, 20);
          ctx.fillText(`Coordinates: (${(index / 4) % width}, ${Math.floor((index / 4) / width)})`, 60, 40);
          ctx.fillText(`Color: rgba(${r}, ${g}, ${b}, ${a})`, 60, 60);
        }
      };

      const drawProcessedPixel = (r: number, g: number, b: number, a: number, index: number, width: number, height: number) => {
        if (processedCanvas.value) {
          const processedCtx = processedCanvas.value.getContext("2d")!;
          if (index === 0) {
            processedCanvas.value.width = width;
            processedCanvas.value.height = height;
          }
          processedCtx.fillStyle = `rgba(${r}, ${g}, ${b}, ${a / 255})`;
          processedCtx.fillRect((index / 4) % width, Math.floor((index / 4) / width), 1, 1);
        }
      };

      const isBlackWhiteTransparent = (r: number, g: number, b: number, a: number) => {
        return (
            (r === 255 && g === 255 && b === 255) ||
            (r === 0 && g === 0 && b === 0) ||
            a === 0
        );
      };

      const handleFilteredPixels = (filteredPixels: number[]) => {
        if (filteredPixels.length === 0) {
          alert("No pixels left after filtering. The image may contain only black, white, or transparent pixels.");
          processedCanvasData.value = null;
          return;
        }

        const totalPixels = filteredPixels.length / 4;
        const newWidth = Math.ceil(Math.sqrt(totalPixels));
        const newHeight = Math.ceil(totalPixels / newWidth);

        const newCanvas = document.createElement("canvas");
        newCanvas.width = newWidth;
        newCanvas.height = newHeight;
        const newContext = newCanvas.getContext("2d")!;
        const newImageData = newContext.createImageData(newWidth, newHeight);

        newImageData.data.set(filteredPixels);
        newContext.putImageData(newImageData, 0, 0);

        processedCanvasData.value = newCanvas.toDataURL();
        processResizedImage(newCanvas);
      };

      const processResizedImage = (canvas: HTMLCanvasElement) => {
        const smallCanvas = document.createElement("canvas");
        smallCanvas.width = 2;
        smallCanvas.height = 2;
        const smallContext = smallCanvas.getContext("2d")!;
        smallContext.imageSmoothingEnabled = false;
        smallContext.drawImage(canvas, 0, 0, canvas.width, canvas.height, 0, 0, 2, 2);
        resizedCanvasData.value = smallCanvas.toDataURL();

        const smallImgData = smallContext.getImageData(0, 0, 2, 2);
        const smallData = smallImgData.data;

        const displayColors: string[] = [];
        for (let i = 0; i < smallData.length; i += 4) {
          const r = smallData[i];
          const g = smallData[i + 1];
          const b = smallData[i + 2];
          const a = smallData[i + 3];
          if (a === 0) {
            displayColors.push("transparent");
          } else {
            displayColors.push(`rgba(${r}, ${g}, ${b}, ${a / 255})`);
          }
        }

        pixelColors.value = displayColors;
        generateCardColors();
      };

      // Function to generate colors for the card ensuring contrast
      const generateCardColors = () => {
        if (pixelColors.value.length === 4) {
          cardBackgroundColor.value = pixelColors.value[2];
          buttonColor.value = pixelColors.value[3];
          textColor.value = pixelColors.value[1];
          buttonTextColor.value = pixelColors.value[0];

          // Ensure contrast levels between elements
          ensureContrast();
        }
      };

      // Helper function to parse RGBA color strings, https://stackoverflow.com/a/21966100
      // by NiettheDarkAbsol and F. Hauri
      const parseRGBA = (color: string): { r: number; g: number; b: number; a: number } => {
        const res = color.split("(")[1].split(")")[0].split(",")
        if (res) {
          const r = parseInt(res[1], 10);
          const g = parseInt(res[2], 10);
          const b = parseInt(res[3], 10);
          const a = res[4] !== undefined ? parseFloat(res[4]) : 1;
          return {r, g, b, a};
        }
        throw new Error("Invalid color format");
      };


      // https://stackoverflow.com/questions/2353211/hsl-to-rgb-color-conversion
      // By Garry Tan
      /**
       * Converts an RGB color value to HSL. Conversion formula
       * adapted from http://en.wikipedia.org/wiki/HSL_color_space.
       * Assumes r, g, and b are contained in the set [0, 255] and
       * returns h, s, and l in the set [0, 1].
       **/
      function rgbToHsl(r: number, g: number, b: number) {
        (r /= 255), (g /= 255), (b /= 255);
        const vmax = Math.max(r, g, b), vmin = Math.min(r, g, b);
        let h:number, s:number, l:number = (vmax + vmin) / 2;

        if (vmax === vmin) {
          return [0, 0, l]; // achromatic
        }

        const d = vmax - vmin;
        s = l > 0.5 ? d / (2 - vmax - vmin) : d / (vmax + vmin);
        if (vmax === r) h = (g - b) / d + (g < b ? 6 : 0);
        if (vmax === g) h = (b - r) / d + 2;
        if (vmax === b) h = (r - g) / d + 4;
        h /= 6;

        return [h, s, l];
      }

      /**
       * Converts an HSL color value to RGB. Conversion formula
       * adapted from https://en.wikipedia.org/wiki/HSL_color_space.
       * Assumes h, s, and l are contained in the set [0, 1] and
       * returns r, g, and b in the set [0, 255].
       */
      function hslToRgb(h: number, s: number, l: number) {
        let r:number, g:number, b: number;

        if (s === 0) {
          r = g = b = l; // achromatic
        } else {
          const q = l < 0.5 ? l * (1 + s) : l + s - l * s;
          const p = 2 * l - q;
          r = hueToRgb(p, q, h + 1 / 3);
          g = hueToRgb(p, q, h);
          b = hueToRgb(p, q, h - 1 / 3);
        }

        return [Math.round(r * 255), Math.round(g * 255), Math.round(b * 255)];
      }

      function hueToRgb(p: number, q: number, t: number) {
        if (t < 0) t += 1;
        if (t > 1) t -= 1;
        if (t < 1 / 6) return p + (q - p) * 6 * t;
        if (t < 1 / 2) return q;
        if (t < 2 / 3) return p + (q - p) * (2 / 3 - t) * 6;
        return p;
      }


      // Get lightness from RGBA color
      const getLightness = (color: string): number => {
        const {r, g, b} = parseRGBA(color);
        const hsl = rgbToHsl(r, g, b);
        return hsl[2] * 100; // Lightness as percentage
      };

      // Set lightness for RGBA color
      const setLightness = (color: string, lightness: number): string => {
        const {r, g, b, a} = parseRGBA(color);
        const hsl = rgbToHsl(r, g, b);
        const rgb = hslToRgb(hsl[0], hsl[1], lightness / 100);
        return `rgba(${Math.round(rgb[0])}, ${Math.round(rgb[1])}, ${Math.round(rgb[2])}, ${a})`;
      };

      // Function to ensure contrast between colors
      const ensureContrast = () => {
        const bgLightness = getLightness(cardBackgroundColor.value);
        let buttonLightness = getLightness(buttonColor.value);
        let textLightness = getLightness(textColor.value);
        let buttonTextLightness = getLightness(buttonTextColor.value);

        // Adjust button color to ensure difference ≥ 20
        if (Math.abs(bgLightness - buttonLightness) <= 20) {
          if (bgLightness >= 50) {
            // Background is light, make button darker
            buttonLightness = bgLightness - 20;
          } else {
            // Background is dark, make button lighter
            buttonLightness = bgLightness + 20;
          }
          buttonColor.value = setLightness(buttonColor.value, buttonLightness);
        }

        // Adjust text color to ensure difference ≥ 60
        if (Math.abs(bgLightness - textLightness) < 60) {
          if (bgLightness >= 50) {
            // Background is light, make text darker
            textLightness = bgLightness - 60;
          } else {
            // Background is dark, make text lighter
            textLightness = bgLightness + 60;
          }
          textColor.value = setLightness(textColor.value, textLightness);
        }

        // Ensure button text color contrasts with button background
        if (Math.abs(buttonLightness - buttonTextLightness) < 60) {
          if (buttonLightness > 50) {
            // Button is light, make button text darker
            buttonTextLightness = buttonLightness - 60;
          } else {
            // Button is dark, make button text lighter
            buttonTextLightness = buttonLightness + 60;
          }
          buttonTextColor.value = setLightness(buttonTextColor.value, buttonTextLightness);
        }
      };

      // Watch for changes in processedCanvasData and update processed canvas
      onMounted(() => {
        watch(
            () => processedCanvasData.value,
            (newVal) => {
              if (newVal && processedCanvas.value) {
                const ctx = processedCanvas.value.getContext("2d")!;
                const img = new Image();
                img.onload = () => {
                  processedCanvas.value!.width = img.width;
                  processedCanvas.value!.height = img.height;
                  ctx.drawImage(img, 0, 0);
                };
                img.src = newVal;
              }
            }
        );
      });

      return {
        imageSrc,
        pixelColors,
        onFileChange,
        processedCanvasData,
        resizedCanvasData,
        processedCanvas,
        currentPixelCanvas,
        cardBackgroundColor,
        buttonColor,
        textColor,
        buttonTextColor,
      };
    },
  };
</script>

<template>
  <div class="flex">
    <div>
      <input type="file" @change="onFileChange" accept="image/*" />
      <div class="squares">
        <!-- Original Image -->
        <div class="square original">
          <p>Original Image</p>
          <img :src="imageSrc" alt="Original Image" v-if="imageSrc" />
        </div>
        <!-- Processed Image -->
        <div class="square processed">
          <p>Processed Image</p>
          <canvas ref="processedCanvas" v-if="imageSrc"></canvas>
        </div>
        <!-- Final 2x2 Pixel Grid -->
        <div class="square modified">
          <p>Final 2x2 Pixel Grid</p>
          <div class="pixel-grid" v-if="pixelColors.length === 4">
            <div
              v-for="(color, index) in pixelColors"
              :key="index"
              class="pixel"
              :style="{ backgroundColor: color }"
            ></div>
          </div>
        </div>
      </div>

      <!-- Current Pixel Information -->
      <div class="current-pixel-info">
        <h3>Processing Pixel</h3>
        <canvas ref="currentPixelCanvas" width="400" height="100"></canvas>
      </div>
    </div>

    <!-- Card with Original Image and Generated Colors -->
    <div
      v-if="imageSrc && pixelColors.length === 4"
      class="card"
      :style="{ backgroundColor: cardBackgroundColor }"
    >
      <img :src="imageSrc" alt="Original Image" class="card-image" />
      <h3 class="card-title" :style="{ color: textColor }">
        Smart App that Let's You Do Some Lorem Ipsum
      </h3>
      <button
        class="card-button"
        :style="{ backgroundColor: buttonColor, color: buttonTextColor }"
      >
        Download Application
      </button>
    </div>
  </div>
</template>

<style scoped>
  .flex {
    display: flex;
    gap: 16px;
    align-items: center;
    justify-content: center;
  }

  .squares {
    display: flex;
    gap: 20px;
  }

  input {
    margin-bottom: 1rem;
  }

  .square {
    width: 200px;
    position: relative;
    border: 1px solid #ccc;
    padding: 10px;
    text-align: center;
  }

  .square img,
  .square canvas {
    width: 100%;
    height: auto;
    object-fit: contain;
  }

  .pixel-grid {
    display: grid;
    grid-template-columns: repeat(2, 1fr);
    grid-template-rows: repeat(2, 1fr);
  }

  .pixel {
    aspect-ratio: 1/1;
  }

  .current-pixel-info {
    margin-top: 20px;
    padding: 10px;
    border: 1px solid #007BFF;
  }

  .current-pixel-info canvas {
    display: block;
    margin-bottom: 10px;
  }

  .card {
    width: 300px;
    margin-top: 20px;
    padding: 20px;
    border-radius: 10px;
    text-align: center;
    display: inline-block;
    box-shadow: 0 10px 20px 10px rgba(0, 0, 0, 0.2);
  }

  .card-image {
    max-width: 200px;
    object-fit: cover;
  }

  .card-title {
    margin-top: 10px;
    font-size: 18px;
    font-weight: bold;
  }

  .card-button {
    margin-top: 15px;
    padding: 10px 20px;
    border: none;
    border-radius: 5px;
    cursor: pointer;
  }
</style>