<template>
  <div class="gardening">
    <h2>DebugMenu</h2>
    <h3>data control</h3>
    <button @click="resetCountPlayer">Reset Player Data</button>
  </div>
</template>

<script>
import { initializeApp } from "firebase/app";
import { getFirestore, doc, writeBatch, getDoc } from 'firebase/firestore';
import firebaseConfig from "../script/firebase.js";

export default {
  name: 'debugPage',
  data: function() {
    return {
      playerSettingButtons: [
        { cmd: 'setPlayer1', title: "player-1", color: "赤" },
        { cmd: 'setPlayer2', title: "player-2", color: "青" },
        { cmd: 'setPlayer3', title: "player-3", color: "白" },
        { cmd: 'setPlayer4', title: "player-4", color: "ピンク" },
        { cmd: 'setPlayer5', title: "player-5", color: "オレンジ" },
        { cmd: 'setPlayer6', title: "player-6", color: "紫" },
        { cmd: 'setPlayer7', title: "player-7", color: "黄" },
        { cmd: 'setPlayer8', title: "player-8", color: "アヤメ" },
      ],
      firestoreDB: null,
    }
  },

  mounted: function () {
    const app = initializeApp(firebaseConfig);
    this.firestoreDB = getFirestore(app);
    console.log("firebase: "+this.firestoreDB);
    
    console.log(this.playerSettingButtons);
  },

  methods: {
    getCountPlayer: async function () {
      const docRef
    },

    resetCountPlayer: async function () {
      const batch = writeBatch(this.firestoreDB);

      this.playerSettingButtons.forEach(async (playerSettingButton) => {
        const playersRef = doc(this.firestoreDB, "players", playerSettingButton.title);
        batch.update(playersRef, {
          motivation: 0,
          nod: 0,
          speech: 0
        });
        console.log("Reset player: ", playerSettingButton.title);
      });

      await batch.commit();
    },
  }
}
</script>

<style scoped>
button {
  font-size: 1.3rem;
  font-weight: bold;
  position: relative;
  display: inline-block;
  padding: 1rem 1.3rem;
  cursor: pointer;
  -webkit-transition: all 0.3s;
  transition: all 0.3s;
  text-align: center;
  letter-spacing: 0.1em;
  border-radius: 0.5rem;
  color: rgb(58, 58, 58);
  background: #d6d6d6;
  border: 1px solid #ccc;
  margin: 10px;
}

button:hover {
  color: rgb(58, 58, 58);
  background: #8d8d8d;
}
</style>