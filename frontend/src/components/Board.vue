<script setup lang="ts">
import Canvas from "./Canvas.vue";
import { computed, ref, watch } from "vue";
import { useEyeDropper } from "@vueuse/core";
import { useAnimationPageWidth } from "@/utils/hooks";
import { useFullscreen } from "@vueuse/core";
import { toolbarOptions } from "@/enum/toolbar";
import { shapeOptions } from "@/enum/shape";
import { useRouter } from "vue-router";
import { StorageKey } from "@/store/state";
import { ElMessage } from "element-plus";
import store from "@/store";

const router = useRouter();
const uid = ref(sessionStorage.getItem(StorageKey.UID));
const canvas = ref();

// 全屏
const { isFullscreen, toggle } = useFullscreen();
// 着色器
const { isSupported, open, sRGBHex } = useEyeDropper();

let pageWidth = computed(() => store.state.pageWidth);
let isclick = computed(() => (pageWidth.value === 0 ? false : true));
let currentType = computed(() => store.state.currentType);
let toggleWidth = useAnimationPageWidth();

// 当前缩放值
let scale = ref(1);

// 是否悬浮形状
let isHover = ref(false);

let currpage = computed(() => store.state.page + 1);
let totalCount = computed(() => store.state.pageList.length);
let toolbarData = ref(toolbarOptions);
let shapeData = ref(shapeOptions);

const selectActionHandle = (type: string) => {
  store.commit("changeCurrentType", type);
  if (isSupported && type === "shaders") {
    open();
  }
};

const handleCopy = () => {
  let inputDom: any = document.createElement("input");
  inputDom.setAttribute("readonly", "readonly"); // 防止手机上弹出软键盘
  inputDom.value = sessionStorage.getItem(StorageKey.UID);
  document.body.appendChild(inputDom);
  inputDom.select();
  document.execCommand("Copy"); // 复制api
  inputDom.style.display = "none";
  inputDom.remove();
  ElMessage("房间号复制成功");
};

// 撤回
const withdraw = () => {
  canvas.value.tapHistoryBtn(-1, "withdraw");
};

// 还原
const reduction = () => {
  canvas.value.tapHistoryBtn(1, "reduction");
};

// 放大
const zoomIn = () => {
  if (scale.value >= 3) {
    scale.value = 3;
  } else scale.value += 0.1;
};

// 缩小
const zoomOut = () => {
  if (scale.value <= 0.1) {
    scale.value = 0.1;
  } else scale.value -= 0.1;
};
let timer: any;
const mouseMoveHandle = (type: string) => {
  if (["shape", "rectangle", "triangle", "circle"].includes(type)) {
    if (timer) {
      clearTimeout(timer);
    }
    isHover.value = true;
  }
};

const mouseLeaveHanle = () => {
  if (timer) {
    clearTimeout(timer);
  }
  timer = setTimeout(() => {
    isHover.value = false;
  }, 1000);
};

const changeShape = (type: string) => {
  const data = shapeData.value.find((item) => item.type === type);
  console.log(type, data);
  (toolbarData.value as any)[1] = {
    ...data,
    key: 2,
  };
  store.commit("changeCurrentType", type);
};

// 点击上传文件
const handleUpload = (e: any) => {
  canvas.value.setImage(e);
};

const clearSrceen = () => {
  canvas.value.clearSrceen();
};
</script>

