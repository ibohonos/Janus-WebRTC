<script setup>
  import { ref } from "vue";
  import { log } from "@/services/utility.js";
  import adapter from 'webrtc-adapter';
  import Janus from 'janus-gateway'

  // const JANUS_URL = 'http://localhost:8088/janus'
  const JANUS_URL = ['ws://localhost:8188', 'http://localhost:8088/janus']

  let janus = ref(null)
  let janusStream = ref(null)
  let plugin = ref(null)
  let sharePlugin = ref(null)
  let localStream = ref(null)
  let localScreenShare = ref(null)
  let roomId = ref(1234)
  let isStreaming = ref(false)
  let streamVideos = ref({})
  let shareTrack = ref(null)

  async function startStreaming() {
    try {
      localStream.value = await navigator.mediaDevices.getUserMedia({ video: true, audio: true });

      janus.value = new Janus({
        server: JANUS_URL,
        success: () => {
          attachStreamingPlugin(localStream.value)
        },
        error: (error) => {
          console.error('Failed to connect to janus server', error)
        },
        destroyed: () => {
          window.location.reload()
          log("destroyed")
        }
      })
    } catch (error) {
      onError('Error accessing camera:', error)
    }
  }

  async function startShare() {
    try {
      localScreenShare.value = await navigator.mediaDevices.getUserMedia({
        video: {
          displaySurface: "window",
        },
        audio: false,
      })

      log("startShare", localScreenShare.value)

      janusStream.value = new Janus({
        server: JANUS_URL,
        success: () => {
          attachSharePlugin(localScreenShare.value)
        },
        error: (error) => {
          console.error('Failed to connect to janus server', error)
        },
        destroyed: () => {
          // window.location.reload()
          log("destroyed")
        }
      })
    } catch (error) {
      onError('Error accessing screen:', error)
    }
  }

  function stopShare() {
    if (shareTrack.value) {
      log("stopShare", shareTrack.value)
      shareTrack.value.stop()
      delete streamVideos.value[shareTrack.value["id"]]
    }

    if (sharePlugin.value) {
      sharePlugin.value.send({ 
        message: { request: 'unpublish' },
        success: () => {
          if (janusStream.value) {
            janusStream.value.destroy()
          }
        }
      })
    }
  }

  function attachStreamingPlugin(stream) {
    janus.value.attach({
      plugin: "janus.plugin.videoroom",
      opaqueId: Janus.randomString(12),
      success: (pluginHandle) => {
          log("Publisher plugin attached!");
          log(pluginHandle);
          plugin.value = pluginHandle

          let request = {
            request: "join",
            room: roomId.value,
            ptype: "publisher",
            id: 54321,
            display: "broadcaster",
            pin: "123123"
          };

          plugin.value.send({ message: request });
        },
        onmessage: async (message, jsep) => {
          log("onmessage", message, jsep)

          if (message.videoroom === "joined") {
            // Joined successfully, create SDP Offer with our stream
            log("Joined room! Creating offer...");

            let mediaConfig = [];

            if (stream === null || typeof stream === "undefined") {
              mediaConfig = [
                  { type: 'audio', capture: false, recv: false, add: false },
                  { type: 'video', capture: false, recv: false, add: false },
              ]
            } else {
              mediaConfig = [
                  { type: 'audio', capture: true, recv: false, add: true },
                  { type: 'video', capture: true, recv: false, add: true },
              ]
            }

            log("Media Configuration for Publisher set! ->");
            log(mediaConfig);

            plugin.value.createOffer({
              tracks: mediaConfig,
              success: (sdpAnswer) => {
                // SDP Offer answered, publish our stream
                log("Offer answered! Start publishing...")
                let publish = {
                  request: "configure",
                  audio: true,
                  video: true,
                  data: true
                };
                plugin.value.send({ message: publish, jsep: sdpAnswer })
              },
              error: (error) => {
                log("createOffer error", error)
              }
            })
          } else if (message.videoroom === "destroyed") {
            // Room has been destroyed, time to leave...
            log("Room destroyed! Time to leave...")
          }

          if (message.unpublished) {
            // We've gotten unpublished (disconnected, maybe?), leaving...
            if (message.unpublished === "ok") {
              log("We've gotten disconnected, hanging up...")
              plugin.value.hangup();
            }
          }

          if (jsep) {
            log("Handling remote JSEP SDP")
            log(jsep)
            plugin.value.handleRemoteJsep({ jsep: jsep })
          }
        },
        onlocaltrack: (track, on) => {
          log("Successfully published local track: ", on, track);
          if(!on) {
            if(track.kind === "video") {
              delete streamVideos.value[track["id"]]
              
              isStreaming.value = Object.keys(streamVideos.value).length > 0
            }

            return
          }

          onLocalStream(track)
        },
        error: (err) => {
          log("Publish: Janus VideoRoom Plugin Error!", true);
          log(err, true);
        },
        oncleanup: () => {
            // PeerConnection with the plugin closed, clean the UI
            // The plugin handle is still valid so we can create a new one
          log("oncleanup")
          onCleanup()
        },
        ondetached: () => {
            // Connection with the plugin closed, get rid of its features
            // The plugin handle is not valid anymore
          log("ondetached plugin")
        }
    })
  }

  function attachSharePlugin(stream) {
    janusStream.value.attach({
      plugin: "janus.plugin.videoroom",
      opaqueId: Janus.randomString(12),
      success: (pluginHandle) => {
          log("sharePlugin Publisher plugin attached!");
          log(pluginHandle);
          sharePlugin.value = pluginHandle

          let request = {
            request: "join",
            room: roomId.value,
            ptype: "publisher",
            id: 543210,
            display: "broadcaster",
            pin: "123123"
          };

          sharePlugin.value.send({ message: request });
        },
        onmessage: async (message, jsep) => {
          log("sharePlugin onmessage", message, jsep)

          // is Joined
          if (message.error_code === 436) {
            joinToShare(stream)
          }

          if (message.videoroom === "joined") {
            // Joined successfully, create SDP Offer with our stream
            joinToShare(stream)
          } else if (message.videoroom === "destroyed") {
            // Room has been destroyed, time to leave...
            log("sharePlugin Room destroyed! Time to leave...")
          }

          if (message.unpublished) {
            // We've gotten unpublished (disconnected, maybe?), leaving...
            if (message.unpublished === "ok") {
              log("sharePlugin We've gotten disconnected, hanging up...")
              sharePlugin.value.hangup();
            }
          }

          if (jsep) {
            log("sharePlugin Handling remote JSEP SDP")
            log(jsep)
            sharePlugin.value.handleRemoteJsep({ jsep: jsep })
          }
        },
        onlocaltrack: (track, on) => {
          log("sharePlugin Successfully published local track: ", on, track);
          if(!on) {
            if(track.kind === "video") {
              delete streamVideos.value[track["id"]]
              track.stop()
              stopShare()
            }

            return
          }

          onLocalStream(track)
          shareTrack.value = track
        },
        error: (err) => {
          log("sharePlugin Publish: Janus VideoRoom Plugin Error!", true)
          log(err, true)
          onError("sharePlugin Publish: Janus VideoRoom Plugin Error!", err)
        },
        oncleanup: () => {
            // PeerConnection with the plugin closed, clean the UI
            // The plugin handle is still valid so we can create a new one
          log("sharePlugin oncleanup")
          localScreenShare.value = null
          shareTrack.value = null
        },
        ondetached: () => {
            // Connection with the plugin closed, get rid of its features
            // The plugin handle is not valid anymore
          log("sharePlugin ondetached plugin")
        }
    })
  }

  function stopStreaming() {
    if (plugin.value) {
      plugin.value.send({ 
        message: { request: 'unpublish' },
        success: () => {
          if (janus.value) {
            janus.value.destroy()
          }
        }
      })
    }
  }

  function joinToShare(stream) {
    log("sharePlugin Joined room! Creating offer...");

    let mediaConfig = [];

    if (stream === null || typeof stream === "undefined") {
      mediaConfig = [
          { type: 'screen', capture: false, recv: false, add: false },
      ]
    } else {
      mediaConfig = [
          { type: 'screen', capture: true, recv: false, add: true },
      ]
    }

    log("sharePlugin Media Configuration for Publisher set! ->");
    log(mediaConfig);

    sharePlugin.value.createOffer({
      tracks: mediaConfig,
      success: (sdpAnswer) => {
        // SDP Offer answered, publish our stream
        log("sharePlugin Offer answered! Start publishing...")
        let publish = {
          request: "configure",
          audio: true,
          video: true,
          data: true
        };
        sharePlugin.value.send({ message: publish, jsep: sdpAnswer })
      },
      error: (error) => {
        onError("sharePlugin createOffer error", error)
      }
    })
  }

  function onLocalStream(track) {
    log("onLocalStream", track)

    if (track.kind === "audio") {
      // New audio track: create a stream out of it, and use a hidden <audio> element
      // let audioStream = new MediaStream([track]);
      // log("Created remote audio stream:", audioStream);
      // Janus.attachMediaStream(audio.value, stream);
      // audio.value.srcObject = audioStream
    } else {
      // New video track: create a stream out of it
      let videoStream = new MediaStream([track]);
      log("Created local video stream:", videoStream);
      streamVideos.value[track["id"]] = videoStream
    }

    isStreaming.value = true
  }

  function onCleanup() {
    let tracks = localStream.value.getTracks()

    tracks.forEach(track => { track.stop() })

    if (localScreenShare.value) {
      let streamTracks = localScreenShare.value.getTracks()
      streamTracks.forEach(track => { track.stop() })
    }

    isStreaming.value = false
    streamVideos.value = {}
    localStream.value = null
    localScreenShare.value = null
  }

  Janus.init({
    debug: true,
    dependencies: Janus.useDefaultDependencies({ adapter: adapter }),
    callback: () => {
      log("Janus.init success")
    }
  })
</script>

<template>
  <div class="stream">
    <button @click="startStreaming" v-if="!isStreaming">Start Streaming</button>
    <button @click="stopStreaming" v-else>Stop Streaming</button>
    <button @click="startShare" v-if="isStreaming && localScreenShare == null">Start Share</button>
    <button @click="stopShare" v-if="isStreaming && localScreenShare != null">Stop Share</button>
    <div>
      <video v-for="(video, index) in streamVideos" :id="index" :srcObject="video" width="100%" autoplay muted playsinline />
    </div>
  </div>
</template>