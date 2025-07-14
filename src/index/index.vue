<template>
  <div class="flex items-center flex-col text-center p-12 h-screen">
    <div class="relative h-full mb-4">
      <video ref="localVideo" class="localVideo"
        :class="videoStyle ? 'w-full h-full bg-gray-200 mb-4 object-cover' : 'w-80 h-48 absolute bottom-0 right-0 object-cover'"
        @click="videoStyle = !videoStyle"></video>
      <video ref="remoteVideo" class="remoteVideo"
        :class="videoStyle ? 'w-80 h-48 absolute bottom-0 right-0 object-cover' : 'w-full h-full bg-gray-200 mb-4 object-cover'"
        @click="videoStyle = !videoStyle"></video>
      <div v-if="caller && calling" class="absolute bottom-8 w-full flex flex-col items-center">
        <p class="mb-4 text-white">等待对方接听...</p>
        <img @click="hangUp" src="/refuse.svg" class="w-16 cursor-pointer" alt="">
      </div>
      <div v-if="called && calling" class="absolute bottom-8 w-full flex flex-col items-center">
        <p class="mb-4 text-white">收到视频邀请...</p>
        <div class="flex">
          <img @click="hangUp" src="/refuse.svg" class="w-16 cursor-pointer mr-4" alt="">
          <img @click="acceptCall" src="/accept.svg" class="w-16 cursor-pointer" alt="">
        </div>
      </div>
    </div>
    <div class="flex gap-2 mb-4" v-if="communicating">
      <button class="rounded-md bg-indigo-600 px-4 py-2 text-sm font-semibold text-white" @click="closeAudio">关闭麦克风
      </button>
      <button class="rounded-md bg-red-600 px-4 py-2 text-sm font-semibold text-white" @click="closeVideo">
        {{ videoState ? '关闭摄像头' : '开启摄像头' }}
      </button>
      <!-- 👇 新增：切换摄像头按钮 -->
      <button class="rounded-md bg-yellow-600 px-4 py-2 text-sm font-semibold text-white" @click="switchCamera">
        切换摄像头
      </button>
    </div>
    <div class="flex gap-2 mb-4">
      <input type="text" class="rounded-md px-4 py-2 text-sm border border-blue-300" placeholder="请输入房间号"
        v-model="roomId">
      <button class="rounded-md bg-green-600 px-4 py-2 text-sm font-semibold text-white" @click="enterRoom">进入房间
      </button>

    </div>

    <div class="flex gap-2 mb-4">
      <button v-if="inRoom" class="rounded-md bg-indigo-600 px-4 py-2 text-sm font-semibold text-white"
        @click="callRemote">发起视频
      </button>
      <button class="rounded-md bg-red-600 px-4 py-2 text-sm font-semibold text-white" @click="hangUp"
        v-if="communicating">挂断视频
      </button>
    </div>
  </div>
  <!-- <img alt="Vue logo" src="../assets/logo.png" /> -->
  <div>
    <button @click="openNewWin">打开新窗口</button>
  </div>
</template>
<script setup lang="ts">
import { useIpcRenderer } from "@vueuse/electron";
const ipcRenderer = useIpcRenderer();
const openNewWin = () => {
  ipcRenderer.send("window-new", {
    route: "/helloworld",
    width: 500,
    height: 500,
  });
};
import { onMounted, ref } from 'vue'

const roomId = ref<any>('1') // 房间ID

const called = ref<boolean>(false) // 是否是接收方
const caller = ref<boolean>(false) // 是否是发起方
const calling = ref<boolean>(false) // 呼叫中
const communicating = ref<boolean>(false) // 视频通话中
const localVideo = ref<HTMLVideoElement>() // video标签实例，播放本人的视频
const remoteVideo = ref<HTMLVideoElement>() // video标签实例，播放对方的视频
const localStream = ref<MediaStream>() // 本地流
const ws = ref<WebSocket>() // WebSocket 实例
const peer = ref<any>() // RTCPeerConnection实例

const inRoom = ref<boolean>(false) // 是否在房间中，开房间才可通信

const serverConfig = ref<any>(null)

