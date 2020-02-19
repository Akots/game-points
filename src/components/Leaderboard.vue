<template>
   <div>
      <h3 class="mb-4">Leader Board</h3>
      <div class="leaderboard-table">
         <table class="table">
            <tbody>
               <tr v-for="item in leadersData.slice().reverse()" v-bind:key="item.id">
                  <td>{{item.winner}}</td>
                  <td>{{item.date}}</td>
               </tr>
            </tbody>
         </table>
      </div>
   </div>
</template>

<script>
   import axios from 'axios';

   export default {
      props: ['initUpdate'],

      data() {
         return {
            leadersData: []
         };
      },

      watch: {
         initUpdate(val) {
            if (val == true) {
               this.setData();   
            }
         }
      },
         
      mounted() {
         this.setData();
      },

      methods: {
         setData() {
            axios
               .get('https://starnavi-frontend-test-task.herokuapp.com/winners')
               .then(response => (this.leadersData = response.data));
         }
      }
   }
</script>