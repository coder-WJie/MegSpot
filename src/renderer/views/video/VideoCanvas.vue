<template>
  <div
    class="video-canvas"
    flex="dir:top box:first"
    :class="{ selected: selected }"
  >
    <div class="header" flex="cross:center">
      <CoverMask :mask="maskDom">
        <HistContainer
          ref="hist-container"
          @changeVisible="handleHistVisible"
        />
      </CoverMask>
      <el-tooltip placement="bottom">
        <span
          class="compare-name"
          flex-box="1"
          v-html="
            (selected ? `<span style='color: red'>(✔)</span>` : ``) +
              $options.filters.getFileName(videoSrc)
          "
        ></span>
        <div slot="content">
          {{ videoSrc }}
          <br /><br />
          <span class="size">
            {{ bitMap && bitMap.width }} x {{ bitMap && bitMap.height }}</span
          >
        </div>
      </el-tooltip>
      <RGBAExhibit
        :RGBAcolor="RGBAcolor"
        v-model="radius"
        @update="changeRadius"
      ></RGBAExhibit>
      <EffectPreview @change="changeCanvasStyle" />
    </div>
    <OperationContainer
      id="canvas-container"
      ref="canvas-container"
      @drag="handleDrag"
      @zoom="handleZoom"
      @scrollEnd="handleZoomEnd"
      @dbclick="handleDbclick"
      @mouseMove="handleMove"
    >
      <div ref="canvas-item" class="canvas-item" @contextmenu.prevent>
        <ScaleEditor
          class="scale-editor"
          :scale="imgScale"
          :scaleEditorVisible.sync="scaleEditorVisible"
          @show="showScaleEditor"
          @reset="resetZoom"
          @update="setZoom"
        />
        <canvas ref="canvasDom" :width="width" :height="height"> </canvas>
        <div ref="feedback" id="feedback" v-show="traggerRGB"></div>
      </div>
    </OperationContainer>
  </div>
</template>
<script>
import Vue from 'vue';
const cv = Vue.prototype.$cv;
import OperationContainer from '@/components/operation-container';
import HistContainer from '@/components/hist-container';
import CoverMask from '@/components/cover-mask';
import RGBAExhibit from '@/components/rgba-exhibit';
import ScaleEditor from '@/components/scale-editor';
import EffectPreview from '@/components/effect-preview';
import { createNamespacedHelpers } from 'vuex';
const { mapGetters, mapActions } = createNamespacedHelpers('videoStore');
import { getImageUrlSync } from '@/utils/image';
import { throttle } from '@/utils';
import { SCALE_CONSTANTS, DRAG_CONSTANTS } from '@/constants';

