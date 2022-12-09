<template>
  <div class="garden-block">
    <h2>■ 花のメタファによるフィードバック</h2>
      <div class="flower-block" :style="{position: `relative`}">
        <img id="bed" :src="require(`@/assets/flower-bed.png`)" width="535">
        <!-- 花の表示 -->
        <div class="flower" v-for="(item, index) in displayFlowerPostionArray" :key="`red-${index}`">
          <img
            id="flower-red"
            width="50"
            v-if="item.status"
            :src="item.image"
            :style="{position: 'absolute', top: gardenPadding + item.y + 'px', left: gardenPadding + item.x + 'px' }"
          >
        </div>
          <div class="set-support-button" v-if="isStartedFeedback">
            <h3>補助ボタン</h3>
            <button v-on:click="listeningButton">聞いています</button>
            <button v-on:click="motivationButton">発言したい</button>
          </div>
      </div>
      <div class="setting-block">
        <div class="player-status">
          <h3>1. プレイヤー設定：{{ playerStatusText }}</h3>
          <button
            v-for="btn in playerSettingButtons" 
            :key="btn.id"
            @click="setPlayer(btn)">{{btn.title}}:{{ btn.color }}
          </button>
        </div>
        <div class="feedback-control">
          <h3>2. DB接続: {{ dbConnectionStatusText }}</h3>
          <button class="test-start-button" v-on:click="startFeedback">Start</button>
          <button class="test-stop-button" v-on:click="stopFeedback">Stop</button>
        </div>
        <div class="tracking-control">
          <h3>3. 会話行動検知: {{ trackingStatusText }}</h3>
          <div>※ 上手く検出されない場合は，Stop</div>
          <button class="test-start-button" v-on:click="startAllTracking">Start</button>
          <button class="test-stop-button" v-on:click="stopAllTracking">Stop</button>
        </div>
      </div>
      <video ref="video" id="video" width="500" height="500" autoplay></video>
  </div>
</template>

<script>
import * as faceLandmarksDetection from '@tensorflow-models/face-landmarks-detection';
import vad from 'voice-activity-detection';
import * as tf from '@tensorflow/tfjs';
import positionArray from "../script/position.js";
import firebaseConfig from "../script/firebase.js";
import { initializeApp } from "firebase/app";
import { getFirestore, onSnapshot, collection, query, doc, updateDoc } from 'firebase/firestore';

