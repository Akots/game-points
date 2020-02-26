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
   export default {
      props: ['initLeadersData'],

      data() {
         return {
            leadersData: this.initLeadersData
         };
      }
   }
</script>