export default {
  components: {
    OperationContainer,
    HistContainer,
    CoverMask,
    RGBAExhibit,
    ScaleEditor,
    EffectPreview
  },
  props: {
    videoSrc: {
      type: String,
      default: ''
    },
    video: HTMLVideoElement,
    width: {
      type: Number,
      default: 500
    },
    height: {
      type: Number,
      default: 500
    }
  },
  data() {
    return {
      canvas: null,
      maskDom: undefined,
      currentHist: undefined,
      histImage: null,
      cs: null,
      image: {
        width: 0,
        height: 0
      },
      bitMap: null,
      imagePosition: null,
      cachedPositionData: null,
      imgScale: 'N/A',
      scaleEditorVisible: false,
      afterFullSize: false,
      selected: false,
      selectedId: null,
      // 原图的mat 缓存用于便捷获取
      syncCanvasActions: [
        {
          event: 'imageCenter_resetCanvas',
          action: 'reset'
        },
        {
          event: 'imageCenter_rotate',
          action: 'rotate'
        },
        {
          event: 'imageCenter_reverse',
          action: 'reverse'
        }
      ],
      // 调度事件
      scheduleCanvasActions: [
        {
          event: 'image_broadcast',
          action: 'handleBroadcast'
        },
        {
          event: 'image_handleSelect',
          action: 'handleSelect'
        },
        {
          event: 'videoCenter_getSelectedPosition',
          action: 'getSelectedPosition'
        },
        {
          event: 'videoCenter_align',
          action: 'align'
        }
      ],
      histVisible: true,
      traggerRGB: false,
      radius: 10,
      RGBAcolor: {
        R: 0,
        G: 0,
        B: 0,
        A: 0
      }
    };
  },
  mounted() {
    this.canvas = this.$refs.canvasDom;
    this.initCanvas();
    this.initImage();
    this.listenEvents();
  },
  beforeDestroy() {
    this.removeEvents();
    this.bitMap.close();
  },
  computed: {
    ...mapGetters(['videoList', 'videoConfig', 'selectedPosition']),
    canvasStyle() {
      let filter = '';
      ['brightness', 'contrast', 'saturate', 'grayscale', 'opacity'].forEach(
        item => {
          filter += `${item}(${this[item]}%) `;
        }
      );
      ['blur'].forEach(item => {
        filter += `${item}(${this[item]}px) `;
      });
      return filter;
    }
  },
  watch: {
    'videoConfig.smooth': {
      handler(newVal, oldVal) {
        this.setSmooth();
      },
      immediate: true
    }
  },
  methods: {
    ...mapActions(['setCanvasSize']),
    // 检查边界， 保证图像至少部分在canvas内(显示大小至少为当前图像大小的DRAG_CONSTANTS)
    checkBorder(transX, transY, width, height) {
      const cw = this.width,
        ch = this.height,
        iw = width ?? this.imagePosition.width,
        ih = height ?? this.imagePosition.height;
      const constantsW = DRAG_CONSTANTS * iw,
        constantsH = DRAG_CONSTANTS * ih;
      let isFullFilled =
        transX <= constantsW &&
        transY <= constantsH &&
        transX + iw >= cw &&
        transY + ih >= ch; // 判断图像是否占据整个canvas
      if (isFullFilled) return true;
      let isTooLeft = transX + iw < constantsW; // 图像是否过于偏左
      let isTooRight = transX > cw - constantsW; // 图像是否过于偏右
      let isTooTop = transY + ih < constantsH; // 图像是否过于偏上
      let isTooBottom = transY > ch - constantsH; // 图像是否过于偏下
      return !(isTooLeft || isTooRight || isTooTop || isTooBottom);
    },
    // 约束缩放，保证图像的宽(或高)不小于canvas宽(或高)的SCALE_CONSTANTS
    checkSize(transW, transH) {
      let isTooSmall =
        transW < this.canvas.width * SCALE_CONSTANTS ||
        transH < this.canvas.height * SCALE_CONSTANTS;
      if (this.afterFullSize && !isTooSmall) {
        this.afterFullSize = false;
      }
      return this.afterFullSize || !isTooSmall;
    },
    resetImageStyle() {
      // 预览处理效果
      this.brightness = 100;
      this.contrast = 100;
      this.saturate = 100;
      this.grayscale = 0;
      this.opacity = 100;
      this.blur = 0;
    },
    changeCanvasStyle(newStyle) {
      // FIXME: 具体的值没有被广播
      this.broadCast({
        name: 'setCanvasStyle',
        data: { style: newStyle }
      });
    },
    setCanvasStyle({ style }) {
      this.canvas.style.filter = style;
    },
    setSmooth() {
      if (this.cs) {
        this.cs.imageSmoothingEnabled = this.videoConfig.smooth;
        this.reDraw();
      }
    },
    // 提供外部直接调用
    getSnapshot() {
      // 叠加显示时候 生成快照
      return {
        snapShot: this.canvas,
        hist: this.currentHist
      };
    },
    listenEvents() {
      this.scheduleCanvasActions.forEach(item => {
        this.$bus.$on(item.event, this[item.action]);
      });
      // 分发事件 执行各个子组件的方法 同步状态
      this.syncCanvasActions.forEach(item => {
        this.$bus.$on(item.event, ({ name, data }) => {
          this.dispatchCanvasAction({ name, data });
        });
      });
      // 鼠标点击时隐藏编辑scale的输入框
      this.$refs['canvas-item'].addEventListener(
        'click',
        () => {
          this.scaleEditorVisible = false;
        },
        false
      );
    },
    removeEvents() {
      this.scheduleCanvasActions.forEach(item => {
        this.$bus.$off(item.event, this[item.action]);
      });
      // 分发事件 执行各个子组件的方法 同步状态
      this.syncCanvasActions.forEach(item => {
        this.$bus.$off(item.event, this[item.action]);
      });
    },
    dispatchCanvasAction({ name, data }) {
      if (!this.selectedId || this.selectedId === this.videoSrc) {
        this[name](data);
      }
    },
    handleBroadcast({ name, data }) {
      if (this.selectedId) {
        if (data.id === this.videoSrc || this.selectedId === this.videoSrc) {
          this[name](data);
        }
      } else {
        this[name](data);
      }
    },
    async initImage() {
      let offsreen = new OffscreenCanvas(
        this.video.videoWidth,
        this.video.videoHeight
      );
      let offCtx = offsreen.getContext('2d');
      offCtx.drawImage(this.video, 0, 0);
      this.bitMap = await offsreen.transferToImageBitmap();
      this.imagePosition = this.getImageInitPos(this.canvas, this.bitMap);
      this.doZoomEnd();
      this.drawImage();

      // 生成hist直方图 需要一个和video一样大的canvas作为入参
      const histCanvas = document.createElement('canvas');
      histCanvas.width = this.video.videoWidth;
      histCanvas.height = this.video.videoHeight;
      let histCanvasCtx = histCanvas.getContext('2d');
      histCanvasCtx.drawImage(this.video, 0, 0);
      this.currentHist = this.$refs['hist-container'].generateHist(
        cv.imread(histCanvas)
      );
    },
    initCanvas() {
      this.cs = this.canvas.getContext('2d');
      this.$nextTick(() => {
        this.cs.imageSmoothingEnabled = this.videoConfig.smooth;
        // 设置生成gif图的宽高
        this.setCanvasSize({
          width: this.canvas.width,
          height: this.canvas.height
        });
      });
    },
    // 供外部直接调用 待测试
    reMount() {
      this.initCanvas();
      this.image.onload = () => {
        this.imagePosition = this.getImageInitPos(this.canvas, this.bitMap);
        this.drawImage();
      };
      this.image.src = getImageUrlSync(this.videoSrc);
    },
    drawImage() {
      let { x, y, width, height } = this.imagePosition;
      this.cs.clearRect(0, 0, this.canvas.width, this.canvas.height);
      this.cs.drawImage(
        this.bitMap,
        x, //x
        y, //y
        width,
        height
      );
    },
    handleDbclick() {
      this.$bus.$emit(
        'image_handleSelect',
        this.selected ? null : this.videoSrc
      );
    },
    handleSelect(selectedId) {
      if (this.videoSrc === selectedId) {
        this.selected = true;
      } else {
        this.selected = false;
      }
      this.selectedId = selectedId;
    },
    handleHistVisible(visible) {
      this.broadCast({
        name: 'doHistVisible',
        data: { visible }
      });
    },
    doHistVisible({ visible }) {
      this.$refs['hist-container'].setVisible(visible);
    },
    pickColor({ status }) {
      this.traggerRGB = status;
    },
    changeRadius(radius) {
      this.broadCast({
        name: 'doChangeRadius',
        data: { radius }
      });
    },
    doChangeRadius({ radius }) {
      this.radius = radius;
    },
    handleMove: throttle(40, function(mousePos) {
      this.broadCast({
        name: 'doHandleMove',
        data: { mousePos }
      });
    }),
    doHandleMove({ mousePos }) {
      if (!this.traggerRGB) return;
      const { x, y } = mousePos;
      const feedback = this.$refs.feedback;
      feedback.style.left = x - this.radius + 'px';
      feedback.style.top = y - this.radius + 'px';
      feedback.style.width = this.radius * 2 + 'px';
      feedback.style.height = this.radius * 2 + 'px';
      Promise.resolve().then(() => {
        const data = this.cs.getImageData(x, y, this.radius, this.radius).data;
        const count = data.length / 4;
        let r = 0,
          g = 0,
          b = 0,
          a = 0;
        for (var i = 0; i < count; i++) {
          r += data[i * 4];
          g += data[i * 4 + 1];
          b += data[i * 4 + 2];
          a += data[i * 4 + 3];
        }
        this.RGBAcolor = {
          R: parseInt(r / count),
          G: parseInt(g / count),
          B: parseInt(b / count),
          A: parseInt(a / count)
        };
      });
    },
    // 外部直接调用
    setCoverStatus({ snapShot, hist }, status) {
      if (status) {
        this.cs.clearRect(0, 0, this.canvas.width, this.canvas.height);
        this.cs.drawImage(snapShot, 0, 0);
        if (this.$refs['hist-container'].visible) {
          this.maskDom = hist;
        }
      } else {
        this.maskDom = null;
        this.drawImage();
      }
    },
    reset(val) {
      if (val) {
        let x = 0,
          y = 0;
        let width = this.image.naturalWidth;
        let height = this.image.naturalHeight;
        this.afterFullSize = true;
        this.imagePosition = { x, y, width, height };
        this.doZoomEnd();
        this.drawImage();
      } else {
        this.initImage();
      }
    },
    reDraw() {
      window.requestAnimationFrame(this.drawImage);
    },
    getImageInitPos(canvas, image) {
      const cw = this.width;
      const ch = this.height;

      const iw = image.width;
      const ih = image.height;

      const canvasRadio = cw / ch;
      const imageRadio = iw / ih;

      let x = 0;
      let y = 0;
      let height = ch;
      let width = cw;

      if (canvasRadio > imageRadio) {
        //比较高，所以高占100%,宽居中
        width = ch * imageRadio;
      } else {
        //比较宽，所以宽占100%,高居中
        height = cw / imageRadio;
      }
      return {
        x,
        y,
        width,
        height
      };
    },
    doDrag(data) {
      let offset = data.offset;
      let transX = this.imagePosition.x + offset.x;
      let transY = this.imagePosition.y + offset.y;
      if (this.checkBorder(transX, transY)) {
        // 判断是否只在指定范围内拖动
        this.imagePosition.x = transX;
        this.imagePosition.y = transY;
        this.drawImage();
      }
    },
    doZoom(data) {
      let mousePos = data.mousePos;
      let rate = data.rate;
      let x = mousePos.x - (mousePos.x - this.imagePosition.x) * rate;
      let y = mousePos.y - (mousePos.y - this.imagePosition.y) * rate;
      let height = this.imagePosition.height * rate;
      let width = this.imagePosition.width * rate;
      if (
        (rate > 1 || this.checkSize(width, height)) &&
        this.checkBorder(x, y, width, height)
      ) {
        this.imagePosition = {
          x,
          y,
          height,
          width
        };
        this.imgScale = 'N/A';
        this.drawImage();
      }
    },
    doZoomEnd() {
      this.imgScale = Number(
        this.imagePosition.width / this.bitMap.width
      ).toFixed(2);
    },
    handleDrag(offset) {
      this.broadCast({
        name: 'doDrag',
        data: { offset: offset, id: this.videoSrc }
      });
    },
    // 判断图像是否占据整个canvas
    isFullFilled() {
      const cw = this.canvas.width,
        ch = this.canvas.height,
        iw = this.imagePosition.width,
        ih = this.imagePosition.height;
      const constantsW = DRAG_CONSTANTS * iw,
        constantsH = DRAG_CONSTANTS * ih;
      const { x: transX, y: transY } = this.imagePosition;
      return (
        transX <= constantsW &&
        transY <= constantsH &&
        transX + iw >= cw &&
        transY + ih >= ch
      );
    },
    /******************ScaleEditor START******************/
    showScaleEditor() {
      this.scaleEditorVisible = true;
      this.cachedPositionData = {
        scale: this.imgScale,
        imagePosition: { ...this.imagePosition }
      };
    },
    resetZoom() {
      const { scale, imagePosition } = this.cachedPositionData;
      this.imagePosition = imagePosition;
      this.cachedPositionData = {
        scale: scale,
        imagePosition: { ...imagePosition }
      };
      this.imgScale = scale;
      this.drawImage();
    },
    setZoom(scale) {
      if (scale == this.imgScale) {
        this.imgScale = scale; // 必需, 触发scaleEditor组件更新
        return;
      }
      const oldScale = this.imgScale ?? scale;
      const scaleRatio = scale / oldScale;
      // 从画布中心放大
      const position = { x: this.canvas.width / 2, y: this.canvas.height / 2 };
      this.doZoom({
        rate: scaleRatio,
        mousePos: position
      });
      this.imgScale = scale;
    },
    /******************ScaleEditor END******************/
    handleZoom(rate, mousePos) {
      this.broadCast({
        name: 'doZoom',
        data: { rate: rate, mousePos: mousePos, id: this.videoSrc }
      });
    },
    handleZoomEnd() {
      this.broadCast({
        name: 'doZoomEnd',
        data: {}
      });
    },
    broadCast({ name, data }) {
      if (!this.selected) {
        throttle(16, () => {
          this.$bus.$emit('image_broadcast', {
            name: name,
            data: data
          });
        })();
      } else {
        this[name](data);
      }
    },
    rotate(degree) {
      let offscreenWidth = this.bitMap.height;
      let offscreenHight = this.bitMap.width;
      let offsreen = new OffscreenCanvas(offscreenWidth, offscreenHight);
      let offCtx = offsreen.getContext('2d');
      if (degree < 0) {
        offCtx.translate(0, this.bitMap.width);
        offCtx.rotate((-90 * Math.PI) / 180);
        [this.imagePosition.width, this.imagePosition.height] = [
          this.imagePosition.height,
          this.imagePosition.width
        ];
      } else if (degree > 0) {
        offCtx.translate(this.bitMap.height, 0);
        offCtx.rotate((90 * Math.PI) / 180);
        [this.imagePosition.width, this.imagePosition.height] = [
          this.imagePosition.height,
          this.imagePosition.width
        ];
      }
      offCtx.drawImage(this.bitMap, 0, 0);
      this.bitMap.close();
      this.bitMap = offsreen.transferToImageBitmap();
      this.drawImage();
    },
    reverse(direction) {
      let offscreenWidth = this.bitMap.width;
      let offscreenHight = this.bitMap.height;
      let offsreen = new OffscreenCanvas(offscreenWidth, offscreenHight);
      let offCtx = offsreen.getContext('2d');
      if (direction > 0) {
        //左右翻转
        offCtx.translate(this.bitMap.width, 0);
        offCtx.scale(-1, 1);
      } else if (direction < 0) {
        //上下翻转
        offCtx.translate(0, this.bitMap.height);
        offCtx.scale(1, -1);
      }
      offCtx.drawImage(this.bitMap, 0, 0);
      this.bitMap.close();
      this.bitMap = offsreen.transferToImageBitmap();
      this.drawImage(this.bitMap);
    },
    align({ name, data }) {
      const { beSameSize, position } = data;
      if (
        this.selectedId ||
        (!this.selectedId && this.videoSrc !== this.videoList[0])
      ) {
        if (beSameSize) {
          let { x, y, width, height } = position;
          this.imagePosition = { x, y, width, height };
          this.afterFullSize = true;
          this.doZoomEnd();
        } else {
          this.imagePosition.x = position.x;
          this.imagePosition.y = position.y;
        }
        this.drawImage();
      }
    },
    getSelectedPosition(data, callback) {
      if (
        this.selected ||
        this.videoSrc === this.selectedId ||
        (!this.selectedId && this.videoSrc === this.videoList[0])
      ) {
        callback({
          id: this.videoSrc,
          position: this.imagePosition
        });
      }
    }
  }
};
</script>
<style scoped lang="scss">
@import '@/styles/variables.scss';
.video-canvas {
  height: 100%;
  z-index: 1;
  .header {
    height: 18px;
    line-height: 16px;
    background-color: #f6f6f6;
  }
  #canvas-container {
    height: auto;
    width: 100%;
    .canvas-item {
      position: relative;
      width: 100%;
      height: 100%;
      ::v-deep {
        input,
        .el-input-group__append {
          border-radius: 0;
        }
        .el-input-group__append,
        .el-input-group__prepend {
          padding: 0 6px;
          &:hover {
            color: green;
          }
        }
        .el-input__inner {
          padding: 0 2px;
        }
      }
      canvas {
        background: #e3e7e9;
        background-image: linear-gradient(45deg, #f6fafc 25%, transparent 0),
          linear-gradient(45deg, transparent 75%, #f6fafc 0),
          linear-gradient(45deg, #f6fafc 25%, transparent 0),
          linear-gradient(45deg, transparent 75%, #f6fafc 0);
        background-position: 0 0, 10px 10px, 10px 10px, 20px 20px;
        background-size: 20px 20px;
        vertical-align: middle;
        font-size: 0;
      }
      img {
        object-fit: contain;
        vertical-align: middle;
      }
      #feedback {
        position: absolute;
        border: 1px solid red;
      }
      #preview {
        position: absolute;
        top: 0;
        right: 14px;
        z-index: 1;
        padding: 0;
        font-size: 17px;
      }
    }
  }
}
</style>
