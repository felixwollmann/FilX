<script setup lang="ts">
import { ref, useTemplateRef } from "vue";

const configuration = {
  // iceServers: [{ urls: "stun:stun.l.google.com:5349" }],
  iceServers: [{ urls: "stun:freestun.net:3478" }],
};

const peerConnection = new RTCPeerConnection(configuration);
let dataChannel = peerConnection.createDataChannel("data");
dataChannel.addEventListener("open", (event) => {
  console.log("YAAAAAY", event);
});

dataChannel.addEventListener("message", handleMessage);

console.log("peerConnection:", peerConnection);
const offer = await peerConnection.createOffer();
await peerConnection.setLocalDescription(offer);

// peerConnection.setRemoteDescription

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
    console.log('YAY 2');
    dataChannel = event.channel;
    dataChannel.addEventListener("message", handleMessage);
  });
}

async function connectWithRemoteAnswer() {
  await peerConnection.setRemoteDescription(
    JSON.parse(remoteAnswerInput.value)
  );
}

async function handleMessage(event: MessageEvent) {
  console.log('Recieved message:', event.data);
}

async function sendMessage() {
  const payload = `${Math.random()}`;
  console.log('Sending Message:', payload);
  dataChannel.send(payload);
}
</script>

<template>
  <main>
    <p>Trying out WebRTC.</p>
    <button @click="copyOffer" class="bg-amber-200 p-1 rounded">
      Copy offer: {{ offerIsFinished ? "Offer ready" : "Making offer..." }}
    </button>

    <br>
    <textarea v-model="remoteOfferInput"></textarea>
    <button @click="respondToRemoteOffer">Copy answer</button>
    <br>
    <textarea v-model="remoteAnswerInput"></textarea>
    <button @click="connectWithRemoteAnswer">Connect to answer</button>
    <br>
    <button @click="sendMessage">Send hello message</button>
    
    

    <!-- <code>{{ peerConnection.localDescription?.sdp }}</code> -->
  </main>
</template>