<template>
  <div class="boardContainer">
    <Canvas :scale="scale" ref="canvas" />
    <div
      class="top-action"
      :style="{
        left: pageWidth + 'px',
      }"
    >
      <div class="top-action-item">
        <div class="iconfont icon-tuichu" @click="router.go(-1)"></div>
        <div class="info" @click="handleCopy">房间号：{{ uid }}</div>
        <div class="iconfont icon-fuzhi" @click="handleCopy"></div>
      </div>
      <div class="top-action-item">
        <div class="iconfont icon-huanyuan" @click="withdraw"></div>
        <div class="iconfont icon-huanyuan-01" @click="reduction"></div>
        <div class="iconfont icon-qingping" @click="clearSrceen"></div>
      </div>
    </div>
    <div class="right-show">
      <span> 着色器：</span>
      <div
        :style="{ background: sRGBHex ? sRGBHex : '#fff' }"
        class="right-show-item"
      ></div>
      <span class="text">{{ sRGBHex ? sRGBHex : "rgb(0,0,0)" }}</span>
    </div>
    <div
      class="toolbar"
      :style="{
        left: pageWidth + 'px',
      }"
    >
      <div
        class="toolbar-item"
        v-for="item in toolbarData"
        :key="item.key"
        @click="selectActionHandle(item.type)"
      >
        <el-tooltip
          class="box-item"
          :content="item.tips"
          placement="right-start"
        >
          <div
            :class="[item.iconClass, item.type === currentType ? 'active' : '']"
            @mousemove="mouseMoveHandle(item.type)"
            @mouseleave="mouseLeaveHanle"
          >
            <input
              type="file"
              class="input"
              @change="handleUpload"
              v-if="item.type === 'upload'"
              accept="image/*"
            />
          </div>
        </el-tooltip>
      </div>
    </div>
    <div
      class="shape-action"
      @mousemove="mouseMoveHandle('shape')"
      @mouseleave="mouseLeaveHanle"
      v-if="isHover"
      :style="{
        left: pageWidth + 70 + 'px',
      }"
    >
      <div
        class="shape-action-item"
        v-for="item in shapeData"
        @click="changeShape(item.type)"
        :key="item.key"
      >
        <el-tooltip
          class="box-item"
          :content="item.tips"
          placement="right-start"
        >
          <div
            :class="[item.iconClass, item.type === currentType ? 'active' : '']"
          ></div>
        </el-tooltip>
      </div>
    </div>
    <div
      class="left-action"
      :style="{
        left: pageWidth + 'px',
      }"
    >
      <div
        :class="['left-action-left', isclick ? 'selected' : '']"
        @click="toggleWidth"
      >
        {{ currpage }} / {{ totalCount }}
      </div>
      <!-- 拓展 -->
      <div class="left-action-right"></div>
    </div>
    <div class="right-action">
      <div class="right-action-items">
        <div
          class="right-action-item iconfont icon-jian"
          @click="zoomOut"
        ></div>
        <div class="right-action-item">
          {{ parseInt(String(100 * scale)) }}%
        </div>
        <div
          class="right-action-item iconfont icon-tianjia"
          @click="zoomIn"
        ></div>
        <div
          :class="[
            'right-action-item',
            'iconfont',
            isFullscreen ? 'icon-guanbiquanping' : 'icon-quanping',
          ]"
          @click="toggle"
        ></div>
      </div>
    </div>
  </div>
</template>

