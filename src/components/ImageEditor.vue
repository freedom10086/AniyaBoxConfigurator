<script setup>
import {inject, onMounted, provide, reactive, ref, watch} from "vue";

defineProps({
  msg: {
    type: String,
    required: true
  }
})

const deviceConnected = inject("deviceConnected");
const bluetoothServer = inject('bluetoothServer');

const emit = defineEmits(['uploadImgSuccess']);
let fileId = Math.floor((Math.random() * 254) + 1);

const avatarCanvas = ref(null);
const imgInput = ref(null);
const preViewImg = ref(null);
const hasChooseImg = ref(false);

const redValueModel = defineModel('redValueModel', {default: 30});
const greenValueModel = defineModel('greenValueModel', {default: 59});
const blueValueModel = defineModel('blueValueModel', {default: 11});
const grayValueModel = defineModel('grayValueModel', {default: 128});
const inverseValueModel = defineModel('inverseValueModel', {default: false});
const effectModel = defineModel('effectModel', {default: "0"});

const colorModels = reactive({
  redValueModel,
  greenValueModel,
  blueValueModel,
  grayValueModel,
  inverseValueModel,
  effectModel
});

watch(colorModels, () => {
  console.log("colorModels change update");
  updatePreview();
})

let context;
let img, imgX = 0, imgY = 0, imgScale = 1;
let imgWidth, imgHeight;
let len;

onMounted(() => {
  len = avatarCanvas.value.offsetWidth;
  context = avatarCanvas.value.getContext('2d', {willReadFrequently: true,});

  imgInput.value.onchange = function (e) {
    let file = e.target.files[0];
    let size = file.size;
    let kbsize = Math.round(size * 100 / 1024) / 100;
    if (kbsize > 5120) {
      alert("图片太大,大小为" + kbsize + "kb！请选择小于5M的图片");
      return false;
    }
    let reader = new FileReader();
    reader.onload = function () {
      let res = reader.result;
      img = new Image();
      img.src = res;
      img.onload = function () {
        imgX = imgY = 0;
        if (img.width < img.height) {//w小
          imgWidth = len;
          imgHeight = img.height * len / img.width;
        } else {//h 小
          imgHeight = len;
          imgWidth = img.width * len / img.height;
        }
        //图片的坐标
        let px = (len - imgWidth) / 2;
        let py = (len - imgHeight) / 2;
        context.clearRect(0, 0, len, len);
        context.drawImage(img, px, py, imgWidth, imgHeight);

        //context.fillStyle = "red";
        //context.fillRect(0, 0, len, len);

        hasChooseImg.value = true;

        updatePreview();
      }
    };
    reader.readAsDataURL(file);
  };
})

function drawImage() {
  context.clearRect(0, 0, len, len);
  context.drawImage(img, imgX, imgY, imgWidth * imgScale, imgHeight * imgScale);

  updatePreview();
}

function avatarCanvasMouseDown(event) {
  if (!hasChooseImg.value) {
    return;
  }

  let pos = windowToCanvas(avatarCanvas.value, event.clientX, event.clientY);
  avatarCanvas.value.onmousemove = function (event) {
    avatarCanvas.value.style.cursor = "move";
    let pos1 = windowToCanvas(avatarCanvas.value, event.clientX, event.clientY);
    let x = pos1.x - pos.x;
    let y = pos1.y - pos.y;
    pos = pos1;
    imgX += x;
    imgY += y;

    if (imgX < len - imgWidth * imgScale) {
      imgX = len - imgWidth * imgScale;
    }
    if (imgX > 0) {
      imgX = 0;
    }
    if (imgY < len - imgHeight * imgScale) {
      imgY = len - imgHeight * imgScale;
    }
    if (imgY > 0) {
      imgY = 0;
    }
    drawImage();
  };

  document.onmouseup = function () {
    avatarCanvas.value.onmousemove = null;
    document.onmouseup = null;
    avatarCanvas.value.style.cursor = "default";
  };

  return false;
}

function avatarCanvasMouseWheel(event) {
  if (!hasChooseImg.value) {
    return;
  }

  let pos = windowToCanvas(avatarCanvas.value, event.clientX, event.clientY);
  console.log("pos:", pos.x);
  if (event.wheelDelta > 0) {
    imgScale *= 2;
    if (imgScale > 5) {
      imgScale /= 2;
      return;
    }
    imgX = imgX * 2 - pos.x;
    imgY = imgY * 2 - pos.y;
    drawImage();
  } else {
    imgScale /= 2;
    if (imgScale < 1 || imgWidth * imgScale < len || imgHeight * imgScale < len) {
      imgScale *= 2;
      return;
    }
    imgX = imgX / 2 + pos.x / 2;
    imgY = imgY / 2 + pos.y / 2;
    drawImage();
  }
  return false;
}

