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

   watch: {
      leadersData: function(val){
         console.log(val);
      }
   },

   methods: {
      setGameModes() {
         const _this = this;

         axios
            .get('https://starnavi-frontend-test-task.herokuapp.com/game-settings')
            .then(response => {
               var data = response.data;
               var gameModes = Object.entries(data);

               for (let [mode, value] of gameModes) {
                  _this.gameModes.push({'name': mode, 'mode': value});
               }

               _this.settingsLoaded = true;
               _this.initGame(this.gameModes[0]);
            });
      },

      setLeadersData() {
         const _this = this;

         axios
            .get('https://starnavi-frontend-test-task.herokuapp.com/winners')
            .then(response => {
               _this.leadersData = response.data;
               _this.leadersLoaded = true;
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
         var point = null;

         for (let i = 0; i <= this.gameBoardItemsNumb - 1; i++) {
            if (this.gameBoardItems[i].highlight == true) {
               point = this.gameBoardItems[i];

               break;
            }
         }

         return point;
      },

      getOpenPoints(){
         var pointsId = [];

         for (let i = 0; i <= this.gameBoardItemsNumb - 1; i++) {
            if (this.gameBoardItems[i].state == null) {
               pointsId.push(this.gameBoardItems[i].id);
            }
         }

         return pointsId;
      },

      calculatePointPlace() {
         var openPointsId = this.getOpenPoints();
         var newPointId;
         var minNumber = 1;

         newPointId = Math.floor(Math.random() * (this.gameBoardItemsNumb)) + minNumber;

         return (
            openPointsId.length ? 
               (openPointsId.includes(newPointId) ? newPointId : this.calculatePointPlace()) : 
            false);
      },

      gameRowClick(itemId) {
         var pointClicked = this.gameBoardItems[itemId - 1];
         var pointHighlighted = this.getHighlightedPoint();

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
         let _this = this;

         this.gameBoardLocked = false;

         this.showPointInterval = setTimeout(function(){
            let newPointId = _this.calculatePointPlace();

            if (newPointId) {
               let boardItem = _this.gameBoardItems[newPointId - 1];

               boardItem.highlight = true;

               _this.finishShowPointInterval = setTimeout(function(){
                  _this.gameBoardLocked = true;
                  _this.renderPointResult();

               }, _this.dessapearPointDelay);
            }
            else {
               _this.renderResultMessage();
            }
            

         }, this.highlightPointDelay)
      },

      renderPointResult() {
         var boardItem = this.getHighlightedPoint();

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
         var missPointsId = [];
         var winPointsId = [];
         var winScore = Math.floor(this.gameBoardItemsNumb/2);

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
         var date = this.getFormattedDate();
         var winnerName = (winner == "user" ? this.userName : "Computer");
         var _this = this;

         axios
            .post('https://starnavi-frontend-test-task.herokuapp.com/winners', {
               'winner': winnerName,
               'date': date
            }).then(response => {
               const data = response.data;
               var newEnties = data.slice(_this.leadersData.length);

               newEnties.forEach(entry => (_this.leadersData.push(entry)));
            });
      }
   }
}
</script>

<style lang="scss">
  @import '~bootstrap/scss/bootstrap.scss';
  @import './assets/css/base.scss';
</style>