onMounted(() => {
  const socket = new WebSocket("ws://localhost:3002/ws")
  loadVideoDevices()

  socket.onopen = () => {
    console.log("WebSocket 连接建立成功")
    // todo 创建房间，服务端要拼接上时间戳，防止房间号一致
    // send({event: "joinRoom", roomId})
  }

  socket.onmessage = (e) => {
    const msg = JSON.parse(e.data)
    handleEvent(msg)
  }

  socket.onclose = () => {
    console.log("WebSocket 已关闭")
    reset()
  }

  socket.onerror = (err) => {
    console.error("WebSocket 错误:", err)
  }

  function send(message: any) {
    if (socket.readyState === WebSocket.OPEN) {
      socket.send(JSON.stringify(message))
    } else {
      setTimeout(() => send(message), 100)
    }
  }

  function handleEvent(msg: any) {
    const { event, roomId: msgRoomId, data } = msg

    if (msgRoomId !== roomId.value) {
      alert("房间不存在")
      return
    }

    switch (event) {
      case "connectionSuccess":
        // todo 接收到加密密钥
        serverConfig.value = data
        inRoom.value = true
        break

      case "joinRoomError":
        // 加入失败
        alert(data)
        // todo 禁用一些选项
        break

      case "receiveCall":
        console.log("收到视频邀请", caller.value)
        if (!caller.value) {
          called.value = true
          calling.value = true
        }
        break

      case "acceptCall":
        if (caller.value) {
          createPeerConnection()
        }
        break

      case "sendOffer":
        if (called.value) {
          handleOffer(data)
        }
        break

      case "receiveAnswer":
        if (caller.value) {
          console.log("收到视频应答", data)
          // ✅ 适配 Android 格式
          const answer: RTCSessionDescriptionInit = {
            type: data.type?.toLowerCase?.(),              // "ANSWER" → "answer"
            sdp: data.sdp || data.description || ""        // 用 sdp，兼容 description 字段
          }

          if (!answer.sdp || !answer.type) {
            console.error("无效的 answer 数据", data)
            return
          }

          // ✅ 设置远端 SDP
          peer.value?.setRemoteDescription(new RTCSessionDescription(answer))
            .then(() => {
              console.log("✅ 成功设置远端 answer")
            })
            .catch(err => {
              console.error("❌ 设置远端 answer 失败", err)
            })
        }
        break

      case "receiveCandidate":
        if (peer.value && data) {
          peer.value.addIceCandidate(new RTCIceCandidate(data))
        }
        break

      case "hangUp":
        reset()
        break

      default:
        console.warn("未知事件：", event)
    }
  }

  /**
   * 发起者创建PeerConnection
   */
  async function createPeerConnection() {
    peer.value = new RTCPeerConnection(
      {
        iceServers: serverConfig.value?.iceServers
      }
    )
    console.log("发送方PeerConnection：", peer.value)

    peer.value.addStream(localStream.value)

    peer.value.onicecandidate = (e) => {
      if (e.candidate) {
        send({ event: "sendCandidate", roomId: roomId.value, data: e.candidate })
      }
    }

    peer.value.onaddstream = (e) => {
      calling.value = false
      communicating.value = true
      remoteVideo.value!.srcObject = e.stream
      remoteVideo.value!.play()
    }

    // 发送offer
    const offer = await peer.value.createOffer({
      offerToReceiveAudio: true,
      offerToReceiveVideo: true
    })

    await peer.value.setLocalDescription(offer)
    send({ event: "sendOffer", roomId: roomId.value, data: offer })
  }

  /**
   * 接收方处理offer
   * @param offer
   */
  async function handleOffer(rawOffer: any) {
    // ✅ 自动兼容 Android 发来的 `{type, description}` 格式
    const offer: RTCSessionDescriptionInit = (() => {
      if (typeof rawOffer === 'string') rawOffer = JSON.parse(rawOffer)

      // 修正大小写问题，并兼容 description 字段
      const type = rawOffer.type?.toLowerCase?.() || 'offer'
      const sdp = rawOffer.sdp || rawOffer.description

      return { type, sdp }
    })()
    console.log("接收方收到offer：", offer)
    const stream = await getLocalStream()
    console.log('🟡 本地流 tracks:', stream.getTracks())
    peer.value = new RTCPeerConnection()
    console.log("接收方PeerConnection：", peer.value)

    peer.value.addStream(stream)

    peer.value.onicecandidate = (e) => {
      if (e.candidate) {
        send({ event: "sendCandidate", roomId: roomId.value, data: e.candidate })
      }
    }

    peer.value.onaddstream = (e) => {
      calling.value = false
      communicating.value = true
      remoteVideo.value!.srcObject = e.stream
      remoteVideo.value!.play()
    }

    await peer.value.setRemoteDescription(new RTCSessionDescription(offer))

    const answer = await peer.value.createAnswer()
    await peer.value.setLocalDescription(answer)
    send({ event: "sendAnswer", roomId: roomId.value, data: answer })
  }

  ws.value = socket
})

