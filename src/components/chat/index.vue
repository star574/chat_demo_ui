<template>
  <div class="contanier">
    <video id="localVideo" autoplay />
    <video id="remoteVideo" autoplay />
    <el-button type="primary" @click="connect">connect</el-button>
  </div>
</template>

<script>
export default {
  data() {
    return {
      wsurl: "ws://192.168.31.205:8080/server/",
      wsStatus: false,
      localStream: null,
      peerConnection: null,
      ws: null
    }
  },
  created() {
    // 链接ws
    this.wsConnect()
    this.openVideo()
  },
  methods: {

    // 本地视频部分
    openVideo() {
      // 捕捉视频 
      var _this = this;
      // 打开摄像头
      navigator.getUserMedia = ( navigator.getUserMedia ||
                       navigator.webkitGetUserMedia ||
                       navigator.mozGetUserMedia ||
                       navigator.msGetUserMedia);
      // 调用成功回调
      navigator.getUserMedia({ video: true, audio: false }, function (stream) {
        // 如果没有远程链接 
        _this.localStream = stream
        _this.addVideoURL("localVideo", stream)
      }, console.log)
    },
    // 视频url赋值
    addVideoURL(elementId, stream) {
      let video = document.getElementById(elementId);
      video.srcObject = stream;
      video.play()
    },
    // ws部分
    wsConnect() {
      // 构建url
      let userId = Math.ceil(Math.random() * 99) + 1;
      let wsurl = this.wsurl + userId
      let _this = this
      // 链接ws
      _this.ws = new WebSocket(wsurl)
      _this.ws.onopen = function (event) {
        console.log(userId + "已连接WebSocket");
        _this.wsStatus = true
      }
      _this.ws.onclose = function (event) {
        _this.stop()
        console.log(userId + "已关闭WebSocket");
      }
      _this.ws.onerror = function (event) {
        _this.stop()
        console.log(userId + "发生错误 尝试重连");
        // this.wsConnect()
      }
      _this.ws.onmessage = _this.onmessage

    },
    onmessage(event) {
      let _this = this
      try {
        let msg = JSON.parse(event.data)
        if (msg.type === 'offer') {
          console.log("接收到offer,设置offer,发送answer....")
          _this.onOffer(msg);
        } else if (msg.type === 'answer') {
          console.log('接收到answer,设置answer SDP-------------------------------');
          _this.onAnswer(msg);
        } else if (msg.type === 'candidate') {
          console.log('接收到ICE候选者..');
          _this.onCandidate(msg);
        }
        console.log(msg)
      }
      catch (error) {
        this.$notify.success(
          {
            title: '提示',
            message: event.data,
            type: 'success'
          }
        )
      }

    },
    onCandidate(evt) {
      console.log(evt);
      let _peerConnection = this.peerConnection
      let candidate = new RTCIceCandidate({
        sdpMLineIndex: evt.sdpMLineIndex,
        sdpMid: evt.sdpMid, candidate: evt.candidate
      });
      _peerConnection.addIceCandidate(candidate);
    },
    onAnswer(evt) {
      this.setAnswer(evt);
    },
    setAnswer(evt) {
      if (!this.peerConnection) {
        console.error('peerConnection不存在!');
        return;
      }
      this.peerConnection.setRemoteDescription(new RTCSessionDescription(evt));
    },
    // 其他客户端接收到offer 设置offer 发送answer
    onOffer(evt) {
      let _this = this
      _this.setOffer(evt);
      _this.sendAnswer();
    },
    setOffer(evt) {
      if (this.peerConnection) {
        console.error('peerConnection已存在!');
        return;
      }
      // 处理连接
      this.peerConnection = this.RTCConnection()
      this.peerConnection.setRemoteDescription(new RTCSessionDescription(evt));
    },
    sendAnswer() {
      let _this = this
      let _peerConnection = this.peerConnection
      if (!_peerConnection) {
        console.error('peerConnection不存在!');
        return;
      }
      let mediaConstraints = {
        'mandatory': {
          'OfferToReceiveAudio': true,
          'OfferToReceiveVideo': true
        }
      }
      console.log('准备创建Answer-SDP');
      _peerConnection.createAnswer(function (sessionDescription) {//成功时
        console.log("发送: Answer-SDP");
        console.log(sessionDescription);
        _this.sendSDP(sessionDescription);
        _peerConnection.setLocalDescription(sessionDescription);
      }, function () {                                             //失败时
        console.log("创建Answer失败");
      }, mediaConstraints);
    },
    // 点击call 发送offer
    connect() {
      let _this = this
      // 创建offer
      if (_this.wsStatus) {
        let mediaConstraints = {
          'mandatory': {
            'OfferToReceiveAudio': true,
            'OfferToReceiveVideo': true
          }
        };
        _this.peerConnection = this.RTCConnection()
        _this.peerConnection.createOffer(function (sessionDescription) {
          _this.peerConnection.setLocalDescription(sessionDescription);
          _this.sendSDP(sessionDescription);
        }, function (err) {  //失败时调用
          console.log("创建Answer失败");
        }, mediaConstraints)
      } else {
        alert("发送offer失败");
      }
    },
    RTCConnection() {
      let _this = this
      // ice 配置
      var pc_config = { "iceServers": [
      ] };
      // 创建RTC链接
      var peer = null;
      try {
        peer = new webkitRTCPeerConnection(pc_config);
      }
      catch (e) {
        console.log("建立连接失败，错误：" + e.message);
      }
      // 发送所有ICE候选者给对方
      peer.onicecandidate = function (evt) {
        if (evt.candidate) {
          // 发送候选者
          _this.sendCandidate({
            type: "candidate",
            sdpMLineIndex: evt.candidate.sdpMLineIndex,
            sdpMid: evt.candidate.sdpMid,
            candidate: evt.candidate.candidate
          });
        }
      };
      console.log('添加本地视频流...');
      peer.addStream(_this.localStream);

      // 监听视频流
      peer.addEventListener("addstream", _this.onRemoteStreamAdded, false);
      peer.addEventListener("removestream", _this.onRemoteStreamRemoved, false);

      return peer

    },
    // 发送候选者
    sendCandidate(candidate) {
      this.sendSDP(candidate)// socket发送
    },

    // 监听视频流
    // 当接收到远程视频流时，使用本地video元素进行显示
    onRemoteStreamAdded(event) {
      let _this = this
      console.log("添加远程视频流");
      // remoteVideo.src = window.URL.createObjectURL(event.stream);
      console.log("远程视频流信息:");
      console.log(event.stream);
      _this.addVideoURL("remoteVideo", event.stream)
    },

    // 当远程结束通信时，取消本地video元素中的显示
    onRemoteStreamRemoved(event) {
      let _this = this
      console.log("移除远程视频流");
    }
    ,
    sendSDP(text) {
      let msg = JSON.stringify(text)
      this.ws.send(msg)
    },
    stop() {
      this.wsStatus = false,
        this.localStream = null,
        this.peerConnection = null
    }
  }
}
</script>

<style>
.contanier {
  width: 100vw;
  height: 100vh;
  background-image: linear-gradient(45deg, red, yellow);
  display: flex;
  justify-content: center;
  align-items: center;
}
#localVideo {
  width: 400px;
  height: 800px;
}
#remoteVideo {
  width: 400px;
  height: 800px;
  margin: 0 auto;
}
</style>