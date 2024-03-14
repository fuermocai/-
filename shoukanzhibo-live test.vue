<template>
  <el-container class="app-container">
    <el-main>
      <el-row>
        <!-- Video Preview Section -->
        <el-col :span="18">
          <el-card class="preview-roomInfo">
            <div>RoomID: <span>{{ roomID }}</span></div>
            <div>RoomState:
              <template v-if="connectStatus === 'CONNECTED'">
                <el-badge value="Connected" type="success"></el-badge>
              </template>
              <template v-else-if="connectStatus === 'DISCONNECTED'">
                <el-badge value="Disconnected" type="danger"></el-badge>
              </template>
            </div>
            <div>
              <span>Preview</span>&emsp;Publish StreamID: <span>{{ publishInfoStreamID }}</span>
              <div class="preview-content">
                <video v-if="version<`2.17.0`" controls id="playVideo" ref="playVideo" autoplay playsinline
                       :class="(mirrorVal=='onlyPlay' || mirrorVal=='both')?'mirror':''"></video>
                <div v-else id="remoteVideo"></div>
              </div>
            </div>
          </el-card>
        </el-col>

        <!-- Actions Section -->
        <el-col :span="6">
          <el-card class="action-section">
            <el-button @click="createZegoExpressEngineOption" :disabled="createSuccessSvgStatus" type="info">创建引擎</el-button>
            <el-button @click="checkSystemRequire" :type="checkSystemRequireStatus === 'ERROR' ? 'danger' : 'info'">测试设备</el-button>
            <el-form label-position="top">
              <el-form-item label="RoomID">
                <el-input v-model="roomID" :disabled="isLogin"></el-input>
              </el-form-item>
              <el-form-item label="UserID">
                <el-input v-model="userID" :disabled="isLogin"></el-input>
              </el-form-item>
              <el-form-item label="Token">
                <el-input v-model="token"></el-input>
              </el-form-item>
              <el-form-item>
                <el-button @click="loginRoomOption" :disabled="isLogin" type="primary">登录</el-button>
              </el-form-item>
              <el-form-item label="视频">
                <el-switch v-model="videoCheckStatus" :disabled="playStreamStatus"></el-switch>
              </el-form-item>
              <el-form-item label="声音">
                <el-switch v-model="audioCheckStatus" :disabled="playStreamStatus"></el-switch>
              </el-form-item>
              <el-form-item>
                <el-button @click="startPlaying" :disabled="playStreamStatus" type="success">开始拉流</el-button>
              </el-form-item>
              <el-form-item>
                <el-button @click="reset" type="danger">重置</el-button>
              </el-form-item>
            </el-form>
          </el-card>
        </el-col>
      </el-row>
    </el-main>
  </el-container>
</template>


<script>
import { ZegoExpressEngine } from 'zego-express-engine-webrtc'
const { generateToken04 } = require('../util/zegoServerAssistant');
const appID = ;
const serverSecret = '';
const userId = "student" +
    '_' + new Date().getTime();