function windowToCanvas(canvas, x, y) {
  let bbox = canvas.getBoundingClientRect();
  return {
    x: x - bbox.left - (bbox.width - canvas.width) / 2,
    y: y - bbox.top - (bbox.height - canvas.height) / 2
  };
}

function updatePreview() {
  preViewImg.value.src = CanvasToBMP.toDataURL(avatarCanvas.value);
}

let CanvasToBMP = {
  toArrayBuffer: function (canvas) {
    let w = canvas.width, h = canvas.height;
    let w4 = w * 4;
    // let context = canvas.getContext("2d", {
    //   willReadFrequently: true,
    // });

    let idata = context.getImageData(0, 0, w, h);
    let data32 = new Uint32Array(idata.data.buffer); // 32-bit representation of canvas

    if (effectModel.value === "1") {
      data32 = floydSteinberg({data: data32, width: w});
    } else if (effectModel.value === "2") {
      data32 = atkinsonDithering({data: data32, width: w});
    }

    let stride = Math.floor((w + 31) * 32 / 32) >> 3; // row length incl. padding
    let pixelArraySize = stride * h;                 // total bitmap size
    let fileLength = 0x3e + pixelArraySize;           // header size is known + bitmap

    let file = new ArrayBuffer(fileLength);          // raw byte buffer (returned)
    let view = new DataView(file);                   // handle endian, reg. width etc.
    let pos = 0, x, y = 0, p, s = 0, a, v;

    // write file header
    setU16(0x4d42);           // BM BMP文件头
    setU32(fileLength);             // total length
    setU16(0);                // reserve 1
    setU16(0);                // reserve 2
    setU32(0x3e);             // offset to pixels

    // DIB header
    setU32(40);             // header size
    setU32(w);
    setU32(-h >>> 0);        // negative = top-to-bottom
    setU16(1);               // 1 plane
    setU16(1);               // 黑白单色
    setU32(0);               // no compression (BI_RGB, 0)
    setU32(stride * h);      // bitmap size incl. padding (stride x height)
    setU32(0);                // pixels/meter h (~72 DPI x 39.3701 inch/m)
    setU32(0);                // pixels/meter v
    setU32(0);               // biClrUsed
    setU32(0);               // biClrImportant

    setU32(0x00ffffff);
    setU32(0);

    // bitmap data, change order of ABGR to BGRA
    while (y < h) {
      p = 0x3e + y * stride; // offset + stride x height
      x = 0;
      while (x < w4) {
        v = data32[s++];                     // get ABGR
        a = v >>> 24;                        // alpha channel
        // 0 , 1
        let colored = bgrToGray(v) < Number(grayValueModel.value) ? 1 : 0;
        if (inverseValueModel.value) {
          colored = colored === 0 ? 1 : 0;
        }

        if (colored === 1) {
          let d = view.getUint8(p + (x >> 5));
          view.setUint8(p + (x >> 5), d | 1 << (7 - ((x % 32) >> 2)));
        }
        x += 4;
      }
      y++
    }

    return file;

    /**
     * rgb 转黑白
     * mono = (0.2125 * color.r) + (0.7154 * color.g) + (0.0721 * color.b);
     * mono = (0.3* color.R + 0.59 * color.G + 0.11 * color.B);
     */
    function bgrToGray(bgrColor) {
      let sum = Number(redValueModel.value) + Number(greenValueModel.value) + Number(blueValueModel.value);
      if (sum === 0) {
        return 255;
      }

      return ((bgrColor & 0xff) * Number(redValueModel.value)
              + ((bgrColor >> 8) & 0xff) * Number(greenValueModel.value)
              + ((bgrColor >> 16) & 0xff) * Number(blueValueModel.value))
          / (sum);
    }

    // helper method to move current buffer position
    function setU16(data) {
      view.setUint16(pos, data, true);
      pos += 2
    }

    function setU32(data) {
      view.setUint32(pos, data, true);
      pos += 4
    }
  },

  toBlob: function (canvas) {
    return new Blob([this.toArrayBuffer(canvas)], {
      type: "image/bmp"
    });
  },

  toDataURL: function (canvas) {
    var buffer = new Uint8Array(this.toArrayBuffer(canvas)),
        bs = "", i = 0, l = buffer.length;
    while (i < l) bs += String.fromCharCode(buffer[i++]);
    return "data:image/bmp;base64," + btoa(bs);
  }
};

