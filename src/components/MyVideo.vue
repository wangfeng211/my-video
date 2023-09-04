<template>
  <div class="my-player" @mouseout="mouseOut" @mousemove="mouseMove">
    <!-- 不要使用css控制canvas的宽高 应该在绘制的时候设置宽高，画布和canvas宽高要一致，否则会出现视频模糊或者显示不完全的情况 -->
    <canvas id="canvas" @click="onPlay"></canvas>

    <div
      v-show="data.showControls"
      @click="onPlay"
      class="play-icon-big"
      :style="{
        background: `url(${
          data.isPlaying ? playIconBig : pauseIconBig
        }) no-repeat center / 100% 100%`,
      }"
    ></div>
    <div class="controls" v-show="data.showControls">
      <div
        @click="onPlay"
        class="play-btn"
        :style="{
          background: `url(${
            data.isPlaying ? playIconSmall : pauseIconSmall
          }) no-repeat center / 100% 100%`,
        }"
      ></div>
      <div
        @click="onMuted"
        class="volumn"
        :style="{
          background: `url(${
            data.isMuted ? valumnIconMuted : valumnIconUnMuted
          }) no-repeat left / 100% 100%`,
        }"
      ></div>

      <div class="progress-line" @click="timeLineClick($event)">
        <div class="progress-point"></div>
      </div>
      <span class="time" v-if="data.showTime"
        >{{ data.currentTimeStr }} / {{ data.durationStr }}</span
      >
    </div>
  </div>
</template>

<script setup>
import { reactive, defineProps, onMounted, onBeforeUnmount, watch } from "vue";
import playIconBig from "../assets/ic_play_big.svg";
import pauseIconBig from "../assets/ic_pause_big.svg";
import playIconSmall from "../assets/ic_play_small.svg";
import pauseIconSmall from "../assets/ic_pause_small.svg";
import valumnIconMuted from "../assets/ic_valumn_muted.svg";
import valumnIconUnMuted from "../assets/ic_valumn_un_muted.svg";
import { jsx } from "vue/jsx-runtime";

const props = defineProps({
  src: {
    type: String,
    required: true,
  },
  poster: {
    type: String,
  },
  parent: {
    type: String,
    default: "#app",
  },
  volume: {
    type: Number,
  },
});

const data = reactive({
  isMuted: true, //是否静音
  isPlaying: false, //是否正在播放
  showControls: false, //是否展示控制条
  timer: "",
  duration: "",
  currentTime: "",
  durationStr: "00:00",
  currentTimeStr: "00:00",
  showTime: false, //是否展示时间
});
let canvas = "";
let ctx = "";
let player = "";
let timePositionEle = ""; //进度条 小圆点

onMounted(() => {
  player = document.createElement("video");

  player.src = props.src;
  if (props.poster) {
    player.poster = props.poster;
  }
  player.muted = true;
  player.autoplay = true;
  player.controls = true;
  player.loop = true;
  player.style.position = "absolute";
  player.style.right = "9000px";
  player.style.top = "-9000px";
  player.style.zIndex = -1;
  player.width = "880px";
  player.height = "384px";
  player.style.display = "none";
  if (props.volume) {
    player.volume = props.volume;
  }
  player.play();
  data.isPlaying = true;

  // 获取视频时常
  const audio = new Audio(props.src);
  audio.addEventListener("loadedmetadata", (_e) => {
    data.duration = audio.duration;
    data.durationStr = parserDuration(audio.duration);
    data.showTime = true;
  });
  //获取当前视频播放时长
  player.addEventListener("timeupdate", function () {
    data.currentTimeStr = parserDuration(data.currentTime);
    data.currentTime = this.currentTime;
    updateCurrentTimePosition();
  });
  data.showControls = true;
  data.timer = setTimeout(() => {
    data.showControls = false;
  }, 2000);
  document.querySelector(props.parent).appendChild(player);

  canvas = document.getElementById("canvas");
  // 864, 384
  canvas.width = 864;
  canvas.height = 384;
  ctx = canvas.getContext("2d");
  render();
});