const effectiveTimeInSeconds = 86400;
const payload = '';
const token =  generateToken04(appID, userId, serverSecret, effectiveTimeInSeconds, payload);
export default {
  data() {
    return {
      appID: ,
      server: ,
      userID: userId,
      roomID: '0001',
      token: token,
      streamID: '0001',
      playStreamID: '0001',
      zg: null,
      localStream: null,
      remoteStream: null,
      isLogin: false,
      videoCodec: localStorage.getItem('VideoCodec') === 'H.264' ? 'H264' : 'VP8',
      audioDeviceList: [],
      videoDeviceList: [],
      createSuccessSvgStatus: false,
      checkSystemRequireStatus: '',
      connectStatus: 'DISCONNECTED',
      microphoneDevicesVal: null,
      cameraDevicesVal: '',
      cameraCheckStatus: true,
      microphoneCheckStatus: true,
      publishStreamStatus: false,
      videoCheckStatus: true,
      audioCheckStatus: false,
      playStreamStatus: false,
      mirrorVal: 'none',
      publishInfoStreamID: '',
      playInfoStreamID: ''
    }
  },
  methods: {
    async enumDevices() {
      const deviceInfo = await this.zg.enumDevices();
      this.audioDeviceList = deviceInfo &&
          deviceInfo.microphones.map((item, index) => {
            if (!item.deviceName) {
              item.deviceName = 'microphone' + index;
            }
            console.log('microphone: ' + item.deviceName);
            return item;
          });
      this.audioDeviceList.push({ deviceID: 0, deviceName: '禁止' });
      this.microphoneDevicesVal = this.audioDeviceList[0].deviceID;
      this.videoDeviceList = deviceInfo &&
          deviceInfo.cameras.map((item, index) => {
            if (!item.deviceName) {
              item.deviceName = 'camera' + index;
            }
            console.log('camera: ' + item.deviceName);
            return item;
          });
      this.videoDeviceList.push({ deviceID: 0, deviceName: '禁止' });
      this.cameraDevicesVal = this.videoDeviceList[0].deviceID;
    },
    initEvent() {
      this.zg.on('roomStateUpdate', (roomId, state) => {
        if (state === 'CONNECTED') {
          this.connectStatus = 'CONNECTED';
        }
        if (state === 'DISCONNECTED') {
          this.connectStatus = 'DISCONNECTED';
        }
      });
      this.zg.on('publisherStateUpdate', (result) => {
        if (result.state === 'PUBLISHING') {
          this.publishInfoStreamID = result.streamID;
        } else if (result.state === 'NO_PUBLISH') {
          this.publishInfoStreamID = "";
        }
      });
      this.zg.on('playerStateUpdate', (result) => {
        if (result.state === 'PLAYING') {
          this.playInfoStreamID = result.streamID;
        } else if (result.state === 'NO_PLAY') {
          this.playInfoStreamID = "";
        }
      });
    },
    createZegoExpressEngine() {
      this.zg = new ZegoExpressEngine(this.appID, this.server);
      window.zg = this.zg;
    },
    async checkSystemRequirements() {
      console.log('sdk version is', this.zg.getVersion());
      try {
        const result = await this.zg.checkSystemRequirements();
        console.warn('checkSystemRequirements ', result);
        if (!result.webRTC) {
          console.error('browser is not support webrtc!!');
          return false;
        } else if (!result.videoCodec.H264 && !result.videoCodec.VP8) {
          console.error('browser is not support H264 and VP8');
          return false;
        } else if (!result.camera && !result.microphone) {
          console.error('camera and microphones not allowed to use');
          return false;
        } else if (result.videoCodec.VP8) {
          if (!result.screenSharing) console.warn('browser is not support screenSharing');
        } else {
          console.log('不支持VP8，请前往混流转码测试');
        }
        return true;
      } catch (err) {
        console.error('checkSystemRequirements', err);
        return false;
      }
    },
    async loginRoom(roomId, userId, userName, token) {
      return await this.zg.loginRoom(roomId, token, {
        userID: userId,
        userName
      });
    },
    async startPlayingStream(streamId, options = {}) {
      try {
        this.remoteStream = await this.zg.startPlayingStream(streamId, options);
        if (this.zg.getVersion() < "2.17.0") {
          this.$refs['playVideo'].srcObject = this.remoteStream;
        } else {
          const remoteView = this.zg.createRemoteStreamView(this.remoteStream);
          remoteView.play("remoteVideo", {
            objectFit: "cover",
            enableAutoplayDialog: true,
          })
        }
        return true;
      } catch (err) {
        alert("当前无直播！")
        return this.$router.go(-1)
      }
    },
    logoutRoom(roomId) {
      this.zg.logoutRoom(roomId);
    },
    async stopPublishingStream(streamId) {
      this.zg.stopPublishingStream(streamId);
    },
    async stopPlayingStream(streamId) {
      this.zg.stopPlayingStream(streamId);
    },
    clearStream() {
      this.localStream && this.zg.destroyStream(this.localStream);
      this.localStream = null;
      this.remoteStream && this.zg.destroyStream(this.remoteStream);
      this.remoteStream = null;
      if (this.zg.getVersion() < "2.17.0") {
        this.$refs['publishVideo'].srcObject = null;
        this.$refs['playVideo'].srcObject = null;
      }
    },
    changeAudioDevices() {
      if (!this.zg || !this.localStream) {
        return
      }
      const isMicrophoneMuted = this.zg.isMicrophoneMuted();
      if (!isNaN(this.microphoneDevicesVal) && !isMicrophoneMuted) {
        this.zg.muteMicrophone(true);
      } else {
        this.zg.muteMicrophone(false);
        this.zg.useAudioDevice(this.localStream, this.microphoneDevicesVal);
      }
    },
    createZegoExpressEngineOption() {
      if (!this.createSuccessSvgStatus) {
        this.createZegoExpressEngine();
        this.createSuccessSvgStatus = true;
        this.initEvent();
      }
    },
    async checkSystemRequire() {
      if (!this.zg) return alert('you should create zegoExpressEngine');
      const result = await this.checkSystemRequirements();
      if (result) {
        this.checkSystemRequireStatus = 'SUCCESS';
        this.enumDevices();
      } else {
        this.checkSystemRequireStatus = 'ERROR';
      }
    },
    async loginRoomOption() {
      try {
        this.isLogin = await this.loginRoom(this.roomID, this.userID, this.userID, this.token);
      } catch (err) {
        this.isLogin = false;
        console.log(err);
      }
    },
    async startPublishing() {
      const flag = await this.startPublishingStream(this.streamID, {
        camera: {
          audioInput: this.microphoneDevicesVal,
          videoInput: this.cameraDevicesVal,
          video: this.cameraCheckStatus,
          audio: this.microphoneCheckStatus,
        }
      })
      if (flag) {
        this.publishStreamStatus = true;
      }
    },
    async startPlaying() {
      const flag = await this.startPlayingStream(this.playStreamID, {
        video: this.videoCheckStatus,
        audio: this.audioCheckStatus
      });
      if (flag) {
        this.playStreamStatus = true;
      }
    },
    async reset() {
      if (!this.zg) {
        return
      }
      await this.stopPublishingStream(this.streamID);
      await this.stopPlayingStream(this.playStreamID);
      if (this.isLogin) {
        this.isLogin = false;
        this.logoutRoom(this.roomID);
      }
      this.clearStream();
      this.zg = null;
      window.zg = null;
      this.playStreamStatus = false;
      this.publishStreamStatus = false;
      this.createSuccessSvgStatus = false;
      this.checkSystemRequireStatus = '';
      this.audioCheckStatus = false;
    }
  },
  mounted() {},
  computed: {
    version(){
      return this.zg?.getVersion() || 0
    }
  }
}
</script>

<style scoped>


.action-section{
  height: 100vh;
}
.preview-content, #remoteVideo {
  width: 100%;
  height: 90vh;
  /*max-height: 600px; !* 调整视频预览区的最大高度，保持响应式 *!*/
  background-color: black; /* 预览区域默认背景色 */
}


select{
  border-radius: 4px; /* 统一输入框和选择框的圆角 */
  height: 34px; /* 调整输入框和选择框的高度 */
}

@media (max-width: 768px) {
  .preview-content, #remoteVideo {
    max-height: 300px; /* 在较小屏幕上调整视频预览区的最大高度 */
  }
}

/* 隐藏不使用的 .icon-question .pop-box */
.icon-question .icon-question:hover {
  display: none;
}
</style>