<style scoped lang="scss">
.boardContainer {
  background: #f5f8f9;
  position: relative;
  width: 100%;
  height: 100%;
  overflow: hidden;
  .icon-shangchuanwenjian {
    position: relative;
    overflow: hidden;
  }
  .input {
    z-index: 1;
    opacity: 0;
    left: 0;
    width: 100%;
    position: absolute;
  }
  .bgImg {
    width: 100%;
    height: 100%;
    overflow: hidden;

    .box-item {
      color: #fff;
    }

    .bgImg {
      width: 100%;
      height: 100%;
      position: absolute;
    }

    .canvasBox {
      position: relative;
    }
    .active {
      color: #3456ff;
    }
  }

  .toolbar {
    position: fixed;
    top: 90px;
    transition: left 0.5s linear;
    margin-left: 12px;
    background: #fff;
    padding: 10px;
    box-shadow: 0 2px 10px rgb(0 0 0 / 3%);
    &-item {
      margin: 4px 0;
      padding: 5px;
      border-radius: 4px;
    }
    &-item:hover {
      background: #f7f7f7;
    }
    .iconfont {
      font-size: 28px;
    }
    .active {
      color: #3456ff;
    }
  }
  .right-show {
    position: fixed;
    right: 12px;
    top: 6px;
    background: #fff;
    padding: 7px;
    display: flex;
    align-items: center;
    box-shadow: 0 2px 10px rgb(0 0 0 / 3%);
    &-item {
      width: 10px;
      height: 10px;
      border-radius: 100%;
      background-color: #fff;
      border: 1px solid #999;
    }
    .text {
      margin-left: 12px;
    }
  }
  .shape-action {
    position: fixed;
    top: 90px;
    transition: left 0.5s linear;
    margin-left: 12px;
    background: #fff;
    padding: 10px;
    box-shadow: 0 2px 10px rgb(0 0 0 / 3%);
    &-item {
      margin: 4px 0;
      padding: 5px;
      border-radius: 4px;
    }
    &-item:hover {
      background: #f7f7f7;
    }
    .iconfont {
      font-size: 28px;
    }
    .active {
      color: #3456ff;
    }
  }
  .top-action {
    position: fixed;
    top: 8px;
    transition: left 0.5s linear;
    display: flex;
    &-item {
      background: #fff;
      display: flex;
      padding: 10px 20px;
      align-items: center;
      margin-left: 12px;
      height: 20px;
      box-shadow: 0 2px 10px rgb(0 0 0 / 3%);
      cursor: pointer;
      .iconfont {
        font-size: 20px;
      }
      .icon-huanyuan-01 {
        font-weight: 600;
        margin-left: 20px;
      }
      .icon-qingping {
        margin-left: 20px;
      }
      .iconfont:hover {
        color: #3456ff;
      }
    }
    .icon-tuichu {
      font-size: 20px;
    }
    .info {
      margin-left: 10px;
    }
    .icon-xinxikongxin {
      margin-left: 10px;
    }
  }
  .left-action {
    position: fixed;
    bottom: 12px;
    transition: left 0.5s linear;
    &-left {
      background: #fff;
      padding: 10px;
      box-shadow: 0 2px 10px rgb(0 0 0 / 3%);

      &-item {
        margin: 4px 0;
        padding: 5px;
        border-radius: 4px;
      }

      &-item:hover {
        background: #f7f7f7;
      }

      .iconfont {
        font-size: 28px;
      }

      .active {
        color: #3456ff;
      }
    }

    .top-action {
      position: fixed;
      top: 8px;
      transition: left 0.5s linear;
      display: flex;

      &-left {
        background: #fff;
        display: flex;
        padding: 10px 20px;
        align-items: center;
        margin-left: 12px;
        height: 20px;
        box-shadow: 0 2px 10px rgb(0 0 0 / 3%);
        cursor: pointer;
      }

      .icon-tuichu {
        font-size: 20px;
      }

      .info {
        margin-left: 10px;
      }

      .icon-fuzhi {
        margin-left: 5px;
      }
    }

    .left-action {
      position: fixed;
      bottom: 12px;
      transition: left 0.5s linear;

      &-left {
        background: #fff;
        padding: 10px 20px;
        margin-left: 12px;
        text-align: center;
        width: 40px;
        height: 20px;
        box-shadow: 0 2px 10px rgb(0 0 0 / 3%);
        cursor: pointer;
      }

      .selected {
        color: #3456ff;
      }
    }
  }
  .right-action {
    position: fixed;
    bottom: 12px;
    right: 12px;
    background: #fff;
    padding: 6px 5px;
    box-shadow: 0 2px 10px rgb(0 0 0 / 3%);

    &-items {
      text-align: center;
      display: flex;

      .iconfont {
        font-size: 20px;
      }
    }

    &-item {
      padding: 5px 15px;
      cursor: pointer;
    }

    &-item:hover {
      background: #f6f6f6;
    }
  }
}
</style>