onBeforeUnmount(() => {
  if (player) {
    player.pause();
    document.querySelector(props.parent).removeChild(player);
  }
});

watch(
  () => props.volume,
  (newVal, oldVal) => {
    console.log(newVal);
    player.volume = newVal;
  }
);

const onPlay = () => {
  if (player.paused) {
    player.play();
  } else {
    player.pause();
    data.showControls = true;
  }
  data.isPlaying = !player.paused;
};
//播放
const play = () => {
  player.play();
  data.isPlaying = true;
};
// 暂停
const pause = () => {
  player.pause();
  data.showControls = true;
  data.isPlaying = false;
};
//静音 取消静音
const onMuted = () => {
  const muted = player.muted;
  if (muted) {
    player.muted = false;
  } else {
    player.muted = true;
  }
  data.isMuted = player.muted;
};

const render = () => {
  window.requestAnimationFrame(render);
  ctx.clearRect(0, 0, canvas.width, canvas.height);
  ctx.drawImage(player, 0, 0, 864, 384); //绘制视频
};
const mouseOut = () => {
  if (data.showControls) {
    clearTimeout(data.timer);
    data.timer = null;
    data.timer = setTimeout(() => {
      data.showControls = false;
    }, 1000);
  }
};
const mouseMove = () => {
  data.showControls = true;
  if (data.timer) {
    clearTimeout(data.timer);
    data.timer = null;
  }
  data.timer = setTimeout(() => {
    data.showControls = false;
  }, 1000);
};

//格式化视频时长
const parserDuration = (duration) => {
  if (!duration) {
    return "";
  }
  const seconds = Math.floor(duration % 60);
  let minutes = Math.floor(duration / 60);
  let hours = "";
  if (minutes > 60) {
    hours = Math.floor(minutes / 60);
    minutes = minutes - hours * 60;
  }

  return `${hours ? hours + ":" : ""}${
    minutes < 10 ? "0" + minutes : minutes
  }:${seconds < 10 ? "0" + seconds : seconds}`;
};
//更新进度条 UI
const updateCurrentTimePosition = () => {
  timePositionEle = document.querySelector(".progress-point");
  timePositionEle.style.left =
    (data.currentTime / data.duration) * (580 - 10) + "px";
};

//进度条点击 更新视频位置和小圆点位置
const timeLineClick = (e) => {
  const position = e.pageX - e.target.offsetLeft;
  //   console.log(
  //     `pageX:${e.pageX}---target.offsetLeft:${e.target.offsetLeft}---position:${position}`
  //   );
  data.currentTime = (position / 580) * data.duration;
  player.currentTime = data.currentTime;
  timePositionEle.style.left = position + "px";
};
</script>

<style scoped>
.my-player {
  position: relative;
}
#canvas {
  margin: 8px;
  border-radius: 18px;
}
/* 播放按钮 1 */
.play-icon-big {
  position: absolute;
  z-index: 11;
  top: 50%;
  left: 50%;
  width: 62px;
  height: 62px;
  transform: translate(-50%, -50%);
}
/* 控制器 */
.controls {
  display: flex;
  justify-content: flex-start;
  align-items: center;
  position: absolute;
  bottom: 20px;
  z-index: 10;
  padding-left: 32px;
  padding-right: 0;
}
/* 播放、暂停 */
.play-btn {
  width: 40px;
  height: 40px;
}
/* 静音 */
.volumn {
  width: 40px;
  height: 40px;
  margin-left: 10px;
  margin-right: 10px;
}
/* 进度条 */
.progress-line {
  position: relative;
  height: 5px;
  width: 580px;
  border-radius: 2.5px;
  background-color: #92989e;
  margin-right: 24px;
}
/* 进度条小圆点 */
.progress-point {
  position: absolute;
  z-index: 11;
  left: 0;
  top: 50%;
  transform: translateY(-50%);
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background-color: #e0e6ec;
}
.time {
  color: #e0e6ec;
  font-size: 14px;
}
</style>
