<template>
  <div>
    <uploader v-if="false" />

    <div class="content-box">
      <div class="title">视频转换成 GIF</div>

      <div class="form-box">
        <div class="flex-row padding">
          <div class="label">视频链接：</div>
          <el-input class="w440" type="text" v-model="options.video" placeholder="请输入以https开头的视频链接" clearable maxlength="200" />
        </div>

        <div class="flex-row padding">
          <div class="flex-row">
            <div class="label">宽度：</div>
            <el-input class="w172" type="text" v-model="options.gifWidth" placeholder="宽度，如：800" maxlength="4">
              <template slot="append">px</template>
            </el-input>
          </div>
          <div class="flex-row l35">
            <div class="label">高度：</div>
            <el-input class="w172" type="text" v-model="options.gifHeight" placeholder="高度，如：600" maxlength="4">
              <template slot="append">px</template>
            </el-input>
          </div>
        </div>

        <div class="flex-row">
          <div class="label">GIF 时长：</div>
          <el-input class="w172" type="text" v-model="options.numFrames" placeholder="秒" maxlength="4">
              <template slot="append">秒</template>
            </el-input>
        </div>
        <div class="tip">
          等待时间取决于网速和电脑性能
          <span v-if="+options.numFrames !== 0">，预计等待 {{ options.numFrames }} ~ {{ 3 * options.numFrames }} 秒</span>
        </div>
        <div class="duration" v-if="duration">
          结果：已进行 {{ (duration / 1000).toFixed(1) }} 秒
          <span v-if="fileSize">，生成的 GIF 文件大小：{{Math.round((fileSize / 1024 / 1024) * 100) / 100}}M</span>
        </div>
      </div>

      <div class="btn-box">
        <el-button @click="reset">重 置</el-button>
        <el-button type="success" @click="download" v-if="base64">下 载</el-button>
        <el-button type="primary" :disabled="doneDisabled" @click="convert" v-else>转 换</el-button>
      </div>

      <video-to-gif :options="optionsDone" @save="save" ref="gifRef" />
    </div>
  </div>
</template>

<script>
import VideoToGif from '@/components/VideoToGif'
import Uploader from '@/components/Uploader'
import downloadFileByBase64 from '@/utils/downloadFileByBase64'
import { clearInterval } from 'timers';

export default {
  name: 'Home',
  data() {
    return {
      options: {
        video: 'https://liuxy0551.github.io/assets/gif.mp4',
        gifWidth: 540,
        gifHeight: 400,
        numFrames: 3
      },
      optionsTemp: null,
      optionsDone: null,
      loading: null,
      base64: null,
      duration: 0,
      fileSize: 0,
      timer: null
    }
  },
  watch: {
    'options.video': {
      handler(nv, ov) {
        localStorage.setItem('video', nv)
      }
    },
    'options.gifWidth': {
      handler(nv, ov) {
        localStorage.setItem('gifWidth', nv)
      }
    },
    'options.gifHeight': {
      handler(nv, ov) {
        localStorage.setItem('gifHeight', nv)
      }
    },
    'options.numFrames': {
      handler(nv, ov) {
        if (nv < 0) {
          this.options.numFrames = '0'
        } else {
          this.options.numFrames = nv.slice(0, 3)
        }
        localStorage.setItem('numFrames', this.options.numFrames)
      }
    }
  },
  computed: {
    doneDisabled() {
      if (!this.options.video) {
        return true
      }
      if (!this.options.gifWidth) {
        return true
      }
      if (!this.options.gifHeight) {
        return true
      }
      if (!this.options.numFrames) {
        return true
      }
      return false
    }
  },
  methods: {
    // 重置按钮 - 清除缓存
    reset() {
      this.options = Object.assign({}, this.optionsTemp)
      this.$refs['gifRef'].url = ''
      this.base64 = null
      this.duration = 0
      this.fileSize = 0
      localStorage.clear()
    },
    // 停止计时器
    stopTimer() {
      window.clearInterval(this.timer)
      this.timer = null
    },
    // 确认按钮
    convert() {
      this.loading = this.$loading({
        lock: true,
        text: '转换中，请稍等...',
        customClass: 'loading-icon',
        background: 'rgba(0, 0, 0, 0.6)'
      })
      this.optionsDone = Object.assign({}, this.options, { video: [this.options.video], numFrames: this.options.numFrames * 10 })

      const startTime = new Date().getTime()
      this.timer = window.setInterval(() => {
        this.duration = new Date().getTime() - startTime
      }, 100)
      this.$refs['gifRef'].createGIF(this.optionsDone, this.loading)
    },
    // 暂存 gif 的 base64
    save(base64) {
      this.stopTimer()
      this.base64 = base64

      // 文件大小
      var eqTagIndex = base64.indexOf('=')
      let file = eqTagIndex === -1 ? base64 : base64.substring(0, eqTagIndex)
      var strLen = file.length
      var fileSize = strLen - (strLen / 8) * 2
      this.fileSize = fileSize
      // this.$message.success(`生成的 GIF 文件大小：${Math.round((this.fileSize / 1024 / 1024) * 100) / 100}M`)

      // 耗时
      this.$notify({
        title: '转换成功',
        message: `耗时 ${this.duration} ms`,
        type: 'success'
      })
      this.loading.close()
    },
    // 下载按钮
    download() {
      let list = this.options.video.split('/')
      downloadFileByBase64(this.base64, list[list.length - 1].split('.')[0])
    }
  },
  mounted() {
    this.optionsTemp = { ...this.options }

    let list = ['video', 'gifWidth', 'gifHeight', 'numFrames']
    for (let i of list) {
      localStorage.getItem(i) && (this.options[i] = localStorage.getItem(i))
    }
  },
  components: { VideoToGif, Uploader }
}
</script>

<style lang="scss" scoped>
.content-box {
  display: flex;
  flex-direction: column;
  align-items: center;
  .title {
    font-size: 30px;
    font-weight: 700;
    padding: 40px 0;
  }

  .form-box {
    .flex-row {
      display: flex;
      align-items: center;
      &.padding {
        padding-bottom: 20px;
      }
      .label {
        width: 80px;
        text-align: left;
      }
      .w440 {
        width: 460px;
      }
      .w172 {
        width: 172px;
      }
      .l35 {
        margin-left: 35px;
      }
    }
  }

  .tip {
    font-size: 14px;
    opacity: 0.7;
    text-align: left;
    margin: 10px 0 0 80px;
  }
  .duration {
    font-size: 14px;
    text-align: left;
    margin: 10px 0 0 80px;
  }

  .btn-box {
    width: 340px;
    margin: 24px 0 24px -40px;
    display: flex;
    justify-content: space-between;
  }
}
</style>

<style lang="scss">
.loading-icon {
  .el-loading-spinner i {
    color: #fff;
    font-size: 30px;
    padding-bottom: 5px;
  }
  .path {
    stroke: #fff;
  }
  .el-loading-text {
    color: #fff;
    font-size: 24px;
    padding-top: 10px;
  }
}
</style>
