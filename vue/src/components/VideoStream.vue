<script setup>
  import { ref } from "vue";
  import { log } from "@/services/utility.js"; // logging browser service
  import adapter from 'webrtc-adapter';
  import Janus from 'janus-gateway'

  // const JANUS_URL = 'http://localhost:8088/janus'
  const JANUS_URL = ['ws://localhost:8188', 'http://localhost:8088/janus']
  const JANUS_ADMIN = "http://localhost:7088/admin"

  let janus = ref(null)
  let plugin = ref(null)
  var opaqueId = ref(Janus.randomString(12))
  let stream = ref()
  let streamVideos = ref({})
  let audio = ref()
  let error = ref(null)
  let status = ref("not connected")
  let roomId = ref(1234)
  let isViewing = ref(false)
  let myid = ref(null)
  let mypvtid = ref(null)
  let isConnected = ref(false)

  function connect(server, token = "") {
    janus.value = new Janus({
      server,
      token: token,
      success: () => {
        attachPlugin()
        status.value = "connected"
        isConnected.value = true
      },
      error: (error) => {
        onError('Failed to connect to janus server', error)
      },
      destroyed: () => {
        isConnected.value = false
        window.location.reload()
      }
    })
  }

  async function saveToken() {
    try {
      let data = {
        "janus" : "add_token",
        "token": "testToken1",
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
        "token": "testToken1",
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

  function attachPlugin() {
    // Attach the videoroom plugin
    janus.value.attach({
      plugin: "janus.plugin.videoroom",
      opaqueId: opaqueId.value,
      success: (pluginHandle) => {
        log("Remote Stream Plugin attached.")
        log(pluginHandle)

        plugin.value = pluginHandle;
        plugin.value.send({
          message: {
            request: "join",
            room: roomId.value,
            feed: 54321,
            ptype: "publisher", // "subscriber", // https://groups.google.com/g/meetecho-janus/c/K1CJsf47nPg
            display: "subscriber",
            pin: "123123"
          },
          success: (data) => {
            log("joining video room success", data)
          },
          error: (error) => {
            onError('Error joining video room:', error)
          },
        });
      },
      onmessage: (message, jsep) => {
        log("onmessage publisher", message, jsep)

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

          if (message.error_code !== undefined && message.error_code === 428) {
            status.value = "Stream not started"
          }

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

        if(jsep) {
          Janus.debug("Handling SDP as well...", jsep);
          plugin.value.handleRemoteJsep({ jsep: jsep });
        }
      },
      error: (err) => {
        log("publisher Remote Feed: Janus VideoRoom Plugin Error!", true)
        log(err, true)
        onError('Error attaching plugin... ', err)
      },
      webrtcState: function(on) {
        Janus.log("publisher Janus says our WebRTC PeerConnection is " + (on ? "up" : "down") + " now");
        if (!on)
          return;

        // plugin.value.send({ message: { request: "configure", bitrate: 2000000 }});
      },
      iceState: (state) => {
        log("publisher iceState", state)
        status.value = state
      },
      mediaState: (medium, receiving, mid) => {
        log("publisher mediaState", medium, receiving, mid)
      },
      onlocaltrack: (local, on) => {
        log("publisher onlocaltrack", local, on)
      },
      onremotetrack: (s, mid, on, metadata) => {
        log("publisher onremotetrack", s, mid, on, metadata)

        if (!on) {
          if(s.kind === "video") {
            stream.value.srcObject = null
            isViewing.value = false
          }

          return
        }

        onRemoteStream(s)
      },
      oncleanup: () => {
        onCleanup()
      },
      ondetached: () => {
        log("publisher plugin: 'janus.plugin.videoroom'", "ondetached")
      }
    })
  }

  function newRemoteFeed(id, display, streams) {
    // A new feed has been published, create a new plugin handle and attach to it as a subscriber
    let remoteFeed = null;

    janus.value.attach({
        plugin: "janus.plugin.videoroom",
        opaqueId: opaqueId.value,
        success: function(pluginHandle) {
          remoteFeed = pluginHandle;
          remoteFeed.remoteTracks = {};
          remoteFeed.remoteVideos = 0;
          remoteFeed.simulcastStarted = false;
          remoteFeed.svcStarted = false;
          Janus.log("Plugin attached! (" + remoteFeed.getPlugin() + ", id=" + remoteFeed.getId() + ")");
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
            remoteFeed.rfid = stream.id;
            remoteFeed.rfdisplay = stream.display;
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
          remoteFeed.send({ message: subscribe });
        },
        error: function(error) {
          log("subscriber  -- Error attaching plugin...", error)
          onError("subscriber -- Error attaching plugin...", error)
        },
        iceState: function(state) {
          Janus.log("subscriber ICE state (feed #" + remoteFeed.rfid + ") changed to " + state);
          status.value = state
        },
        webrtcState: function(on) {
          log("subscriber Janus says this WebRTC PeerConnection (feed #" + remoteFeed.rfid + ") is " + (on ? "up" : "down") + " now");
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
              if (msg.error_code !== undefined && msg.error_code === 428) {
                status.value = "Stream not started"
              }
              // Check if we got a simulcast-related event from this publisher
              let substream = msg["substream"];
              let temporal = msg["temporal"];
              if((substream !== null && substream !== undefined) || (temporal !== null && temporal !== undefined)) {
                if(!remoteFeed.simulcastStarted) {
                  remoteFeed.simulcastStarted = true;
                }
              }
              // Or maybe SVC?
              let spatial = msg["spatial_layer"];
              temporal = msg["temporal_layer"];
              if((spatial !== null && spatial !== undefined) || (temporal !== null && temporal !== undefined)) {
                if(!remoteFeed.svcStarted) {
                  remoteFeed.svcStarted = true;
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
            remoteFeed.createAnswer(
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
                  remoteFeed.send({ message: body, jsep: jsep });
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
        },
        onremotetrack: function(track, mid, on, metadata) {
          log(
            "Remote feed #" + remoteFeed.rfid +
            ", remote track (mid=" + mid + ") " +
            (on ? "added" : "removed") +
            (metadata? " (" + metadata.reason + ") ": "") + ":", track
          );
          if(!on) {
            if(track.kind === "video") {
              delete streamVideos.value[track["id"]]
              
              isViewing.value = Object.keys(streamVideos.value).length > 0
            }

            return
          }

          onRemoteStream(track)
        },
        oncleanup: function() {
          log(" ::: Got a cleanup notification (remote feed " + id + ") :::");
          onCleanup()
        }
      })
  }

  function start() {
    connect(JANUS_URL)
  }

  function startWithToken() {
    connect(JANUS_URL, "testToken1")
  }

  function stop() {
    plugin.value.send({ 
      message: { request: "leave" },
      success: () => {
        // Логіка для обробки успішного відключення від кімнати
        console.log('Left video room successfully');
        onCleanup()
      },
      error: (error) => {
        console.error('Error leaving video room:', error);
      },
    })
  }

  function onRemoteStream(track) {
    log("onRemoteStream", track)

    if (track.kind === "audio") {
      // New audio track: create a stream out of it, and use a hidden <audio> element
      let audioStream = new MediaStream([track]);
      log("Created remote audio stream:", audioStream);
      // Janus.attachMediaStream(audio.value, stream);
      audio.value.srcObject = audioStream
    } else {
      // New video track: create a stream out of it
      let videoStream = new MediaStream([track]);
      log("Created remote video stream:", videoStream);
      // Janus.attachMediaStream(streamVideos.value[track["id"]], videoStream);
      streamVideos.value[track["id"]] = videoStream
    }

    isViewing.value = true
  }

  function onCleanup() {
    streamVideos.value = []
    audio.value.srcObject = null
    status.value = "disconnected"
    error.value = null
    isViewing.value = false
    isConnected.value = false
  }

  function onError(message, e='') {
    Janus.error(message, e)
    error.value = message + e
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
  <div>
    <div v-if="status">Status: {{ status }}</div>
    <button @click="saveToken" v-if="!isConnected">Save token</button>
    <button @click="removeToken" v-if="!isConnected">Delete token</button>
    <br v-if="!isConnected" />
    <button @click="start" v-if="!isConnected">Connect</button>
    <button @click="stop" v-else>Disconnect</button>
    <button @click="startWithToken" v-if="!isConnected">Connect with token</button>
    <div v-if="error">{{ error }}</div>
    <div>
      <!-- <video ref="stream" autoplay playsinline /> -->
      <video v-for="(video, index) in streamVideos" :id="index" :srcObject="video" width="100%" autoplay playsinline />
      <audio ref="audio" hidden autoplay playsinline />
    </div>
  </div>
</template>