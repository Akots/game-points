<template>
   <form class="gamer-setup form-inline">
      <div class="form-group mx-sm-2 mb-2">
         <select autofocus v-model="userSettings" class="form-control">
            <option disabled value="null">Pick game mode</option>
            <option
               v-for="(item, index) in gameModes"
               v-bind:value="item"
               v-bind:key="index">
               {{item.name.replace("Mode", "")}}
            </option>
         </select>
      </div>
      <div class="form-group mx-sm-3 mb-2">
         <input placeholder="Enter your name" class="form-control" type="text" 
            v-model="userName">
      </div>
      
      <button v-on:click.prevent="startGame" class="btn btn-primary mb-2" v-bind:disabled="disabled">{{firstPlay ? 'Play!' : 'Replay'}}</button>
   </form>
</template>

<script>
   export default {
      props: ['initialGameModes', 'initReplay'],

      data() {
         return {
            gameModes: this.initialGameModes,

            firstPlay: true,
            disabled: true,

            userName: '',
            userSettings: null
         };
      },

      watch: {
         userSettings: function(val){
            if (val && this.userName != '') {
               this.disabled = false;
            }
         },

         userName: function(val){
            if (val && this.userSettings != null) {
               this.disabled = false;
            }
         },

         initReplay: function(){
            if (this.initReplay){
               this.firstPlay = false;
               this.disabled = false;
            }
         }
      },

      mounted(){
         this.setDefaultMode();
      },
         
      methods: {
         setDefaultMode() {
            for (var i = this.gameModes.length - 1; i >= 0; i--) {
               if (this.gameModes[i].selected) {
                  this.userSettings = this.gameModes[i];

                  break;
               }
            }
         },

         startGame(){
            let gameProps = this.userSettings;
            gameProps.userName = this.userName;

            this.$emit('start-game', gameProps);
            this.disabled = true;
         }
      }
   }
</script>