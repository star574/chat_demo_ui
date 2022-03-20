<template>
  <div class="contanier">
    <video id="localVideo" autoplay v-if="remoteConnect" />
    <video id="remoteVideo" autoplay />
    <el-button type="primary" @click="connect">connect</el-button>
  </div>
</template>

<script>
export default {
  data() {
    return {
      wsurl: "ws://localhost:8080/server/",
      remoteConnect: false,
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
      navigator.getUserMedia = navigator.getUserMedia || navigator.getUserMedia || navigator.webkitGetUserMedia || navigator.mozGetUserMedia
      // 调用成功回调
      navigator.getUserMedia({ video: true, audio: false }, function (stream) {
        // 如果没有远程链接 
        clearInterval()
        setInterval(_this.setStream(stream), 100)
      }, console.log)
    },
    setStream(stream) {
      var _this = this;
      let _remoteConnect = this.remoteConnect
      console.log(_remoteConnect);
      if (_remoteConnect) {
        _this.addVideoURL("remoteVideo", stream)
      } else {
        _this.addVideoURL("localVideo", stream)
      }
    },
    // 视频url赋值
    addVideoURL(elementId, stream) {
      var video = document.getElementById(elementId);
      this.localStream = stream
      if ('srcObject' in video) {
        video.srcObject = stream;
      } else {
        // 防止在新的浏览器里使用它，应为它已经不再支持了
        video.src = window.URL.createObjectURL(stream);
      }
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
        console.log(userId + "已关闭WebSocket");
      }
      _this.ws.onerror = function (event) {
        console.log(userId + "发生错误 尝试重连");
        // this.wsConnect()
      }
      _this.ws.onmessage = _this.onmessage

    },
    onmessage(event) {
      let _this = this
      let msg = JSON.parse(event.data)
      let evt = msg
      if (msg.type === 'offer') {
        console.log("接收到offer,设置offer,发送answer....")
        _this.onOffer(evt);
      } else if (msg.type === 'answer' && this.remoteConnect) {
        console.log('接收到answer,设置answer SDP');
        _this.onAnswer(evt);
      } else if (msg.type === 'candidate' && this.remoteConnect) {
        console.log('接收到ICE候选者..');
        _this.onCandidate(evt);
      } else if (msg.type === 'bye' && this.remoteConnect) {
        console.log("WebRTC通信断开");
        _this.stop();
      }
    },
    onCandidate(evt) {
      var candidate = new RTCIceCandidate({
        sdpMLineIndex: evt.sdpMLineIndex,
        sdpMid: evt.sdpMid, candidate: evt.candidate
      });
      console.log("接收到Candidate...")
      console.log(candidate);
      this.peerConnection.addIceCandidate(candidate);
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
      _this.sendAnswer(evt);
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
    sendAnswer(evt) {
      console.log('发送Answer,创建远程会话描述...');
      if (!peerConnection) {
        console.error('peerConnection不存在!');
        return;
      }
      peerConnection.createAnswer(function (sessionDescription) {//成功时
        let mediaConstraints = {
          'mandatory': {
            'OfferToReceiveAudio': true,
            'OfferToReceiveVideo': true
          }
        };
        this.setLocalDescription(sessionDescription);
        console.log("发送: Answer-SDP");
        this.sendSDP(sessionDescription);
      }, function () {                                             //失败时
        console.log("创建Answer失败");
      }, mediaConstraints);
    },
    // 点击call 发送offer
    connect() {
      let _this = this
      // 创建offer
      console.log('remoteConnect :>> ', _this.remoteConnect);
      console.log('wsStatus :>> ', _this.wsStatus);
      if (!_this.remoteConnect && _this.wsStatus) {
        let mediaConstraints = {
          'mandatory': {
            'OfferToReceiveAudio': true,
            'OfferToReceiveVideo': true
          }
        };
        _this.peerConnection = this.RTCConnection()
        _this.peerConnection.createOffer(function (sessionDescription) {
          _this.peerConnection.setLocalDescription(sessionDescription);
          console.log("发送: SDP offer");
          _this.sendSDP(sessionDescription);
        }, function (err) {  //失败时调用
          console.log("创建Offer失败");
        }, mediaConstraints),
          this.remoteConnect = true;

      } else {
        alert("发送offer失败");
      }
    },
    RTCConnection() {
      let _this = this
      // ice 配置
      var pc_config = { "iceServers": [] };
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
      peer.addEventListener("addstream", this.onRemoteStreamAdded, false);
      peer.addEventListener("removestream", this.onRemoteStreamRemoved, false);

      return peer

    },
    // 发送候选者
    sendCandidate(candidate) {
      var text = JSON.stringify(candidate);
      this.ws.send(text)// socket发送
    },

    // 监听视频流
    // 当接收到远程视频流时，使用本地video元素进行显示
    onRemoteStreamAdded(event) {
      let _this = this
      console.log("添加远程视频流");
      // remoteVideo.src = window.URL.createObjectURL(event.stream);
      _this.addVideoURL("remote", event.stream)
    },

    // 当远程结束通信时，取消本地video元素中的显示
    onRemoteStreamRemoved(event) {
      let _this = this
      console.log("移除远程视频流");
      _this.remoteConnect = false
    }
    ,
    sendSDP(text) {
      let msg = JSON.stringify(text)
      this.ws.send(msg)
    },
    stop() {
      this.remoteConnect = false,
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
  position: relative;
}
#localVideo {
  width: 200px;
  height: 250px;
  background-color: rgb(0, 255, 98);
  position: absolute;
  left: 47%;
  bottom: 600px;
  opacity: 0.7;
}
#remoteVideo {
  width: 400px;
  height: 800px;
  background-color: rgba(0, 255, 13, 0.5);
  margin: 0 auto;
}
</style>