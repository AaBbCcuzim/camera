<template>
  <div class="main">
    <el-row justify="center">
      <video
        data-tauri-drag-region
        ref="video"
        class="video"
        :style="{
          width: size + 'px',
          height: shape === 'circle' ? size + 'px' : (size * 9) / 16 + 'px',
        }"
        autoplay
        playsinline></video>
    </el-row>
    <el-row justify="center">
      <el-button-group class="button-group" size="small">
        <el-button text icon="FullScreen" color="#5842B5" @click="changFullScreen" />
        <el-button text icon="Switch" color="#5842B5" @click="changeShape" />
        <el-button text icon="Plus" color="#5842B5" @click="zoomIn" />
        <el-button text icon="Minus" color="#5842B5" @click="zoomOut" />
        <el-button text :icon="isTrace ? 'View' : 'Hide'" color="#5842B5" @click="traceFace" />
        <el-button text icon="More" color="#5842B5" />
      </el-button-group>
    </el-row>
    <!-- <el-row justify="center">
      <el-select
        v-model="currentDevice"
        placeholder="Select"
        size="large"
        style="width: 240px"
        @change="getCamera(currentDevice)"
      >
        <el-option
          v-for="item in devices"
          :key="item.deviceId"
          :label="item.label"
          :value="item.label"
        />
      </el-select>
    </el-row>  -->
  </div>
</template>

<script setup>
import { ref, onMounted, watch } from "vue";
import { appWindow, LogicalSize } from "@tauri-apps/api/window";
import * as faceapi from "face-api.js";

let video = ref(null);
let size = ref(200);
let currentDevice = ref();
let devices = ref([]);
let shape = ref("circle");
let isTrace = ref(false);
let traceTimer = null;

/**
 * 获取摄像头
 */
async function getCamera(label) {
  const deviceId = devices.value.find((item) => item.label === label).deviceId;

  if ("mediaDevices" in navigator && "getUserMedia" in navigator.mediaDevices) {
    const constraints = {
      video: {
        width: { ideal: 1920 },
        height: { ideal: 1080 },
        deviceId: deviceId,
      },
    };
    const videoStream = await navigator.mediaDevices.getUserMedia(constraints);
    video.value.srcObject = videoStream;
    video.value.play();
  }
}

/**
 * 获取摄像头列表
 */
async function getCameraList() {
  if ("mediaDevices" in navigator && "getUserMedia" in navigator.mediaDevices) {
    const deviceInfos = await navigator.mediaDevices.enumerateDevices();
    const videoDevices = deviceInfos.filter((deviceInfo) => deviceInfo.kind === "videoinput");
    devices.value = videoDevices;
    currentDevice.value = videoDevices[0].label;
  }
}

/**
 * 调整窗口大小
 */
async function reSize() {
  await appWindow.setSize(new LogicalSize(size.value, size.value + 30));
}

/**
 * 全屏切换
 */
async function changFullScreen() {
  const fullscreen = await appWindow.isFullscreen();

  if (!fullscreen) {
    appWindow.setFullscreen(true);
    video.value.style.borderRadius = "0";
    video.value.style.width = "100vw";
    video.value.style.height = "95vh";
  } else {
    setTimeout(() => {
      appWindow.setFullscreen(false);
    }, 200);
    video.value.style.borderRadius = "50%";
    video.value.style.width = size.value + "px";
    video.value.style.height = size.value + "px";
  }
}

/**
 * 窗口比例切换
 */
async function changeShape() {
  if (shape.value === "circle") {
    video.value.style.borderRadius = "0";
    video.value.style.width = size.value + "px";
    video.value.style.height = (size.value * 9) / 16 + "px";
    shape.value = "rectangle";
  } else {
    video.value.style.borderRadius = "50%";
    video.value.style.width = size.value + "px";
    video.value.style.height = size.value + "px";
    shape.value = "circle";
  }
}

/**
 * 放大窗口
 */
function zoomIn() {
  setTimeout(() => {
    if (size.value >= 500) return;
    size.value += 10;
  }, 200);
}

/**
 * 缩小窗口
 */
function zoomOut() {
  if (size.value <= 200) return;
  size.value -= 10;
}

/**
 * 跟踪人脸
 */
function traceFace() {
  isTrace.value = !isTrace.value;
}

/**
 * 初始化模型
 */
async function initModel() {
  try {
    await faceapi.nets.tinyFaceDetector.loadFromUri("/models");
    console.log("Model loaded successfully");
  } catch (error) {
    console.error("Failed to load model", error);
  }
}

/**
 * 检测人脸并更新 video 元素的位置
 */
async function detectFaces() {
  const detections = await faceapi.detectSingleFace(video.value, new faceapi.TinyFaceDetectorOptions());

  if (detections) {
    const { x, y, width, height } = detections.box;
    const faceCenterX = x + width / 2;
    const faceCenterY = y + height / 2;

    // 计算 object-position 的值
    const posX = ((faceCenterX - width / 2) / video.value.videoWidth) * 100;
    const posY = ((faceCenterY - height / 2) / video.value.videoHeight) * 100;

    // 更新 video 元素的 object-position 属性
    video.value.style.objectPosition = `${posX}% ${posY}%`;
  }
}

onMounted(async () => {
  await getCameraList();
  await getCamera(currentDevice.value);
  await initModel();
});

watch(size, async (oldValue, newValue) => {
  if (oldValue < newValue) {
    setTimeout(() => {
      reSize();
    }, 200);
  } else {
    reSize();
  }
});

watch(isTrace, (value) => {
  if (value) {
    traceTimer = setInterval(() => {
      detectFaces();
    }, 100);
  } else {
    clearInterval(traceTimer);
  }
});
</script>

<style scoped>
.main {
  width: 100vw;
  height: 100vh;
}

.main:hover .button-group {
  visibility: visible;
  opacity: 1;
}

.canvas {
  border-radius: 50%;
  box-shadow: 1px 1px 5px 1px rgba(23, 23, 23, 0.5);
}

.slider {
  width: 200px;
}

.video {
  object-fit: cover;
  border-radius: 50%;
  transition: 0.2s linear; /* 添加这一行 */
}

.button-group {
  background-color: rgba(255, 255, 255, 0.5);
  margin-top: 5px;
  visibility: hidden;
  opacity: 0;
  transition: visibility 0.2s, opacity 0.2s linear;
  background-color: rgb(146, 4, 255);
}
</style>
