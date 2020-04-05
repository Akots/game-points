<template>
   <div class="game container">
      <div class="row">
         <div class="col">
             <user-setup
                v-if="settingsLoaded"
                v-bind:initialGameModes="gameModes"
                v-bind:initReplay="showResult"
                v-on:start-game="startGame"
                ></user-setup>

            <hr>

            <div class="game-board-cont">
               <div v-if="showResult" class="status-message alert alert-primary" role="alert">{{resultsMessage}}</div>

               <div class="game-board"
                   v-bind:class="{'game-board-locked': gameBoardLocked}"
                   v-bind:style="boardSize">
                  <div class="game-board-cell"
                      v-for="item in gameBoardItems"
                      v-on:click.prevent="gameRowClick(item.id, $event)"
                      v-bind:key="item.id"
                      v-bind:class="{'game-board-cell--highlight' : item.highlight, 'game-board-cell--win' : item.state == 'win', 'game-board-cell--miss' : item.state == 'miss'}"
                      v-bind:style="cellSize"
                      >
                     </div>
                </div>
             </div>
             
         </div>
          
         <div class="col">
            <leader-board
               v-if="leadersLoaded"
               v-bind:init-leaders-data="leadersData"
            ></leader-board>
         </div>
      </div>
   </div>
</template>

<script>
import axios from 'axios';

import UserSetup  from './components/UserSetup.vue';
import LeaderBoard from './components/Leaderboard.vue';

