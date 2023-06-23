<script setup>
import GIF from "gif.js";
import {decompressFrames, parseGIF} from "gifuct-js";
import {ref, watch} from "vue";

const scale = ref(1);
const imgUint8Array = ref(null);
const picsList = ref([]);
const file = ref(null);
const imgURL = ref(null);
const result = ref(false);
const fileType = ref(null);
const finishedGif = ref(null);
const uploadImg = ref(null);

const handleFileUpload = (event) => {
  fileType.value = event.target.files[0].type;
  if(fileType.value !== 'image/gif') {
    file.value = event.target.files[0];


    var reader = new FileReader();
    // 读取文件数据
    reader.readAsDataURL(file.value);
    // 使用 FileReader 读取文件数据
    reader.onload = function (e) {
      var imageData = new Image();
      imageData.src = e.target.result;
      // 添加读取的图像数据作为 GIF 的一帧
      setTimeout(() => {
        transToGif([imageData]).then((res) => {
          file.value = res;
          const reader = new FileReader();
          reader.onload = (event) => {
            imgURL.value = event.target.result;
          };
          reader.readAsDataURL(res);
        });
      }, 100)
    };


    //
    // imgURL.value = URL.createObjectURL(file.value);
    // const img=document.getElementById('uploadImg')

  }else{
    file.value = event.target.files[0];
    const reader = new FileReader();
    reader.onload = (event) => {
      imgURL.value = event.target.result;
    };
    reader.readAsDataURL(file.value);
  }

}

watch(finishedGif, (newVal) => {
  if (newVal) {
    console.log('newVal', newVal)
    const img = document.createElement("img");
    img.src = URL.createObjectURL(newVal);

    document.body.appendChild(img);
  }
});
const startRendering = (img) => {
  const gif = parseGIF(imgUint8Array.value);
  const frames = decompressFrames(gif, true);
  for (let i = 0; i < frames.length; i++) {
    const frame = frames[i];
    const canvas = document.createElement("canvas");
    canvas.width = frame.dims.width;
    canvas.height = frame.dims.height;
    const context = canvas.getContext("2d");
    const imageData = context.createImageData(
        frame.dims.width,
        frame.dims.height
    );
    imageData.data.set(frame.patch);
    context.putImageData(imageData, 0, 0);

    // 对Canvas进行等比例缩小
    const scaledCanvas = document.createElement("canvas");
    const scaleRatio = scale.value; // 缩放比例为0.5
    scaledCanvas.width = canvas.width * scaleRatio;
    scaledCanvas.height = canvas.height * scaleRatio;
    const scaledContext = scaledCanvas.getContext("2d");
    scaledContext.drawImage(
        canvas,
        0,
        0,
        scaledCanvas.width,
        scaledCanvas.height
    );

    const image = new Image();

    image.src = scaledCanvas.toDataURL();
    picsList.value.push(image);
    // document.body.appendChild(image);
  }


  setTimeout(async () => {
    finishedGif.value = await transToGif(picsList.value);
  }, 100);
}
const transToGif = (piclist) => {
  return new Promise((resolve, reject) => {
    const gif = new GIF({
      workers: 4,
      quality: 1,
      transparent:'#fff' //这个在背景图片是透明时会用到
    });

    for (let i = 0; i < piclist.length; i++) {
      gif.addFrame(piclist[i], {delay: 100});
    }

    gif.on("finished", function (blob) {
      // const img = document.createElement("img");
      // img.src = URL.createObjectURL(blob);
      resolve(blob);
    });

    gif.on('error', function (err) {
      reject(err);
    });

    // 开始生成GIF
    gif.render();
  });

}

const mountFile = () => {
  picsList.value = [];
  const reader = new FileReader();
  // 将文件读取为二进制数据
  reader.readAsArrayBuffer(file.value);
  reader.onload = () => {
    imgUint8Array.value = new Uint8Array(reader.result);
    startRendering();
  };
}
</script>

<template>
  <div class="box">

    <input type="file" accept="image/gif,image/png,image/jpg" @change="handleFileUpload"/>
    <img  ref="uploadImg" id="uploadImg" :src="imgURL" :style="{ transform: `scale(${scale})` }"  alt="" />
    <input type="range" v-model="scale" min="0" max="1" step="0.1">
    <button @click="mountFile">开始渲染</button>


  </div>
</template>

<style scoped lang="scss">

.box {
  width: 100%;
  display: flex;
  flex-direction: column;
  align-items: center;

}
#uploadImg{
  max-width: 100vw;
}
</style>
