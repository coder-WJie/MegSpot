<template>
  <div class="video-container" flex="dir:top">
    <div class="container" ref="container" flex-box="1" flex="dir:top">
      <div v-if="!videoImageVisiable" class="video-header" flex="cross:center">
        <el-tooltip placement="bottom" :open-delay="300">
          <template slot="content" style="float: left">
            {{ path }}
          </template>
          <span
            class="file-name"
            flex-box="1"
            v-html="$options.filters.getFileName(videoSrc)"
          ></span>
        </el-tooltip>
        <el-button
          type="text"
          @click="removeVideos(path)"
          v-tip="`close this item`"
        >
          <svg-icon icon-class="close" class="close"></svg-icon>
        </el-button>
      </div>
      <video
        class="video-style"
        preload
        :src="videoSrc"
        type="video/mp4"
        :loop="loop"
        ref="video"
        controls
        muted
        v-show="!videoImageVisiable"
        :disabled="!videoImageVisiable"
        @ended="handleEnd"
      >
        Your browser does not support the video tag.
      </video>
      <VideoCanvas
        ref="videoImage"
        v-if="videoImageVisiable"
        :video="video"
        :videoSrc="videoSrc"
        :width="container.clientWidth"
        :height="container.clientHeight - 22"
      ></VideoCanvas>
    </div>
  </div>
</template>
<script>
import * as CONSTANTS from './video-constants';
import VideoCanvas from './VideoCanvas.vue';
const UPDATE_VIDEO_PROGRESS = 'UPDATE_VIDEO_PROGRESS';
import { createNamespacedHelpers } from 'vuex';
const { mapActions } = createNamespacedHelpers('videoStore');

export default {
  name: 'VideoContainer',
  components: { VideoCanvas },
  props: ['path', 'loop', 'playbackRate', 'videoImageVisiable'],
  data() {
    return {
      // `https://oss.iap.hh-b.brainpp.cn/lilinyang/lovelive/test/video/R9%E6%9A%B4%E5%8A%9B%E6%B5%8B%E8%AF%95.m4v`
      video: undefined,
      container: undefined,
      currentTime: 0,
      // 调度事件
      scheduleCanvasActions: [
        {
          event: 'getVideo',
          action: 'getVideo'
        },
        {
          event: 'getVideoImage',
          action: 'getVideoImage'
        },
        {
          event: UPDATE_VIDEO_PROGRESS,
          action: 'syncSpeed'
        },
        {
          event: CONSTANTS.BUS_VIDEO_COMPARE_ACTION,
          action: 'executeAction'
        }
      ]
    };
  },
  computed: {
    videoSrc() {
      return `file://${this.path.replace(/#/g, '%23')}`;
    }
  },
  watch: {
    playbackRate(val) {
      this.video.playbackRate = val;
    }
  },
  methods: {
    ...mapActions(['removeVideos']),
    handleEnd(arg) {},
    play() {
      this.video.currentTime = this.currentTime;
      this.video.play();
    },
    // 提供外部直接调用
    getVideo({ name, data }, callback) {
      if (data === this.path) {
        const {
          clientWidth,
          clientHeight,
          videoWidth,
          videoHeight
        } = this.video;
        const video = { clientWidth, clientHeight, videoWidth, videoHeight };
        console.log(video);
        callback(video);
        return this.video;
      }
    },
    // 提供外部直接调用
    executeAction(action) {
      switch (action) {
        case CONSTANTS.VIDEO_STATUS_START:
          this.video.play();
          break;
        case CONSTANTS.VIDEO_STATUS_PAUSE:
          this.video.pause();
          break;
        case CONSTANTS.VIDEO_STATUS_RESET:
          this.video.currentTime = 0;
          break;
        default:
          console.error('unknown actions:' + action);
          break;
      }
    },
    // time单位毫秒  duration单位秒
    syncSpeed({ time, duration }) {
      if (Math.abs(this.video.duration - duration) < 0.5) {
        // 同步播放进度
        this.currentTime = time;
        this.video.currentTime = time;
      }
    },
    getVideoCanvas() {
      return this.$refs.videoImage;
    }
  },
  beforeDestroy() {
    this.scheduleCanvasActions.forEach(item => {
      this.$bus.$off(item.event, this[item.action]);
    });
  },
  mounted() {
    this.video = this.$refs.video;
    this.container = this.$refs.container;
    this.scheduleCanvasActions.forEach(item => {
      this.$bus.$on(item.event, this[item.action]);
    });
    this.video.addEventListener('timeupdate', evt => {
      // 监听视频播放过程中的时间
      if (Math.abs(this.video.currentTime - this.currentTime) > 1) {
        this.$bus.$emit(UPDATE_VIDEO_PROGRESS, {
          name: this.name,
          time: this.video.currentTime,
          duration: this.video.duration
        });
      } else {
        this.currentTime = this.video.currentTime;
      }
    });
  }
};
</script>
<style lang="scss" scoped>
@import '@/styles/variables.scss';
.video-container {
  height: 100%;
  .video-header {
    padding: 5px;
    font-size: 12px;
    color: gray;
    font-weight: bold;
    text-align: center;
    word-break: break-all;
    text-overflow: ellipsis;
    line-height: 100%;
    height: 22px;
    display: inline-flex;
    .svg-icon {
      font-size: 13px;
    }
    .file-name {
      margin: 0 10px;
      overflow: hidden;
      white-space: nowrap;
      text-align: center;
      direction: rtl;
      text-overflow: ellipsis;
      z-index: 999;
      font-size: 12px;
      color: gray;
      font-weight: bold;
      width: 0;
    }
    .close:hover {
      color: red;
    }
  }
  .container {
    overflow: hidden;
    position: relative;
    .video-style {
      max-width: 100%;
      max-height: 100%;
      object-fit: contain;
    }
  }
}
</style>