export default {
  name: 'feedbackPage',
  data: function() {
    return {
      displayFlowerPostionArray: [],
      gardenPadding: 30,
      playerStatusText: "player-1", //"no setting"
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
      dbConnectionStatusText: "未選択",
      trackingStatusText: "未選択",
      firestoreDB: null,
      unsubscribeDB: null,
      countStatus: {
        motivation: 0,
        nod: 0,
        speech: 0
      },
      flowerImageType: {
        redFlower: require(`@/assets/flower-red.png`),
        blueFlower: require(`@/assets/flower-blue.png`),
        whiteFlower: require(`@/assets/flower-white.png`),
        pinkFlower: require(`@/assets/flower-pink.png`),
        greenFlower: require(`@/assets/flower-ayame.png`),
        orangeFlower: require(`@/assets/flower-orange.png`),
        yellowFlower: require(`@/assets/flower-yellow.png`),
        purpleFlower: require(`@/assets/flower-purple.png`),
        witheredFlower: require(`@/assets/flower-withered.png`),
        redBud: require(`@/assets/bud-red.png`),
        blueBud: require(`@/assets/bud-blue.png`),
        whiteBud: require(`@/assets/bud-white.png`),
        pinkBud: require(`@/assets/bud-pink.png`),
        greenBud: require(`@/assets/bud-ayame.png`),
        orangeBud: require(`@/assets/bud-orange.png`),
        yellowBud: require(`@/assets/bud-yellow.png`),
        purpleBud: require(`@/assets/bud-purple.png`),
        none: null,
      },
      beforeRedSpeechMotivation: 0,
      beforeBlueSpeechMotivation: 0,
      beforeWhiteSpeechMotivation: 0,
      beforePinkSpeechMotivation: 0,
      beforeAyameSpeechMotivation: 0,
      beforeOrangeSpeechMotivation: 0,
      beforeYellowSpeechMotivation: 0,
      beforePurpleSpeechMotivation: 0,
      isDebug: false,
      nodCount: 0,
      speechCount: 0,
      motivationCount: 0,
      meanNodArray: [],
      flowerPosition: positionArray,
      // speech detection
      vadObject: null,
      context: null,
      isSpeechCount: false,
      isSpeech: false,
      // nod detection
      faceModel: null,
      video: null,
      detector: null,
      nodAverageScore: 0,
      // Flower Array
      flowerImage: null,
      faceIntervalTimer: null,
      isStartedFeedback: false,
    }
  },
  mounted: async function () {
    window.AudioContext = window.AudioContext || window.webkitAudioContext;
    this.video = document.querySelector("video");
    let constraints = {
      video: true,
      audio: false
    };
    try {
      const stream = await navigator.mediaDevices.getUserMedia(constraints);
      this.video.srcObject = stream;
      this.video.play()
    } catch (error) {
      console.log(error)
    }

    const app = initializeApp(firebaseConfig);
    this.firestoreDB = getFirestore(app);
    console.log("firebase: "+this.firestoreDB);
  },

  watch: {
    nodCount: function() {
      if (this.isDebug) return; 
      console.log("post");
      this.postCountPlayer();
    },
    
    speechCount: function() {
      if (this.isDebug) return; 
      this.postCountPlayer();
    },

    motivationCount: function() {
      if (this.isDebug) return; 
      this.postCountPlayer();
    },
  },

  computed: {
    calcSum: function() {
      return this.meanNodArray.reduce(function(sum, element) {
        return sum + element;
      })
    },
    calcAverage: function() {
      return Math.floor(this.calcSum / this.meanNodArray.length);
    }
  },

  methods: {
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
    },

    startFeedback: async function() {
      if (this.playerStatusText === "no setting") {
        alert("playerを設定してください。");
        return;
      }
      this.dbConnectionStatusText = "開始";
      this.isStartedFeedback = true;
      
      const q = query(collection(this.firestoreDB, "players"));
      this.unsubscribeDB = onSnapshot(q, (snapshot) => {
        snapshot.docChanges().forEach((change) => {
          if (change.type === "added") {
            
            console.log("Modified: ", change.doc.data());
            // DBのカウント数に合わせて花の表示を行う（ランダム表示）
            this.setFlowerCountWithDB(change.doc.id, change.doc.data());
            if (change.doc.id == this.playerStatusText) {
              this.countStatus = change.doc.data(); 
            }
          }

          if (change.type === "modified") {
            console.log("Modified: ", change.doc.data());
            console.log("Player: ", change.doc.id);

            switch(change.doc.id) {
              case 'player-1':
                if (this.beforeRedSpeechMotivation < change.doc.data().motivation) {
                  // speechtとnod または motivationか判断するために変更前のカウントを格納しておく
                  this.beforeRedSpeechMotivation = change.doc.data().motivation;
                  this.setBudTypeWithDB(change.doc.id);
                } else {
                  this.setFlowerTypeWithDB(change.doc.id);
                }
                break
              case 'player-2':
                if (this.beforeBlueSpeechMotivation < change.doc.data().motivation) {
                  this.beforeBlueSpeechMotivation = change.doc.data().motivation;
                  this.setBudTypeWithDB(change.doc.id);
                } else {
                  this.setFlowerTypeWithDB(change.doc.id);
                }
                break
              case 'player-3':
                if (this.beforeWhiteSpeechMotivation < change.doc.data().motivation) {
                  this.beforeWhiteSpeechMotivation = change.doc.data().motivation;
                  this.setBudTypeWithDB(change.doc.id);
                } else {
                  this.setFlowerTypeWithDB(change.doc.id);
                }
                break
              case 'player-4':
                if (this.beforePinkSpeechMotivation < change.doc.data().motivation) {
                  this.beforePinkSpeechMotivation = change.doc.data().motivation;
                  this.setBudTypeWithDB(change.doc.id);
                } else {
                  this.setFlowerTypeWithDB(change.doc.id);
                }
                break
              case 'player-5':
                if (this.beforeOrangeSpeechMotivation < change.doc.data().motivation) {
                  this.beforeOrangeSpeechMotivation = change.doc.data().motivation;
                  this.setBudTypeWithDB(change.doc.id);
                } else {
                  this.setFlowerTypeWithDB(change.doc.id);
                }
                break
              case 'player-6':
                if (this.beforePurpleSpeechMotivation < change.doc.data().motivation) {
                  this.beforePurpleSpeechMotivation = change.doc.data().motivation;
                  this.setBudTypeWithDB(change.doc.id);
                } else {
                  this.setFlowerTypeWithDB(change.doc.id);
                }
                break
              case 'player-7':
                if (this.beforeYellowSpeechMotivation < change.doc.data().motivation) {
                  this.beforeYellowSpeechMotivation = change.doc.data().motivation;
                  this.setBudTypeWithDB(change.doc.id);
                } else {
                  this.setFlowerTypeWithDB(change.doc.id);
                }
                break
              case 'player-8':
                if (this.beforeAyameSpeechMotivation < change.doc.data().motivation) {
                  this.beforeAyameSpeechMotivation = change.doc.data().motivation;
                  this.setBudTypeWithDB(change.doc.id);
                } else {
                  this.setFlowerTypeWithDB(change.doc.id);
                }
                break
            }
          }

          if (change.type === "removed") {
            console.log("Removed: ", change.doc.data());
          }
        });
      });
    },

    stopFeedback: function() {
      console.log("stopFeedback");
      this.dbConnectionStatusText = "終了";
      this.isStartedFeedback = false;
      if (this.unsubscribeDB == undefined) return;
      this.unsubscribeDB();
    },

    startAllTracking: function() {
      this.trackingStatusText = "開始";
      this.startTracking();
      this.startDetectionSpeech();
    },

    stopAllTracking: function() {
      this.trackingStatusText = "終了";
      this.stopTracking();
      this.stopVAD();
    },

    setFlowerCountWithDB: function(playerID, docData) {
      const flowerCount = docData.speech + docData.nod;
      for (let i=0; i<flowerCount; i++) {
        this.setFlowerTypeWithDB(playerID);
      }

      for (let i=0; i<docData.motivation; i++) {
        this.setBudTypeWithDB(playerID);
      }
    },

    setFlowerTypeWithDB: function(playerID) {
      switch(playerID) {
        case 'player-1':
          this.displayFlowerImage(this.flowerImageType.redFlower);
          break
        case 'player-2':
          this.displayFlowerImage(this.flowerImageType.blueFlower);
          break
        case 'player-3':
          this.displayFlowerImage(this.flowerImageType.whiteFlower);
          break
        case 'player-4':
          this.displayFlowerImage(this.flowerImageType.pinkFlower);
          break
        case 'player-5':
          this.displayFlowerImage(this.flowerImageType.orangeFlower);
          break
        case 'player-6':
          this.displayFlowerImage(this.flowerImageType.purpleFlower);
          break
        case 'player-7':
          this.displayFlowerImage(this.flowerImageType.yellowFlower);
          break
        case 'player-8':
          this.displayFlowerImage(this.flowerImageType.greenFlower);
          break
      }
    },

    setBudTypeWithDB: function(playerID) {
      switch(playerID) {
        case 'player-1':
          this.displayFlowerImage(this.flowerImageType.redBud);
          break
        case 'player-2':
          this.displayFlowerImage(this.flowerImageType.blueBud);
          break
        case 'player-3':
          this.displayFlowerImage(this.flowerImageType.whiteBud);
          break
        case 'player-4':
          this.displayFlowerImage(this.flowerImageType.pinkBud);
          break
        case 'player-5':
          this.displayFlowerImage(this.flowerImageType.orangeBud);
          break
        case 'player-6':
          this.displayFlowerImage(this.flowerImageType.purpleBud);
          break
        case 'player-7':
          this.displayFlowerImage(this.flowerImageType.yellowBud);
          break
        case 'player-8':
          this.displayFlowerImage(this.flowerImageType.greenBud);
          break
      }
    },

    startTracking: async function() {
      const model = faceLandmarksDetection.SupportedModels.MediaPipeFaceMesh;
      const detectorConfig = {
        runtime: 'mediapipe',
        solutionPath: 'https://cdn.jsdelivr.net/npm/@mediapipe/face_mesh',
      }
      this.detector = await faceLandmarksDetection.createDetector(model, detectorConfig);

      this.faceIntervalTimer = setInterval(async() => {
        const faces = await this.detector.estimateFaces(this.video);
        // MARK: - Nod Recognition

        // 顔の角度計算
        const rad = this.calcFaceRotation(faces);
        console.log("rad_x: ",rad[0],"rad_y: ",rad[1]);
        this.meanNodArray.push(rad[1]);

        console.log("meanNodArray: ", this.meanNodArray);

        // keypoint
        this.outputDebug("nod point", rad[1]);

        // meanNodArrayのサンプル数が10以下の場合、skipする
        if (this.meanNodArray.length < 10) {
          this.nodCount = 0;
          this.nodAverageScore = this.calcAverage;
          console.log("nod count: skip");
        }

        if (this.meanNodArray.length >= 10) {
          // 10秒ごとのnodの平均算出
          this.outputDebug("nod average", this.nodAverageScore);
          // 平均の基準よりも下を向いているか
          if (this.nodAverageScore-5 > rad[1]) {
            console.log("****** nod count ******");
            this.nodCount += 1;
            this.countStatus.nod += 1;
            console.log(this.nodCount);
            this.outputDebug("this.countStatus.nod", this.countStatus.nod);
          }
          this.meanNodArray.shift();
        }

        // MARK: - Mouth Recognition
        const topLipPoint = faces[0].keypoints[13].y
        const underLipPoint = faces[0].keypoints[14].y
        const mouseOpened = underLipPoint - topLipPoint
        if (!this.isSpeech && (10 < mouseOpened && mouseOpened < 20)) {
          console.log("top: "+topLipPoint+"under: "+underLipPoint+"\n"+"mouseOpened: "+mouseOpened);
          console.log("****** motivation count ******");
          this.motivationCount += 1;
          this.countStatus.motivation += 1;
          this.outputDebug("this.countStatus.motivation", this.countStatus.motivation);
          return
        }
        console.log("motivation count: skip");
      }, 1000) // 1秒間
    },

    stopTracking: function() {
      this.detector = null;
      clearInterval(this.faceIntervalTimer);
    },

    displayFlowerImage: function (displayImageType) {
      if(!this.flowerPosition.length) { return }

      // ランダムにindexを算出して，フラワー表示配列から表示するIDを取得
      const randomIndex = Math.floor(Math.random() * this.flowerPosition.length);
      const randomFlowerPosition = this.flowerPosition[randomIndex];

      // flowerPositionIDに紐づくflowerPositionのステータスを変更
      const index = this.flowerPosition.indexOf(randomFlowerPosition);
      this.flowerPosition[index].status = true;
      this.flowerPosition[index].image = displayImageType
      this.setRemoveFlowerTimer(this.flowerPosition[index]);

      // 表示されるflowerPosition表示済配列に格納
      this.displayFlowerPostionArray.push(this.flowerPosition[index]);
      // 表示されるflowerPositionの要素を配列から削除
      this.flowerPosition.splice(index, 1);
    },

    setRemoveFlowerTimer: function (flowerPosition) {
      // 30秒後に枯れ花表示
      setTimeout(() => {
        const index = this.displayFlowerPostionArray.indexOf(flowerPosition);
        this.displayFlowerPostionArray[index].image = this.flowerImageType.witheredFlower;
      }, 30000);

      // 2秒後に枯れ花表示
      setTimeout(() => {
        console.log("remove flower");
        // 表示済配列から花のオブジェクトと一致するindexを取得
        const index = this.displayFlowerPostionArray.indexOf(flowerPosition);
        this.displayFlowerPostionArray[index].status = false;
        this.displayFlowerPostionArray[index].image = this.flowerImageType.none;

        // 非表示にされた花のオブジェクトをフラワー表示配列に格納
        this.flowerPosition.push(this.displayFlowerPostionArray[index]);
        // 表示済配列から非表示にされた花のオブジェクトの要素を削除
        this.displayFlowerPostionArray.splice(index, 1);
      }, 32000);
    },

    startDebug: function() {
      this.isDebug = !this.isDebug;
    },

    postCountPlayer: async function() {
      const playersRef = doc(this.firestoreDB, "players", this.playerStatusText)
      await updateDoc(playersRef, {
        motivation: this.countStatus.motivation,
        nod: this.countStatus.nod,
        speech: this.countStatus.speech
      });
    },

    startDetectionSpeech: function() {
      console.log("start detection");
      this.setVAD((val) => {
        console.log(val);
      }, ()=>{
        if (!this.isSpeechCount) {
          this.isSpeechCount = true;
          return;
        }
        console.log("****** speechカウント ******");
        this.speechCount += 1;
        this.countStatus.speech += 1;
        this.outputDebug("this.countStatus.speech", this.countStatus.speech);
      })
    },

    setVAD: function(update, stopHandler) {
      let vadOptions = {
        minCaptureFreq: 100,
        maxCaptureFreq: 300,
        minNoiseLevel: 0.4,
        maxNoiseLevel: 0.4,
        onVoiceStart: function() {
          console.log("Voice Start");
          this.isSpeech = true;
        },
        onVoiceStop: function() {
            console.log("Voice Stop");
            this.isSpeech = false;
            stopHandler();
        },
        onUpdate: function(val) {
            // 音声が検出されると発火
            update(val);
        }
      }
      this.context = new AudioContext()

      if (navigator.mediaDevices && navigator.mediaDevices.getDisplayMedia) {
        navigator.mediaDevices.getUserMedia({audio: true}).then((stream) => {
          this.vadObject = vad(this.context, stream, vadOptions);
        })
      }
    },

    stopVAD: function() {
      console.log("VadObject Destroy");
      if (this.vadObject == null) return;
      this.vadObject.destroy();
      this.vadObject = null;
    },

    outputDebug: function(message, value) {
      if(message === null) { message = "" }
      if(value === null) { value = "" }

      console.log("【Debug】: "+message+" - "+value);
    },

    calcFaceRotation: function(faces) {
      let faceOvalArray = [];

      faces[0].keypoints.forEach((keypoint) => {
        if (keypoint.name == undefined) return
        if (keypoint.name == 'faceOval') {
          faceOvalArray.push([keypoint.x, keypoint.y, keypoint.z]);
        }
      });

      // tfの配列格納
      let faceMulciArray = tf.tensor(faceOvalArray);

      // 目の算出
      const rightEye = tf.tensor1d([
        faces[0].keypoints[33].x, faces[0].keypoints[33].y, faces[0].keypoints[33].z
      ]);
      const leftEye = tf.tensor1d([
        faces[0].keypoints[263].x, faces[0].keypoints[263].y, faces[0].keypoints[263].z
      ]);

      // 顔点群の正規化
      const faceNorm = faceMulciArray.div(tf.norm(rightEye.sub(leftEye)).mul(0.06));
      const centerPosition = faceNorm.sub(faceNorm.mean(0));

      const c00 = centerPosition.slice(0, 1).as1D();
      const c09 = centerPosition.slice(9, 1).as1D();
      const c18 = centerPosition.slice(18, 1).as1D();
      const c27 = centerPosition.slice(27, 1).as1D();

      // 回転行列
      const rotate0 = c18.sub(c00).div(tf.norm(c18.sub(c00))); // 顔の垂直方向
      const rotate1 = c09.sub(c27).div(tf.norm(c09.sub(c27))); // 顔の水平方向

      // tfに変換
      let rotate = tf.concat([rotate0, rotate1]).arraySync();

      // y座標のベクトル
      const m00 = rotate[0];
      const m01 = rotate[1];
      const m02 = rotate[2];

      // x座標のベクトル
      const m10 = rotate[3];
      const m11 = rotate[4];
      const m12 = rotate[5];

      // 外積より法線ベクトルを求める
      const m20 = m01 * m12 - m02 * m11;
      const m21 = m02 * m10 - m00 * m12;
      const m22 = m00 * m11 - m01 * m10;

      const sy = Math.sqrt(m00 * m00 + m10 * m10);
      const X = (Math.atan2(m21, m22) * 180) / Math.PI;
      const Y = (Math.atan2(-m20, sy) * 180) / Math.PI;
      const Z = (Math.atan2(m10, m00) * 180) / Math.PI;

      let radX = 0;
      let radY = 0;

      //Y軸のマイナス値の変更
      if (X < 0) {
        radY = Math.floor(X) + 360;
      } else {
        radY = X;
      }

      if (radY < 100) {
        radY += 300;
      }

      radX = Math.floor(Y) + 45;

      return [radX, radY, Z];
    },

    listeningButton: function () {
      console.log("****** nod count ******");
      console.log("count status",this.countStatus.nod);
      this.nodCount += 1;
      console.log("count status",this.countStatus.nod);
      this.countStatus.nod += 1;
      console.log("count status",this.countStatus.nod);
    },

    motivationButton: function () {
      console.log("****** motivation count ******");
      this.motivationCount += 1;
      this.countStatus.motivation += 1;
    }
  },
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

.flower-block {
  padding-bottom: 400px;
}

</style>