<template>
  <div id="app">
    <header class="header">
      <button class="back-button">&lt;</button>
      <span class="header-text">视频音频模拟测试</span>
      <div class="audio-rec">
        <div class="rec">
          <div class="rec-border">
            <div class="rec-radius" v-show="isRecShow">
            </div>
          </div>
          <div class="rec-text">REC</div>
        </div>
        <div class="audio">
          <img class="audio-img" :src="audioImg" alt="">
        </div>
      </div>
    </header>
    <section class="body">
      <video class="video" src="" ref="video"></video>
      <img class="people-img" src="/people.png" alt="nothing">
      <div class="information">
        <div class="video-button">
          <button v-if="videoStatus === 'START'" @click="startRecord" class="startVideo">开始<br>录制</button>
          <button v-else-if="videoStatus === 'STOP'" @click="stopRecord" class="stopVideo">停止<br>录制</button>
          <button v-else-if="videoStatus === 'PLAY'" @click="startPlay" class="startVideo">播放<br>视频</button>
          <button v-else-if="videoStatus === 'PAUSE'" @click="pauseVideo" class="stopVideo">停止<br>播放</button>
        </div>
        <div class="text1">
          点击"开始录制"后请阅读以下文字：
        </div>
        <div class="text2">
          在整个考试过程中，不要移动身体，手部不要有动作，不能使用笔和纸，保持脸部在屏幕虚线框内，眼睛视线要始终直视屏幕，不回答问题时，嘴部不要有说话动作。
        </div>
      </div>
      <button :class="nextDisable ? 'next' : 'next next-enable'" :disabled="nextDisable" @click="nextTab">
        音视频模拟测试正常，下一步
      </button>
      <div :class="isSuccessTextShow ? 'success-text' : ''" v-if="isSuccessTextShow">音视频测试成功</div>
    </section>
  </div>
</template>

<script>

export default {
  name: 'App',
  data() {
    return {
      mediaRecorder: undefined,
      recorderFile: undefined,
      videoStatus: 'START',
      recInterval: undefined,
      scriptProcessor: undefined,
      isRecShow: true,
      averageVolume: 0,
      audioImg: '/0.png',
      nextDisable: true,
      isSuccessTextShow: false,
    }
  },
  watch: {
    averageVolume(newVal) {
      if (newVal >= 100) {
        this.audioImg = '/100.png'
      } else if (newVal >= 80) {
        this.audioImg = '/80.png'
      } else if (newVal >= 60) {
        this.audioImg = '/60.png'
      } else if (newVal >= 40) {
        this.audioImg = '/40.png'
      } else if (newVal >= 20) {
        this.audioImg = '/20.png'
      } else {
        this.audioImg = '/0.png'
      }
    }
  },
  methods: {
    async startRecord() {
      // 获取视频流
      const stream = await navigator.mediaDevices.getUserMedia({
        audio: true,
        video: true,
      })

      let chunks = []
      const video = this.$refs.video

      // 视频录制
      this.mediaRecorder = new MediaRecorder(stream)
      this.mediaRecorder.addEventListener('dataavailable', e => {

        // 录制可用监听
        chunks.push(e.data)
      })
      this.mediaRecorder.addEventListener('stop', () => {

        // 停止录制监听，保存文件
        const mime = 'audio/ogg; codecs=opus'
        this.recorderFile = new Blob(chunks, {
          type: mime
        })
        chunks = []

        // 重新设置播放视频文件
        video.src = URL.createObjectURL(this.recorderFile)
        video.srcObject = undefined
      })

      // 启动播放
      video.srcObject = stream
      await video.play()

      // 启动录制
      this.mediaRecorder.start()

      // 设置按钮图标
      this.videoStatus = 'STOP'

      // 设置 rec 闪烁
      this.recInterval = setInterval(() => {
        this.isRecShow = !this.isRecShow
      }, 500)

      // 检测声音大小
      const audioContext = new AudioContext()
      const analyser = audioContext.createAnalyser()
      const microphone = audioContext.createMediaStreamSource(stream)
      this.scriptProcessor = audioContext.createScriptProcessor(2048, 1, 1)
      analyser.smoothingTimeConstant = 0.8
      analyser.fftSize = 1024
      microphone.connect(analyser)
      analyser.connect(this.scriptProcessor)
      this.scriptProcessor.connect(audioContext.destination)
      this.scriptProcessor.onaudioprocess = () => {
        const array = new Uint8Array(analyser.frequencyBinCount)
        analyser.getByteFrequencyData(array)
        const arraySum = array.reduce((a, value) => a + value, 0)
        this.averageVolume = arraySum / array.length * 1.5
      }
    },
    async stopRecord() {
      // 停止录制
      this.mediaRecorder.stop()
      // 标记 video 为播放
      this.videoStatus = 'PLAY'
      // 清除 rec 定时器
      clearInterval(this.recInterval)
      // 显示 rec
      this.isRecShow = true
      // 关闭音频监控
      this.scriptProcessor.disconnect()
      // 重置音量
      this.averageVolume = 0
    },
    startPlay() {
      // 设置 video 可以暂停
      this.videoStatus = 'PAUSE'
      // 播放视频
      this.$refs.video.play()
      // 添加结束监听起
      this.$refs.video.addEventListener('ended', () => {
        // 解除下一步按钮
        this.nextDisable = false
        // 设置 video 可以播放
        this.videoStatus = 'PLAY'
      }, false);
    },
    pauseVideo() {
      // 设置 video 可以播放
      this.videoStatus = 'PLAY'
      // 暂停视频
      this.$refs.video.pause()
      // 解除下一步按钮
      this.nextDisable = false
    },
    nextTab() {
      this.isSuccessTextShow = true
      setTimeout(() => {
        // this.isSuccessTextShow = false
      }, 4000)
    }
  },
  async mounted() {
    const video = this.$refs.video
    video.srcObject = await navigator.mediaDevices.getUserMedia({
      audio: false,
      video: true,
    })
    await video.play()
  }
}
</script>

