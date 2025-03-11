<script setup lang="ts">
import { useFileDialog } from "@vueuse/core";
import { ref, type Ref } from "vue";
import ProgressBar from "@/components/ProgressBar.vue";

const configuration = {
  iceServers: [
    { urls: "stun:freestun.net:3478" },
    { urls: "turn:freestun.net:3478", username: "free", credential: "free" },
  ],
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

const offerWasCopied = ref(false);
const answerWasCopied = ref(false);

function copyOffer() {
  console.log("copying offer");
  offerWasCopied.value = true;
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
  answerWasCopied.value = true;
  peerConnection.addEventListener("datachannel", (event) => {
    dataChannel = event.channel;
    dataChannel.addEventListener("message", handleMessage);
  });
}

async function connectWithRemoteAnswer() {
  await peerConnection.setRemoteDescription(
    JSON.parse(remoteAnswerInput.value)
  );
}

const recievedFileMeta: Ref<any> = ref(undefined);

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
    console.log(`size`, receiveSize, "of", recievedFileMeta.value);
    if (receiveSize >= recievedFileMeta.value.size) {
      if (receiveSize > recievedFileMeta.value.size)
        console.log("size is bigger????");
      const received = new Blob(receiveBuffer);
      receiveBuffer = [];
      URL.createObjectURL(received);
      let blobUrl = URL.createObjectURL(received);
      let a = document.createElement("a");
      a.href = blobUrl;
      a.download = recievedFileMeta.value.name;
      a.click();
      URL.revokeObjectURL(blobUrl);
      console.log("Done 🏁");
    }
  }
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

const sendingPercentage: Ref<undefined | number> = ref(undefined);

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

  bufferedAmountLow();

  function bufferedAmountLow(event?: any) {
    console.log("Sent data (Blob)", sliceStart, sliceSize, event);
    dataChannel.send(file.slice(sliceStart, (sliceStart += sliceSize)));
    sendingPercentage.value = sliceStart / file.size;
    if (sliceStart > file.size) {
      dataChannel.removeEventListener("bufferedamountlow", bufferedAmountLow);
      sendingPercentage.value = undefined;
    }
  }
}
</script>

<template>
  <div
    class="h-[100svh] grid place-items-center w-full *:bg-slate-50 *:shadow *:w-96 *:p-8 *:flex *:flex-col *:gap-4"
  >
    <main v-if="!isConnected" class="">
      <h1 class="font-bold text-2xl">WebRTC Demo</h1>

      <template v-if="!offerWasCopied && !answerWasCopied">
        <p>Either copy a offer:</p>

        <button @click="copyOffer" class="bg-amber-200 p-1 rounded">
          Copy offer: {{ offerIsFinished ? "Offer ready" : "Making offer..." }}
        </button>
        <div class="border-t-2 rounded border-slate-400 my-4"></div>
        <p>
          Or respond to a offer you've recieved by pasting it in here and
          copying the answer:
        </p>

        <textarea
          v-model="remoteOfferInput"
          class="border-slate-300 border-4 rounded"
        ></textarea>
        <button @click="respondToRemoteOffer" class="bg-amber-200 p-1 rounded">
          Copy answer
        </button>
      </template>
      <template v-else-if="answerWasCopied">
        Now send the answer to your peer and paste it in. Once you're done, a
        connection should be established.
      </template>
      <template v-else>
        <p>
          Now send the offer to your peer and paste the answer in here once
          you've got it:
        </p>
        <textarea
          v-model="remoteAnswerInput"
          class="border-slate-300 border-4 rounded"
        ></textarea>
        <button
          @click="connectWithRemoteAnswer"
          class="bg-amber-200 p-1 rounded"
        >
          Connect to answer
        </button>
      </template>
    </main>
    <main v-else>
      <h1 class="font-bold text-2xl">WebRTC: Connected 📱</h1>
      <button class="p-2 bg-amber-200 rounded" @click="open()">
        Select file to send
      </button>
      <button
        v-if="selectedFile"
        class="items-center flex gap-2 p-2 bg-amber-200 rounded break-all"
        @click="sendSelectedFile"
      >
        <span class="grow"> Send {{ selectedFile.name }} </span>
        <i class="icon-[heroicons--paper-airplane] size-10"></i>
      </button>
      <ProgressBar
        v-if="sendingPercentage"
        :value="sendingPercentage * 100"
      ></ProgressBar>
    </main>
  </div>
</template>