function presetClick(presetType) {
  if (presetType === 1) {
    redValueModel.value = 30;
    greenValueModel.value = 59;
    blueValueModel.value = 11;
  } else {
    redValueModel.value = 21;
    greenValueModel.value = 72;
    blueValueModel.value = 7;
  }
}

function uploadImgClick() {
  uploadImg(CanvasToBMP.toArrayBuffer(avatarCanvas.value));
}

function uploadImg(imgData) {
  console.log("rev upload img event", imgData);
  //imgData  ArrayBuffer
  const PACK_LEN = 512;
  const BOX_SETTING_CMD_UPLOAD_BMP_START = 10;
  const BOX_SETTING_CMD_UPLOAD_BMP_DATA = 11;

  let dataLen = imgData.byteLength;
  let packCount = Math.floor((dataLen + PACK_LEN - 1) / PACK_LEN);

  bluetoothServer.value.getPrimaryService('6e400001-b5a3-f393-e0a9-e50e24dcca9e').then(service => {
        //console.log('获取主要服务成功uart', service);
        service.getCharacteristic('6e400002-b5a3-f393-e0a9-e50e24dcca9e').then(characteristic => {
          fileId = (fileId + 1) % 254 + 1;
          (async () => {
            for (let i = 0; i < packCount; i++) {
              let buffStart = new ArrayBuffer(4);
              let view = new DataView(buffStart);
              if (i === 0) {
                // [0] id [1, 2] file_size, [..., end] data
                // start pack
                view.setUint8(0, BOX_SETTING_CMD_UPLOAD_BMP_START);
                view.setUint8(1, fileId);
                view.setUint16(2, dataLen, true);
              } else {
                // data pack
                // [0] id [1, 2] offset, [..., end] data
                view.setUint8(0, BOX_SETTING_CMD_UPLOAD_BMP_DATA);
                view.setUint8(1, fileId);
                let offset = i * PACK_LEN;
                view.setUint16(2, offset, true);
              }

              let buffData = imgData.slice(i * PACK_LEN, Math.min((i + 1) * PACK_LEN, dataLen));
              let resultBuff = concatBuffers(buffStart, buffData);
              console.log("pack ", i, "data", resultBuff);

              try {
                await characteristic.writeValue(resultBuff);
                console.log("write data success");
                if (i === packCount - 1) {
                  alert("upload success");
                  emit('uploadImgSuccess');
                }
              } catch (e) {
                console.log("write data failed", e);
                alert("上传失败" + e);
                break;
              }
            }
          })();
        }).catch(error => {
          console.error('uart getCharacteristic 错误:', error);
          alert("上传失败" + error);
        })
      }
  ).catch(error => {
    console.error('获取主要服务错误uart:', error);
    alert("上传失败" + error);
  });

  function concatBuffers(a, b) {
    return concatTypedArrays(
        new Uint8Array(a.buffer || a),
        new Uint8Array(b.buffer || b)
    );
  }

  function concatTypedArrays(a, b) { // a, b TypedArray of same type
    let c = new (a.constructor)(a.length + b.length);
    c.set(a, 0);
    c.set(b, a.length);
    return c;
  }
}

function floydSteinberg(image) {
  const originImageData = image.data;
  const imageDataLength = originImageData.length;
  const width = image.width;

  let imageData = new Uint32Array(imageDataLength);

  for (let i = 0; i < imageDataLength; i += 1) {
    const red = originImageData[i] & 0xff;
    const green = ((originImageData[i] >> 8) & 0xff);
    const blue = ((originImageData[i] >> 16) & 0xff);

    let sum = Number(redValueModel.value) + Number(greenValueModel.value) + Number(blueValueModel.value);
    if (sum === 0) {
      imageData[i] = 255;
    } else {
      imageData[i] = Math.floor(red * Number(redValueModel.value) / sum + green * Number(greenValueModel.value) / sum + blue * Number(blueValueModel.value) / sum);
    }
  }

  // Combine greyscaling and dithering in a single loop for efficiency
  for (let i = 0; i < imageDataLength; i += 1) {
    const luminance = imageData[i];
    const newPixel = luminance < 150 ? 0 : 255;
    const err = Math.floor((luminance - newPixel) / 23);

    imageData[i] = newPixel;

    // Distribute error to neighboring pixels
    if (i + 1 < imageDataLength) imageData[i + 1] += err * 7;
    if (i >= width) imageData[i - width] += err * 3;
    if (i >= width) imageData[i - width + 1] += err * 5;
    if (i + width + 1 < imageDataLength) imageData[i + width + 1] += err * 1;
  }

  for (let i = 0; i < imageDataLength; i += 1) {
    imageData[i] = imageData[i] | (imageData[i] << 8) | (imageData[i] << 16);
  }

  return imageData;
}