// 获取本地音视频流
// const getLocalStream = async () => {
//   const stream = await navigator.mediaDevices.getUserMedia({
//     video: true,
//     audio: true
//   })
//
//   localVideo.value!.srcObject = stream
//   localVideo.value!.play()
//   localStream.value = stream
//
//   return stream
// }

// 发起视频请求（发起方）
const callRemote = async () => {
  if (calling.value || communicating.value) return
  calling.value = true
  console.log("发起视频请求，摄像头：", currentVideoDeviceId.value)
  await getLocalStream(currentVideoDeviceId.value)
  caller.value = true
  send({ event: "callRemote", roomId: roomId.value })
}

// 接收视频请求（接受方）
const acceptCall = () => {
  send({ event: "acceptCall", roomId: roomId.value })
}

// 挂断视频
const hangUp = () => {
  send({ event: "hangUp", roomId: roomId.value })
}

// 状态重置
const reset = () => {
  called.value = false
  caller.value = false
  calling.value = false
  communicating.value = false
  peer.value = null
  if (localVideo.value) localVideo.value.srcObject = null
  if (remoteVideo.value) remoteVideo.value.srcObject = null
  if (localStream.value) localStream.value.getTracks().forEach(track => track.stop())
  inRoom.value = false
}

function send(message: any) {
  if (ws.value?.readyState === WebSocket.OPEN) {
    ws.value.send(JSON.stringify(message))
  } else {
    setTimeout(() => send(message), 100)
  }
}

/**
 * 进入房间
 */
const enterRoom = () => {
  send({ event: "joinRoom", roomId: roomId.value })
}

const closeAudio = () => {

}

/**
 * 视频状态 open or close
 */
const videoState = ref(true)

/**
 * 开始或关闭摄像头
 */
const closeVideo = () => {
  videoState.value = !videoState.value
  // 开始或关闭
  localStream.value?.getVideoTracks().forEach(track => track.enabled = videoState.value)
  // if (videoState.value) {
  //
  //   localStream.value?.getVideoTracks().forEach(track => track.enabled = true)
  // } else {
  //   localStream.value?.getVideoTracks().forEach(track => track.enabled = false)
  // }
}

const videoStyle = ref(true)

/**
 * todo 识别所有可用的摄像头
 */
const videoDevices = ref<MediaDeviceInfo[]>([])
const currentVideoDeviceId = ref<string>('')

/**
 * 获取所有摄像头设备
 */
const loadVideoDevices = async () => {
  const devices = await navigator.mediaDevices.enumerateDevices()
  videoDevices.value = devices.filter(d => d.kind === 'videoinput')
  if (!currentVideoDeviceId.value && videoDevices.value.length > 0) {
    currentVideoDeviceId.value = videoDevices.value[0].deviceId
  }
  console.log('摄像头设备：', videoDevices.value)
  console.log('当前摄像头：', currentVideoDeviceId.value)
}

/**
 * 获取本地流，支持传入 deviceId
 */
const getLocalStream = async (deviceId?: string) => {
  const constraints: MediaStreamConstraints = {
    video: deviceId ? { deviceId: { exact: deviceId } } : true,
    audio: true
  }

  const stream = await navigator.mediaDevices.getUserMedia(constraints)

  // 替换当前视频流
  if (localStream.value) {
    localStream.value.getTracks().forEach(track => track.stop())
  }

  localVideo.value!.srcObject = stream
  localVideo.value!.play()
  localStream.value = stream

  return stream
}

/**
 * 切换摄像头
 */
const switchCamera = async () => {
  alert('切换摄像头' + videoDevices.value.length)
  if (videoDevices.value.length < 2) {
    alert('未检测到多个摄像头')
    return
  }

  const currentIndex = videoDevices.value.findIndex(d => d.deviceId === currentVideoDeviceId.value)
  const nextIndex = (currentIndex + 1) % videoDevices.value.length
  const nextDeviceId = videoDevices.value[nextIndex].deviceId

  currentVideoDeviceId.value = nextDeviceId
  await getLocalStream(nextDeviceId)

  // 如果通话中，将替换视频轨道给 peerConnection
  if (peer.value) {
    const videoTrack = localStream.value!.getVideoTracks()[0]
    const sender = peer.value.getSenders().find((s: RTCRtpSender) => s.track?.kind === 'video')
    if (sender) {
      sender.replaceTrack(videoTrack)
    }
  }
}

</script>