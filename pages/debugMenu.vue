<template>
  <div class="gardening">
    <h2>DebugMenu</h2>
    <h3>data control</h3>
    <button @click="getCountPlayers">Get All Player Data</button>
    <button @click="resetCountPlayers">Reset All Player Data</button>
    <button @click="collectCountPlayer">Collection All Player</button>
    <h3>プレイヤー設定：{{ playerStatusText }}</h3>
    <button
      v-for="btn in playerSettingButtons" 
      :key="btn.id"
      @click="setPlayer(btn)">{{btn.title}}:{{ btn.color }}
    </button>
    <button @click="resetCountPlayerX">プレイヤー：{{ playerStatusText }}のデータをリセット</button>
  </div>
</template>

<script>
import { initializeApp } from "firebase/app";
import { getFirestore, doc, writeBatch, collection, getDocs, updateDoc, serverTimestamp } from 'firebase/firestore';
import firebaseConfig from "../script/firebase.js";

export default {
  name: 'debugPage',
  data: function() {
    return {
      playerStatusText: "",
      playerSettingButtons: [
        { cmd: 'setPlayer1', title: "player-1", color: "赤" },
        { cmd: 'setPlayer2', title: "player-2", color: "青"},
        { cmd: 'setPlayer3', title: "player-3", color: "白"},
        { cmd: 'setPlayer4', title: "player-4", color: "ピンク"},
        { cmd: 'setPlayer5', title: "player-5", color: "オレンジ"},
        { cmd: 'setPlayer6', title: "player-6", color: "紫"},
        { cmd: 'setPlayer7', title: "player-7", color: "黄"},
        { cmd: 'setPlayer8', title: "player-8", color: "アヤメ"},
      ],
      countStatus: [],
      firestoreDB: null,
    }
  },

  mounted: function () {
    const app = initializeApp(firebaseConfig);
    this.firestoreDB = getFirestore(app);
    console.log("firebase: "+this.firestoreDB);
  },

  methods: {
    getCountPlayers: async function () {
      const querySnapshot = await getDocs(collection(this.firestoreDB, "players"));
      querySnapshot.forEach((doc) => {
        console.log(doc.id, " => ", doc.data());
      });
    },

    collectCountPlayer: async function () {
      this.playerSettingButtons.forEach((playerSettingButton) => {
        this.countStatus.push({ player: playerSettingButton.title, speech: 0, nod: 0, motivation: 0, time: 0 });
      });
      console.log(this.countStatus);
      const querySnapshot = await getDocs(collection(this.firestoreDB, "players"));
      setInterval(() => {
        querySnapshot.forEach((doc) => {
          console.log(doc.id, " => ", doc.data());
          this.countStatus.speech = doc.data().speech
          this.countStatus.nod = doc.data().nod
          this.countStatus.motivation = doc.data().movtivation
        });
      }, 1000);
    },

    resetCountPlayers: async function () {
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

    resetCountPlayerX: async function() {
      if (this.playerStatusText == "") return;

      console.log("resetCountPlayer: ", this.playerStatusText);
      const playerRef = doc(this.firestoreDB, "players", this.playerStatusText);
      await updateDoc(playerRef, {
        motivation: 0,
        nod: 0,
        speech: 0
      });
    },

    setPlayer(command) {
      switch(command.cmd) {
        case 'setPlayer1':
          this.playerStatusText = "player-1";
          break
        case 'setPlayer2':
          this.playerStatusText = "player-2";
          break
        case 'setPlayer3':
          this.playerStatusText = "player-3";
          break
        case 'setPlayer4':
          this.playerStatusText = "player-4";
          break
        case 'setPlayer5':
          this.playerStatusText = "player-5";
          break
        case 'setPlayer6':
          this.playerStatusText = "player-6";
          break
        case 'setPlayer7':
          this.playerStatusText = "player-7";
          break
        case 'setPlayer8':
          this.playerStatusText = "player-8";
          break
        default:
          this.playerStatusText = "player-1";
      }
    }
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