function atkinsonDithering(image) {
  const originImageData = image.data;
  const imageDataLength = originImageData.length;
  const w = image.width;

  let imageData = new Uint32Array(imageDataLength);

  for (let i = 0; i < imageDataLength; i += 1) {
    const red = originImageData[i] & 0xff;
    const green = ((originImageData[i] >> 8) & 0xff);
    const blue = ((originImageData[i] >> 16) & 0xff);

    let sum = Number(redValueModel.value) + Number(greenValueModel.value) + Number(blueValueModel.value);
    if (sum === 0) {
      imageData[i] = 255;
    } else {
      imageData[i] = Math.floor(red * Number(redValueModel.value) / sum + green * Number(greenValueModel.value) / sum + blue * Number(blueValueModel.value) / sum);
    }
  }

  // Dithering using simplified array access
  for (let currentPixel = 0; currentPixel <= imageDataLength; currentPixel += 1) {
    const currentR = imageData[currentPixel];
    const nextPixel = currentPixel + 1;
    const nextRow = currentPixel + 1 * w;

    const newPixel = currentR < grayValueModel.value ? 0 : 255;
    const err = Math.floor((currentR - newPixel) / 8);

    imageData[currentPixel] = newPixel;
    imageData[nextPixel] += err;
    imageData[nextPixel + 1] += err;
    imageData[nextRow - 1] += err;
    imageData[nextRow] += err;
    imageData[nextRow + 1] += err;
    imageData[nextRow + w * 1] += err;
  }

  for (let i = 0; i < imageDataLength; i += 1) {
    imageData[i] = imageData[i] | (imageData[i] << 8) | (imageData[i] << 16);
  }

  return imageData;
}
</script>

<template>
  <div style="display: flex; flex-direction: column; margin-top: 5px;">
    <div><label style="font-weight: bold">上传图片</label></div>
    <div style="display: flex;flex-direction: row;">
      <div style="display: flex;flex-direction: column;">
        <input ref="imgInput" type="file" accept="image/*;"/>
        <canvas ref="avatarCanvas" @mousedown="avatarCanvasMouseDown" @mousewheel="avatarCanvasMouseWheel"
                width="200" height="200" title="选择图片" style="margin-top: 6px;"></canvas>
      </div>
      <div style="display: flex;flex-direction: column;">
        <div><b>预览</b>
          <button @click="updatePreview">刷新</button>
        </div>
        <img ref="preViewImg" width="200" height="200" alt="" src="" style="margin-top: 6px"/>
      </div>
      <div style="margin-left: 10px">
        <b>参数</b>
        <div class="row">
          <label for="gray">V:</label>
          <input type="range" id="gray" name="gray" min="0" max="255" v-model="grayValueModel"/>
          <label>{{ grayValueModel }}</label>
        </div>
        <div class="row">
          <label for="red">R:</label>
          <input type="range" id="red" name="red" min="0" max="100" v-model="redValueModel"/>
          <label>{{ redValueModel }}</label>
        </div>
        <div class="row">
          <label for="green">G:</label>
          <input type="range" id="green" name="green" min="0" max="100" v-model="greenValueModel"/>
          <label>{{ greenValueModel }}</label>
        </div>
        <div class="row">
          <label for="blue">B:</label>
          <input type="range" id="blue" name="blue" min="0" max="100" v-model="blueValueModel"/>
          <label>{{ blueValueModel }}</label>
        </div>
        <div class="row">
          <label for="inverse">取反:</label>
          <input type="checkbox" id="inverse" name="inverse" v-model="inverseValueModel"/>
        </div>
        <div class="row" style="margin-top: 12px">
          <button @click="presetClick(1)">预设1</button>
          <button @click="presetClick(2)" style="margin-left: 8px;">预设2</button>
        </div>
        <div class="row">
          <label style="font-weight: bold">处理:</label>
          <label>无
            <input type="radio" name="effect" value="0" v-model="effectModel"/></label>
          <label style="margin-left: 5px">floydSteinberg:
            <input type="radio" name="effect" value="1" v-model="effectModel"/></label>
          <label style="margin-left: 5px">atkinsonDithering:
            <input type="radio" name="effect" value="2" v-model="effectModel"/></label>
        </div>
      </div>
    </div>
    <div style="margin-top: 10px;" v-if="hasChooseImg && deviceConnected">
      <button @click="uploadImgClick">上传</button>
    </div>
  </div>
</template>

<style scoped>
.row {
  display: flex;
  flex-direction: row;
}

canvas {
  border: 1px solid #efefef;
  width: 200px;
  height: 200px
}

img {
  border: 1px solid #efefef;
  width: 200px;
  height: 200px;
}
</style>