export default {
   name: 'app',

   components: {
      UserSetup,
      LeaderBoard
   },

   data() {
      return {
         gameModes: [],
         leadersData: [],

         settingsLoaded: false,
         leadersLoaded: false,

         gameInProcess: false,
         gameBoardLocked: true,
         showResult: false,

         resultsMessage: '',

         userName: '',

         gameBoardItems: [],
         gameBoardItemsNumb: 25,
         fieldSize: 5,
         highlightPointDelay: 1500,
         dessapearPointDelay: 3000
      }
   },

   computed: {
      boardSize(){
         return {
            'width': 15 + this.fieldSize + 'rem',
            'height': 15 + this.fieldSize + 'rem'
         }
      },
      cellSize(){
         return {
            'flex-basis': (100/this.fieldSize) + '%',
            'height': (100/this.fieldSize) + '%' 
         }
      }
   },

   beforeMount() {
      this.setGameModes();
      this.setLeadersData();
   },

   methods: {
      setGameModes() {
         axios
            .get('https://starnavi-frontend-test-task.herokuapp.com/game-settings')
            .then(response => {
               var data = response.data;
               var gameModes = Object.entries(data);

               for (let [mode, value] of gameModes) {
                  this.gameModes.push({'name': mode, 'mode': value});
               }

               this.settingsLoaded = true;
               this.initGame(this.gameModes[0]);
            });
      },

      setLeadersData() {
         axios
            .get('https://starnavi-frontend-test-task.herokuapp.com/winners')
            .then(response => {
               this.leadersData = response.data;
               this.leadersLoaded = true;
            })
      },

      initGame(params) {
         this.highlightPointDelay = params.mode.delay;
         this.dessapearPointDelay = params.mode.delay * 2;
         this.gameBoardItemsNumb = params.mode.field * params.mode.field;
         this.fieldSize = params.mode.field;
         this.userName = params.userName || '';

         this.generatePointsData();
      },

      startGame(params) {
         this.showResult = false;
         this.gameBoardLocked = false;
         this.gameInProcess = true;

         this.initGame(params);
         this.renderPointsShow();
      },

      getHighlightedPoint(){
         let point = null;

         for (let i = 0; i <= this.gameBoardItemsNumb - 1; i++) {
            if (this.gameBoardItems[i].highlight == true) {
               point = this.gameBoardItems[i];

               break;
            }
         }

         return point;
      },

      getOpenPoints(){
         let pointsId = [];

         for (let i = 0; i <= this.gameBoardItemsNumb - 1; i++) {
            if (this.gameBoardItems[i].state == null) {
               pointsId.push(this.gameBoardItems[i].id);
            }
         }

         return pointsId;
      },

      calculatePointPlace() {
         let openPointsId = this.getOpenPoints();
         let newPointId;
         let minNumber = 1;

         newPointId = Math.floor(Math.random() * (this.gameBoardItemsNumb)) + minNumber;

         return (openPointsId.length ? 
                     (openPointsId.includes(newPointId) ? newPointId : this.calculatePointPlace()) 
                : false);
      },

      gameRowClick(itemId) {
         let pointClicked = this.gameBoardItems[itemId - 1];
         let pointHighlighted = this.getHighlightedPoint();

         if (this.gameBoardLocked || !pointHighlighted) return false;

         if (pointClicked.highlight == true) {
            pointClicked.state = "win";
         }
         else {
            pointHighlighted.state = "miss";
         }

         this.interruptPointShow();
      },

      interruptPointShow() {
         clearTimeout(this.finishShowPointInterval);
         clearTimeout(this.showPointInterval);
         this.renderPointResult();
      },

      renderPointsShow() {
         this.gameBoardLocked = false;

         this.showPointInterval = setTimeout(() => {
            let newPointId = this.calculatePointPlace();

            if (newPointId) {
               let boardItem = this.gameBoardItems[newPointId - 1];

               boardItem.highlight = true;

               this.finishShowPointInterval = setTimeout(() => {
                  this.gameBoardLocked = true;
                  this.renderPointResult();

               }, this.dessapearPointDelay);
            }
            else {
               this.renderResultMessage();
            }

         }, this.highlightPointDelay);
      },

      renderPointResult() {
         let boardItem = this.getHighlightedPoint();

         boardItem.highlight = false;

         if (boardItem.state != "win") {
            boardItem.state = "miss";
         }

         const gameResults = this.getGameResults();

         if (gameResults.finished) {
            this.renderResultMessage();
            this.saveResults(gameResults.winner);
         }
         else {
            this.renderPointsShow();
         }
      },

      getGameResults() {
         let missPointsId = [];
         let winPointsId = [];
         let winScore = Math.floor(this.gameBoardItemsNumb/2);

         for (let i = 0; i <= this.gameBoardItemsNumb - 1; i++) {
            if (this.gameBoardItems[i].state != null) {

               if (this.gameBoardItems[i].state == "win") {
                  winPointsId.push(this.gameBoardItems[i].id);
               }
               else {
                  missPointsId.push(this.gameBoardItems[i].id);
               }
            }
         }

         return {
                  finished : (missPointsId.length >= winScore || winPointsId.length >= winScore),
                  winner : winPointsId.length >= winScore ?  "user" : "computer"
               };
      },

      generatePointsData() {
         this.gameBoardItems = [];

         for (let i = 0; i <= this.gameBoardItemsNumb - 1; i++) {
            this.gameBoardItems.push({"id" : (i + 1), "highlight" : false, "state" : null});
         }
      },

      renderResultMessage() {
         const gameResults = this.getGameResults();

         this.showResult = true;
         this.gameInProcess = false;

         if (gameResults.winner == "computer") {
            this.resultsMessage = 'Computer win';
         }
         else {
            this.resultsMessage = this.userName + ' win';
         }
      },

      getFormattedDate() {
         const months    = ['January','February','March','April','May','June','July','August','September','October','November','December'];
         const now       = new Date();
         var hours = now.getHours();
         var minutes = now.getMinutes();
         const thisMonth = months[now.getMonth()];

         hours = (hours<10) ? '0' + hours : hours;
         minutes = (minutes<10) ? '0' + minutes : minutes;

         return hours + ':' + minutes + '; ' + now.getDate() + ' ' + thisMonth + ' ' + now.getFullYear();
      },

      saveResults(winner) {
         let date = this.getFormattedDate();
         let winnerName = (winner == "user" ? this.userName : "Computer");

         axios
            .post('https://starnavi-frontend-test-task.herokuapp.com/winners', {
               'winner': winnerName,
               'date': date
            }).then(response => {
               if (response.data && response.data.length) {
                  let newEnties = response.data.slice(this.leadersData.length);

                  newEnties.forEach(entry => (this.leadersData.push(entry)));
               }
            });
      }
   }
}
</script>

<style lang="scss">
  @import '~bootstrap/scss/bootstrap.scss';
  @import './assets/css/base.scss';
</style>