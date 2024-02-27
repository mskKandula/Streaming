<template>
  <div id="app">
    <div class="mt-5 mb-5">
      <t-button
        v-on:click="getStream"
        v-if="!isRecording"
        v-show="canRecord"
        class="ml-10"
      >
        Start Recording 🎥</t-button
      >

      <div v-else>
        <t-button v-on:click="stopStream"> Stop Screen Recording ❌ </t-button>
      </div>
      <t-button @click="downloadImage">download</t-button>
    </div>
    <canvas id="canvas" hidden></canvas>
    <video
      class="center"
      height="500px"
      controls
      autoplay
      id="video"
      hidden
    ></video>
  </div>
</template>

<script>
import JSZip from "jszip";

export default {
  name: "Home",
  components: {},
  data() {
    return {
      youtube_ready: false,
      canRecord: true,
      isRecording: false,
      loaded: false,
      options: {
        audioBitsPerSecond: 128000,
        videoBitsPerSecond: 2500000,
      },
      displayOptions: {
        video: {
          cursor: "always",
        },
        audio: {
          echoCancellation: true,
          noiseSuppression: true,
          sampleRate: 44100,
        },
      },
      mediaRecorder: {},
      stream: {},
      recordedChunks: [],
      file: null,
      fileReady: false,
      sendEmail: "",
      bytes_processed: 0,
      yt_token: "",
      transcript: {},
      vidUrl: "",
      shareReady: false,
      videoTrack: null,
      vid: null,
      new_zip: null,
      blobsArray: [],
    };
  },
  methods: {
    downloadImage() {
      console.log("72");
      this.imageCapture = new ImageCapture(this.videoTrack);
      this.imageCapture.grabFrame().then((bitmap) => {
        // Stop sharing
        // track.stop();
        let canvas = document.getElementById("canvas");
        // Draw the bitmap to canvas
        canvas.width = bitmap.width;
        canvas.height = bitmap.height;
        canvas.getContext("2d").drawImage(bitmap, 0, 0);

        // Grab blob from canvas
        canvas.toBlob((blob) => {
          // Do things with blob here
          blob.name = `screenshot-${new Date().getTime()}`;
          this.blobsArray.push(blob);
          var image = document.createElement("img");
          image.setAttribute("style", "width: 150px; height: 150px;");

          // image.height="15px"
          let url = window.URL.createObjectURL(blob);
          image.src = url;

          document.body.appendChild(image);
          // a.download = blob.name;
          // a.click();
        });
      });
    },
    downloadZip() {
      for (let i = 0; i < this.blobsArray.length; i++) {
        this.new_zip.file("img" + i + ".png", this.blobsArray[i], {
          binary: true,
        });
      }
      this.new_zip
        .generateAsync({
          type: "blob",
        })
        .then(function (content) {
          var a = document.createElement("a");
          document.body.appendChild(a);
          a.style = "display: none";
          let url = window.URL.createObjectURL(content);
          a.href = url;
          a.download = "img_archives.zip";
          a.click();
          window.URL.revokeObjectURL(url);

          //  datafile.download = "DataFiles.zip";
          //                 datafile.href = window.URL.createObjectURL(zip.generate({ type: "blob" }));
          // saveAs(content, "img_archive.zip");
        });
    },

    async getStream() {
      const self = this;
      navigator.mediaDevices
        .getDisplayMedia(self.displayOptions)
        .then(async (stream) => {
          self.vid = document.getElementById("video");
          self.vid.srcObject = stream;
          console.log("93", stream.getVideoTracks()[0]);
          // Grab frame from stream
          self.videoTrack = stream.getVideoTracks()[0];
          console.log("113");
          // self.getscShot();
          for (let i = 0; i < 10; i++) {
            // alert(i)
            self.downloadImage();
            await new Promise((r) => setTimeout(r, 2000));
          }

          self.downloadZip();

          // setInterval(this.downloadImage(), 1000);
          // this.videoTrack.onmute((event) => {
          //   alert("screen captured");
          //   console.log("98", event);
          // });
          // track.addEventListener("eventHandler", alert("102"));

          // copy to clipboard
          // let data = [new window.ClipboardItem({ [blob.type]: blob })];
          // navigator.clipboard.write(data);
          // console.log('output blob:', blob);
        })

        .catch((e) => console.log(e));
    },
  },
  mounted() {
    this.new_zip = new JSZip();
    this.loaded = true;
    const ctx = this;
    window.addEventListener("message", function (e) {
      if (typeof e.data.youtube_token !== "undefined") {
        console.log(e.data.youtube_token);
        ctx.yt_token = e.data.youtube_token;
        ctx.setYouTube(e.data.youtube_token);
        ctx.youtube_ready = true;
      }
    });
    this.$gtag.pageview("/");
    const ua = navigator.userAgent;
    if (
      /(tablet|ipad|playbook|silk)|(android(?!.*mobi))/i.test(ua) ||
      /Mobile|Android|iP(hone|od)|IEMobile|BlackBerry|Kindle|Silk-Accelerated|(hpw|web)OS|Opera M(obi|ini)/.test(
        ua
      )
    ) {
      alert("You must be on desktop to use this application!");
      this.canRecord = false;
      this.$gtag.exception("mobile-device-attempt", {});
    }
    let that = this;
    if (
      Notification.permission !== "denied" ||
      Notification.permission === "default"
    ) {
      try {
        Notification.requestPermission().then(function (result) {
          that.$gtag.event("accepted-notifications", {
            event_category: "Notifications",
            event_label: "Notification accepted",
          });
          console.log(result);
        });
      } catch (error) {
        // Safari doesn't return a promise for requestPermissions and it
        // throws a TypeError. It takes a callback as the first argument
        // instead.
        if (error instanceof TypeError) {
          Notification.requestPermission((result) => {
            that.$gtag.event("accepted-notifications", {
              event_category: "Notifications",
              event_label: "Notification accepted",
            });
            console.log(result);
          });
        } else {
          this.$gtag.exception("notification-error", error);
          throw error;
        }
      }
    }
  },
  async created() {
    try {
      if (localStorage.youtube_key != null) {
        this.setYouTube(localStorage.youtube_key);
        this.youtube_ready = true;
      }
      const registration = await navigator.serviceWorker.ready;
      const tags = await registration.periodicSync.getTags();
      navigator.serviceWorker.addEventListener("message", (event) => {
        this.bytes_processed = event.data;
      });
      if (tags.includes("get-latest-stats")) {
        // this.skipDownloadUseCache()
      } else {
        this.getBytes();
      }
    } catch (e) {
      this.$gtag.exception("application-error", e);
      this.getBytes();
    }
  },
};
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
:picture-in-picture {
  box-shadow: 0 0 0 5px red;
  height: 500px;
  width: 500px;
}
</style>
