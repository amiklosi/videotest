<template>
  <div>
    <video
      ref="videoRec"
      :srcObject.prop="stream"
      class="camera"
      muted
      loop
      controls
      autoplay
    />
    <br />
    <video ref="videoPlay" class="camera" muted loop controls autoplay />
    <br />
    Resolution: {{ resolution }}<br />
    Size: {{ blobSize }}<br />
    Time: {{ elapsedTime }}
    <br />
    <button @click="startRecording" :disabled="!ready || recording">
      Record
    </button>
    <button @click="stopRecording" :disabled="!recording">Stop</button>
    <button @click="saveRecording" :disabled="!recordingReady">Download</button>
  </div>
</template>

<script>
import { saveAs } from "file-saver";

const resolutionObj = { width: 240, height: 320 };

export default {
  name: "VideoRecorder",
  data() {
    return {
      videoUrl: null,
      isValid: true,
      stream: undefined,
      blob: undefined,
      recorder: undefined,
      chunks: [],
      ready: false,
      recording: false,
      recordingReady: false,
      startTime: 0,
      endTime: 0,
      resolution: `${resolutionObj.width}x${resolutionObj.height}`,
      db: undefined,
    };
  },
  computed: {
    blobSize: function () {
      let size = 0;
      (this.chunks || []).forEach((ch) => (size += ch.size));
      console.log(this);
      return (size / 1024 / 1024).toFixed(2);
    },
    elapsedTime: function () {
      if (this.startTime == 0) return 0;
      console.log("q", this.startTime, this.endTime, this.chunks);
      return ((this.endTime || new Date().getTime()) - this.startTime) / 1000;
    },
  },
  mounted() {
    navigator.mediaDevices
      .getUserMedia({
        video: {
          width: { ideal: resolutionObj.width },
          height: { ideal: resolutionObj.height },
          // frameRate: { ideal: 10, max: 15 },
        },
        audio: true,
      })
      .then(this.gotStream)
      .catch(() => (this.isValid = false));

    let request = window.indexedDB.open("mw_videos", 1);

    request.onerror = function () {
      console.log("Database failed to open");
    };

    request.onsuccess = () => {
      console.log("Database opened succesfully");
      this.db = request.result;
      this.getVideoFromDb();
    };

    request.onupgradeneeded = (e) => {
      let db = e.target.result;

      let objectStore = db.createObjectStore("videos_os", { keyPath: "name" });
      objectStore.createIndex("video", "video", { unique: false });
      console.log("Database setup complete");
    };
  },
  destroyed() {
    this.stream.getTracks().forEach(function (track) {
      track.stop();
    });
  },
  methods: {
    getVideoFromDb() {
      let objectStore = this.db
        .transaction("videos_os")
        .objectStore("videos_os");
      let request = objectStore.get("video");
      request.onsuccess = () => {
        if (request.result) {
          console.log("taking videos from IDB", request.result);
          this.blob = window.URL.createObjectURL(request.result.video);
        } else {
          console.log("not in the db");
        }
      };
    },

    gotStream(mediaStream) {
      console.log("got stream", mediaStream);
      this.stream = mediaStream;
      this.recorder = new MediaRecorder(mediaStream);
      this.recorder.ondataavailable = this.videoRecorderDataHandler;
      this.recorder.onstop = this.videoRecorderStopHandler;
      console.log(this.$refs.videoRec);
      // this.$refs.videoRec.src = null;
      // this.$refs.videoRec.srcObject = mediaStream;
      this.ready = true;
    },
    videoRecorderDataHandler(event) {
      this.chunks.push(event.data);
      console.log(event);
    },
    videoRecorderStopHandler() {
      const blob = new Blob(this.chunks, { type: this.recorder.mimeType });
      console.log("Stopped!", blob);
      this.$refs.videoPlay.src = window.URL.createObjectURL(blob);

      this.recording = false;
      this.recordingReady = true;

      let objectStore = this.db
        .transaction(["videos_os"], "readwrite")
        .objectStore("videos_os");

      let record = {
        video: blob,
        name: "video",
      };

      let request = objectStore.put(record);
      request.onsuccess = function () {
        console.log("Record addition attempt finished");
      };

      request.onerror = function () {
        console.log(request.error);
      };
    },
    startRecording() {
      this.chunks = [];
      this.recorder.start(1000);
      this.recording = true;
      this.recordingReady = false;
      this.startTime = new Date().getTime();
      this.endTime = 0;
    },
    stopRecording() {
      this.recorder.stop();
      this.endTime = new Date().getTime();
    },
    saveRecording() {
      const blob = new Blob(this.chunks, { type: this.recorder.mimeType });
      saveAs(blob, `video.webm`);
    },
  },
};
</script>

<style></style>

<style lang="scss">
.camera {
  width: 240px;
  height: 320px;
}
</style>
