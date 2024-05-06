<script setup>
  import { ref } from "vue"
  import { log } from "@/services/utility.js"
  import adapter from 'webrtc-adapter'
  import Janus from 'janus-gateway'
  import appleLogo from '@/assets/images/appleLogo.png'

  // const JANUS_URL = 'http://localhost:8088/janus'
  const JANUS_URL = ['ws://localhost:8188', 'http://localhost:8088/janus']
  const JANUS_ADMIN = "http://localhost:7088/admin"

  let janus = ref(null)
  let janusStream = ref(null)
  let plugin = ref(null)
  let sharePlugin = ref(null)
  let localStream = ref(null)
  let localVideo = ref(null)
  let localScreenShare = ref(null)
  let roomId = ref(1234)
  let error = ref(null)
  let isStreaming = ref(false)
  let localStreamVideos = ref({})
  let streamVideos = ref({})
  let audio = ref({})
  let shareTrack = ref(null)
  let myid = ref(null)
  let mypvtid = ref(null)
  let canvas = ref(null)
  var opaqueId = ref(Janus.randomString(12))

  async function startStreaming() {
    try {
      localStream.value = await navigator.mediaDevices.getUserMedia({
        audio: true,
        video: {
          width: { ideal: 640 },
          height: { ideal: 480 }
        }
      })

      janus.value = new Janus({
        server: JANUS_URL,
        success: () => {
          attachStreamingPlugin(localStream.value)
        },
        error: (error) => {
          console.error('Failed to connect to janus server', error)
          onError('Failed to connect to janus server', error)
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

  async function saveToken() {
    try {
      let data = {
        "janus" : "add_token",
        "token": "testToken",
        "transaction" : Janus.randomString(12),
        "plugins": [
                "janus.plugin.streaming",
                "janus.plugin.videoroom"
        ],
        "admin_secret": "janusoverlord"
      }
      let response = await fetch(JANUS_ADMIN, {
        method: "POST",
        body: JSON.stringify(data)
      })

      let resp = response.json()

      console.log("saveToken response ", resp)
    } catch (error) {
      onError('Error accessing camera:', error)
    }
  }

  async function removeToken() {
    try {
      let data = {
        "janus" : "remove_token",
        "token": "testToken",
        "transaction" : Janus.randomString(12),
        "admin_secret": "janusoverlord"
      }
      let response = await fetch(JANUS_ADMIN, {
        method: "POST",
        body: JSON.stringify(data)
      })

      let resp = response.json()

      console.log("removeToken response ", resp)
    } catch (error) {
      onError('Error accessing camera:', error)
    }
  }

  async function startStreamingWithToken() {
    try {
      localStream.value = await navigator.mediaDevices.getUserMedia({
        audio: true,
        video: {
          width: { ideal: 640 },
          height: { ideal: 480 }
        }
      })

      janus.value = new Janus({
        server: JANUS_URL,
        token: "testToken",
        success: () => {
          attachStreamingPlugin(localStream.value)
        },
        error: (error) => {
          console.error('Failed to connect to janus server', error)
          onError('Failed to connect to janus server', error)
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
      log('Error accessing screen:', error)
    }
  }

  function stopShare() {
    if (shareTrack.value) {
      log("stopShare", shareTrack.value)
      shareTrack.value.stop()
      delete localStreamVideos.value[shareTrack.value["id"]]
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
      opaqueId: opaqueId.value,
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
            let fps = 30
            log("Joined room! Creating offer...");
            localVideo.value.srcObject = stream
            // localVideo.value.muted = "muted"
            localVideo.value.play()
            let myText = "Hi there!"
            let myColor = "white"
            let myFont = "20pt Calibri"
            let myX = 15, myY = 460
            let logoW = 500, logoH = 132
            let logoS = 0.4
            let logoX = 640 - logoW * logoS - 15, logoY = 15
            let newCanvas = canvas.value
            const context = newCanvas.getContext('2d')
            const logo = new Image()
            logo.crossOrigin = "Anonymous"
            // logo.src = "https://janus.conf.meetecho.com/meetecho-logo.png"
            logo.src = appleLogo

            const draw = () => {
              if (!localVideo.value.paused && !localVideo.value.ended) {
                context.drawImage(localVideo.value, 0, 0, newCanvas.width, newCanvas.height)

                if (logo.complete && logo.naturalHeight !== 0) {
                  context.drawImage(logo, 20, 20, 50, 50)
                  // context.drawImage(logo,
                  //     0, 0, logoW, logoH,
                  //     logoX, logoY, logoW * logoS, logoH * logoS)
                }

                context.fillStyle = 'rgba(0,0,0,0.5)'
                context.fillRect(0, 420, 640, 480)
                context.font = myFont
                context.fillStyle = myColor
                context.fillText(myText, myX, myY)

                // requestAnimationFrame(draw);
                setTimeout(draw, 1000 / fps)
              }
            }

            logo.onload = draw

            let newStream = newCanvas.captureStream(30)

            log("newStream", newStream)
            log("newStream.getVideoTracks", newStream.getVideoTracks())

            let mediaConfig = []
            // let body = { audio: true, video: true }
            // plugin.value.send({ message: body })

            if (stream === null || typeof stream === "undefined") {
              mediaConfig = [
                {type: 'audio', capture: false, recv: false, add: false},
                {type: 'video', capture: false, recv: false, add: false},
              ]
            } else {
              log("canvasStream.value.getAudioTracks()[0]", newStream.getAudioTracks()[0])
              log("canvasStream.value.getVideoTracks()[0]", newStream.getVideoTracks()[0] instanceof MediaStreamTrack)

              mediaConfig.push({
                type: 'audio',
                capture: true,
                recv: false,
                add: true
              })

              if (newStream.getVideoTracks().length > 0)
                mediaConfig.push({
                  type: 'video',
                  capture: newStream.getVideoTracks()[0],
                  recv: false,
                  add: true
                })
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
                plugin.value.send({message: publish, jsep: sdpAnswer})
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

          if (message.videoroom === "joined") {
            myid.value = message["id"];
            mypvtid.value = message["private_id"];
            Janus.log("Successfully joined room " + message["room"] + " with ID " + message["id"]);

            if (message["publishers"]) {
              let list = message["publishers"];
              Janus.debug("Got a list of available publishers/feeds:", list);
              for(let f in list) {
                if(list[f]["dummy"])
                  continue;
                let id = list[f]["id"];
                let streams = list[f]["streams"];
                let display = list[f]["display"];
                for(let i in streams) {
                  let stream = streams[i];
                  stream["id"] = id;
                  stream["display"] = display;
                }
                log("  >> [" + id + "] " + display + ":", streams);
                newRemoteFeed(id, display, streams);
              }
            }
          }

          if (message.videoroom === "event") {
            log("videoroom === event")

            if (message["publishers"]) {
              let list = message["publishers"];
              log("Got a list of available publishers/feeds:", list);
              for(let f in list) {
                if(list[f]["dummy"])
                  continue;

                let id = list[f]["id"];
                let streams = list[f]["streams"];
                let display = list[f]["display"];

                for(let i in streams) {
                  let stream = streams[i];

                  stream["id"] = id;
                  stream["display"] = display;
                }
                log("  >> [" + id + "] " + display + ":", streams);
                newRemoteFeed(id, display, streams);
              }
            }
          }
        },
        onlocaltrack: (track, on) => {
          log("Successfully published local track: ", on, track);
          if(!on) {
            if(track.kind === "video") {
              delete localStreamVideos.value[track["id"]]
              
              isStreaming.value = Object.keys(localStreamVideos.value).length > 0
            }

            return
          }

          onLocalStream(track)
        },
        onremotetrack: (s, mid, on, metadata) => {
          log("onremotetrack", s, mid, on, metadata)

          if (!on) {
            if(s.kind === "video") {
              delete streamVideos.value[s["id"]]
            }

            if (s.kind === "audio") {
              delete audio.value[s["id"]]
            }

            return
          }

          onRemoteStream(s)
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

  function newRemoteFeed(id, display, streams) {
    // A new feed has been published, create a new plugin handle and attach to it as a subscriber
    let subscriberPlugin = null;

    janus.value.attach({
      plugin: "janus.plugin.videoroom",
      opaqueId: opaqueId.value,
      success: function(pluginHandle) {
        subscriberPlugin = pluginHandle;
        subscriberPlugin.remoteTracks = {};
        subscriberPlugin.remoteVideos = 0;
        subscriberPlugin.simulcastStarted = false;
        subscriberPlugin.svcStarted = false;
        Janus.log("Plugin attached! (" + subscriberPlugin.getPlugin() + ", id=" + subscriberPlugin.getId() + ")");
        Janus.log("  -- This is a subscriber");
        // Prepare the streams to subscribe to, as an array: we have the list of
        // streams the feed is publishing, so we can choose what to pick or skip
        let subscription = [];
        for(let i in streams) {
          let stream = streams[i];
          // If the publisher is VP8/VP9 and this is an older Safari, let's avoid video
          if(stream.type === "video" && Janus.webRTCAdapter.browserDetails.browser === "safari" &&
              ((stream.codec === "vp9" && !Janus.safariVp9) || (stream.codec === "vp8" && !Janus.safariVp8))) {
            log("Publisher is using " + stream.codec.toUpperCase +
                ", but Safari doesn't support it: disabling video stream #" + stream.mindex);
            continue;
          }
          subscription.push({
            feed: stream.id,  // This is mandatory
            mid: stream.mid   // This is optional (all streams, if missing)
          });
          // FIXME Right now, this is always the same feed: in the future, it won't
          subscriberPlugin.rfid = stream.id;
          subscriberPlugin.rfdisplay = stream.display;
        }
        // We wait for the plugin to send us an offer
        let subscribe = {
          request: "join",
          room: roomId.value,
          ptype: "subscriber",
          streams: subscription,
          use_msid: false,
          private_id: mypvtid.value,
          pin: "123123"
        };
        subscriberPlugin.send({ message: subscribe });
      },
      error: function(error) {
        log("subscriber  -- Error attaching plugin...", error)
        onError("subscriber -- Error attaching plugin...", error)
      },
      iceState: function(state) {
        Janus.log("subscriber ICE state (feed #" + subscriberPlugin.rfid + ") changed to " + state);
        // status.value = state
      },
      webrtcState: function(on) {
        log("subscriber Janus says this WebRTC PeerConnection (feed #" + subscriberPlugin.rfid + ") is " + (on ? "up" : "down") + " now");
      },
      slowLink: function(uplink, lost, mid) {
        log("subscriber Janus reports problems " + (uplink ? "sending" : "receiving") +
            " packets on mid " + mid + " (" + lost + " lost packets)");
      },
      onmessage: function(msg, jsep) {
        log(" ::: Got a message (subscriber) :::", msg)
        let event = msg["videoroom"]
        log("Event: " + event)
        if(msg["error"]) {
          log(msg["error"])
        } else if(event) {
          if(event === "attached") {
            // Subscriber created and attached
            log("Successfully attached to feed in room " + msg["room"]);
          } else if(event === "event") {
            // if (msg.error_code !== undefined && msg.error_code === 428) {
            //   status.value = "Stream not started"
            // }
            // Check if we got a simulcast-related event from this publisher
            let substream = msg["substream"];
            let temporal = msg["temporal"];
            if((substream !== null && substream !== undefined) || (temporal !== null && temporal !== undefined)) {
              if(!subscriberPlugin.simulcastStarted) {
                subscriberPlugin.simulcastStarted = true;
              }
            }
            // Or maybe SVC?
            let spatial = msg["spatial_layer"];
            temporal = msg["temporal_layer"];
            if((spatial !== null && spatial !== undefined) || (temporal !== null && temporal !== undefined)) {
              if(!subscriberPlugin.svcStarted) {
                subscriberPlugin.svcStarted = true;
              }
            }
          } else {
            // What has just happened?
          }
        }

        if(jsep) {
          log("Handling SDP as well...", jsep);
          let stereo = (jsep.sdp.indexOf("stereo=1") !== -1);
          // Answer and attach
          subscriberPlugin.createAnswer(
              {
                jsep: jsep,
                // We only specify data channels here, as this way in
                // case they were offered we'll enable them. Since we
                // don't mention audio or video tracks, we autoaccept them
                // as recvonly (since we won't capture anything ourselves)
                tracks: [
                  { type: 'data' }
                ],
                customizeSdp: function(jsep) {
                  if(stereo && jsep.sdp.indexOf("stereo=1") == -1) {
                    // Make sure that our offer contains stereo too
                    jsep.sdp = jsep.sdp.replace("useinbandfec=1", "useinbandfec=1;stereo=1");
                  }
                },
                success: function(jsep) {
                  log("Got SDP!", jsep);
                  let body = { request: "start", room: roomId.value };
                  subscriberPlugin.send({ message: body, jsep: jsep });
                },
                error: function(error) {
                  onError("WebRTC error: ", error);
                }
              });
        }
      },
      // eslint-disable-next-line no-unused-vars
      onlocaltrack: function(track, on) {
        // The subscriber stream is recvonly, we don't expect anything here
        log("subscriber onlocaltrack", track, on)
      },
      onremotetrack: function(track, mid, on, metadata) {
        log(
            "Remote feed #" + subscriberPlugin.rfid +
            ", remote track (mid=" + mid + ") " +
            (on ? "added" : "removed") +
            (metadata? " (" + metadata.reason + ") ": "") + ":", track
        );
        if (!on) {
          if (track.kind === "video") {
            delete streamVideos.value[track["id"]]
          }

          if (track.kind === "audio") {
            delete audio.value[track["id"]]
          }

          return
        }

        onRemoteStream(track)
      },
      oncleanup: function() {
        // log(" ::: Got a cleanup notification (remote feed " + id + ") :::");
        onCleanup()
      }
    })
  }

  function attachSharePlugin(stream) {
    janusStream.value.attach({
      plugin: "janus.plugin.videoroom",
      opaqueId: opaqueId.value,
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
              delete localStreamVideos.value[track["id"]]
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

  function onError(message, e='') {
    Janus.error(message, e)
    error.value = message + " - " + e
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
        localScreenShare.value = null
        stopShare()
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
      localStreamVideos.value[track["id"]] = videoStream
    }

    isStreaming.value = true
  }

  function onRemoteStream(track) {
    log("onRemoteStream", track)

    if (track.kind === "audio") {
      // New audio track: create a stream out of it, and use a hidden <audio> element
      let audioStream = new MediaStream([track]);
      log("Created remote audio stream:", audioStream);
      // Janus.attachMediaStream(audio.value, stream);
      audio.value[track["id"]] = audioStream
    } else {
      // New video track: create a stream out of it
      let videoStream = new MediaStream([track]);
      log("Created remote video stream:", videoStream);
      // Janus.attachMediaStream(streamVideos.value[track["id"]], videoStream);
      streamVideos.value[track["id"]] = videoStream
    }
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
    localStreamVideos.value = {}
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
    <button @click="saveToken" v-if="!isStreaming">Save token</button>
    <button @click="removeToken" v-if="!isStreaming">Delete token</button>
    <br v-if="!isStreaming" />
    <button @click="startStreaming" v-if="!isStreaming">Start Streaming</button>
    <button @click="stopStreaming" v-else>Stop Streaming</button>
    <button @click="startStreamingWithToken" v-if="!isStreaming">Start Streaming with token</button>
    <button @click="startShare" v-if="isStreaming && localScreenShare == null">Start Share</button>
    <button @click="stopShare" v-if="isStreaming && localScreenShare != null">Stop Share</button>
    <div v-if="error">{{ error }}</div>
    <canvas ref="canvas" width="640" height="480" style="margin: auto; padding: 0; display: none;" />
    <video ref="localVideo" autoplay muted playsinline style="display: none;" />
    <div>
      <div v-for="(item, index) in localStreamVideos">
        <video :id="index" :srcObject="item" width="100%" autoplay muted playsinline />
      </div>
      <div v-for="(item, index) in streamVideos">
        <video :id="index" :srcObject="item" width="100%" autoplay muted playsinline />
      </div>
      <audio v-for="(item, index) in audio" :id="index" :srcObject="item" hidden autoplay playsinline />
    </div>
  </div>
</template>