<style>
/* header start */
.header {
  top: 0;
  width: 100%;
  height: 40px;
  background-color: #D1DBD9;
  position: absolute;
}

.header .back-button {
  padding-left: 10px;
  height: 40px;
  line-height: 40px;
  border: none;
  background-color: #D1DBD9;
}

.header .header-text {
  height: 40px;
  width: 200px;
  line-height: 40px;
  text-align: center;
  font-size: 15px;
  font-weight: bold;
  position: absolute;
  left: 50%;
  margin-left: -100px;
}

.header .audio-rec {
  display: inline-block;
  position: absolute;
  left: 60%;
  height: 40px;
}

.audio-rec .rec {
  display: inline-block;
  width: 40px;
  height: 40px;
  text-align: center;
  float: left;
}

.audio-rec .rec .rec-border {
  margin: 8px auto 0;
  width: 14px;
  height: 14px;
}

.audio-rec .rec .rec-radius {
  margin: 8px auto 0;
  width: 10px;
  height: 10px;
  border-radius: 50%;
  border: 2px black solid;
  background-color: #D91E1D;
}

.audio-rec .rec .rec-text {
  width: 100%;
  height: 10px;
  font-size: 10px;
  -webkit-text-fill-color: #fff;
  -webkit-text-stroke: 0.5px #000000;
}

.audio-rec .audio {
  display: inline-block;
  height: 40px;
  width: 25px;
  float: left;
}

.audio-rec .audio .audio-img {
  width: 25px;
  height: 25px;
  padding-top: 8px;
}

/* header end */

/* body start */
.body {
  width: 100%;
  height: calc(100% - 40px);
  top: 40px;
  bottom: 0;
  position: absolute;
}

.body .video {
  height: 100%;
  width: 100%;
  object-fit: cover;
  display: block;
}

.body .people-img {
  bottom: 20px;
  height: 80%;
  width: 60%;
  left: 50%;
  margin-left: -30%;
  position: absolute;
}

.body .information {
  width: 500px;
  height: 150px;
  bottom: 50px;
  left: 50%;
  margin-left: -250px;
  padding: 10px 0;
  background-color: rgba(255, 255, 255, 90%);
  border-radius: 3px;
  position: absolute;
}

.body .information .video-button {
  height: 50px;
  text-align: center;
}

.body .video-button .startVideo {
  background-color: #669DFC;
  border: none;
  color: #fff;
  width: 50px;
  height: 50px;
  border-radius: 50%;
  font-size: 10px;
  cursor: pointer;
}

.body .video-button .stopVideo {
  background-color: #D86B6B;
  border: none;
  color: #fff;
  width: 50px;
  height: 50px;
  border-radius: 50%;
  font-size: 10px;
  cursor: pointer;
}

.body .information .text1 {
  width: 470px;
  height: 10px;
  font-size: 12px;
  color: #669DFC;
  text-align: center;
  padding: 15px;
}

.body .information .text2 {
  width: 470px;
  height: 40%;
  font-size: 12px;
  padding: 0 15px 10px 15px;
}

.next {
  position: absolute;
  width: 500px;
  height: 30px;
  bottom: 10px;
  left: 50%;
  margin-left: -250px;
  font-size: 12px;
  color: #ffffff;
  background-color: #B9B9B9;
  border-radius: 3px;
  border: none;
  cursor: not-allowed;
}

.next-enable {
  cursor: pointer;
  background-color: #669DFC;
}

@keyframes success-text {
  from {
    top: -100px
  }
  to {
    top: calc(50% - 40px)
  }
}

.body .success-text {
  position: absolute;
  width: 200px;
  height: 100px;
  border-radius: 5px;
  left: 50%;
  top: calc(50% - 40px);
  text-align: center;
  line-height: 100px;
  background-color: #fff;
  margin-left: -100px;
  margin-top: -10px;
  animation-name: success-text;
  animation-duration: 2s;
}
</style>
