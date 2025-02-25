<script setup lang="ts">
import { useFileDialog } from "@vueuse/core";
import { ref, type Ref } from "vue";

const configuration = {
  iceServers: [{ urls: "stun:freestun.net:3478" }],
};

const peerConnection = new RTCPeerConnection(configuration);
let dataChannel = peerConnection.createDataChannel("data");

const isConnected = ref(false);

dataChannel.addEventListener("open", (event) => {
  isConnected.value = true;
});

dataChannel.addEventListener("closing", (event) => {
  isConnected.value = false;
});

dataChannel.addEventListener("error", (event) => {
  isConnected.value = false;
});

dataChannel.addEventListener("message", handleMessage);

console.log("peerConnection:", peerConnection);
const offer = await peerConnection.createOffer();
await peerConnection.setLocalDescription(offer);

peerConnection.addEventListener("icecandidate", (event) => {
  console.log("got ice candidate", event);
});

const offerIsFinished = ref(false);

peerConnection.addEventListener("icegatheringstatechange", (event) => {
  if (peerConnection.iceGatheringState === "complete") {
    console.log(
      "All ICE candidates gathered:",
      event,
      peerConnection.localDescription
    );
    // Now send the offer to the other party
    // console.log(peerConnection.localDescription);
    offerIsFinished.value = true;
  }
});

console.log("offer:", offer);

function copyOffer() {
  console.log("copying offer");
  window.navigator.clipboard.writeText(
    JSON.stringify(peerConnection.localDescription?.toJSON())
  );
}

const remoteOfferInput = ref("");
const remoteAnswerInput = ref("");

async function respondToRemoteOffer() {
  await peerConnection.setRemoteDescription(JSON.parse(remoteOfferInput.value));
  await peerConnection.setLocalDescription();
  await new Promise<void>((resolve, reject) => {
    peerConnection.addEventListener("icegatheringstatechange", (event) => {
      if (peerConnection.iceGatheringState === "complete") {
        resolve();
      }
    });
  });
  window.navigator.clipboard.writeText(
    JSON.stringify(peerConnection.localDescription?.toJSON())
  );
  peerConnection.addEventListener("datachannel", (event) => {
    console.log("YAY 2");
    dataChannel = event.channel;
    dataChannel.addEventListener("message", handleMessage);
  });
}

async function connectWithRemoteAnswer() {
  await peerConnection.setRemoteDescription(
    JSON.parse(remoteAnswerInput.value)
  );
}

const recievedFileMeta: Ref<any> =
  ref(undefined);

let receiveBuffer: Array<Blob> = [];
let receiveSize = 0;

async function handleMessage(event: MessageEvent) {
  console.log("Recieved message:", event.data);
  if (typeof event.data === "string") {
    const json = JSON.parse(event.data);
    recievedFileMeta.value = json;
    console.log(recievedFileMeta.value);
    receiveBuffer = [];
    receiveSize = 0;
  } else {
    if (!recievedFileMeta.value)
      return console.log("received message without filemeta", event.data);
    receiveBuffer.push(event.data);
    receiveSize += (event.data as ArrayBuffer).byteLength;
    console.log(`size`, receiveSize, 'of', recievedFileMeta.value);
    if (receiveSize >= recievedFileMeta.value.size) {
      if (receiveSize > recievedFileMeta.value.size) console.log("size is bigger????");
      const received = new Blob(receiveBuffer);
      receiveBuffer = [];
      URL.createObjectURL(received);
      let blobUrl = URL.createObjectURL(received);
      let a = document.createElement("a");
      a.href = blobUrl;
      a.download = recievedFileMeta.value.name
      a.click();
      URL.revokeObjectURL(blobUrl);
      console.log("Done 🏁");
    }
  }
}

async function sendMessage() {
  const payload = `${Math.random()}`;
  console.log("Sending Message:", payload);
  dataChannel.send(payload);
}

const { files, open, reset, onCancel, onChange } = useFileDialog({
  // accept: 'image/*', // Set to accept only image files
  directory: false, // Select directories instead of files if set true
});

const selectedFile: Ref<undefined | File> = ref();

onChange((files) => {
  if (!files) return (selectedFile.value = undefined);
  const file = files[0];
  selectedFile.value = file;
});

onCancel(() => {
  /** do something on cancel */
});

async function sendSelectedFile() {
  if (!selectedFile.value) return;
  const file = selectedFile.value;

  console.log("sending", file);
  const metadata = {
    name: file.name,
    size: file.size,
  };

  dataChannel.send(JSON.stringify(metadata));
  console.log("Sent data (JSON)");

  let sliceStart = 0;
  const sliceSize = 16 * 1024;

  dataChannel.addEventListener("bufferedamountlow", bufferedAmountLow);

  function bufferedAmountLow() {
    console.log("Sent data (Blob)", sliceStart, sliceSize);
    dataChannel.send(file.slice(sliceStart, (sliceStart += sliceSize)));
    if (sliceStart > file.size)
      dataChannel.removeEventListener("bufferedamountlow", bufferedAmountLow);
  }
}
</script>

<template>
  <main v-if="!isConnected">
    <p>Trying out WebRTC.</p>
    <button @click="copyOffer" class="bg-amber-200 p-1 rounded">
      Copy offer: {{ offerIsFinished ? "Offer ready" : "Making offer..." }}
    </button>

    <br />
    <textarea v-model="remoteOfferInput"></textarea>
    <button @click="respondToRemoteOffer">Copy answer</button>
    <br />
    <textarea v-model="remoteAnswerInput"></textarea>
    <button @click="connectWithRemoteAnswer">Connect to answer</button>
    <br />
    <button @click="sendMessage">Send hello message</button>

    <!-- <code>{{ peerConnection.localDescription?.sdp }}</code> -->
  </main>
  <main v-else>
    <h1>Connected 📱</h1>
    <button @click="open()">Select file</button>
    <p v-if="selectedFile">{{ selectedFile.name }}</p>
    <button v-if="selectedFile" @click="sendSelectedFile">
      Send this file
    </button>
  </main>
</template>
