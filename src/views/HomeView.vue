<template>
 
    <div>
      <AppNavbar />
      <AppHeader />
      <AppHeatmap />

      <h2>Hoe belangrijk vind jij het gebruik van andere vormen van energie, zoals zon- en windenergie?</h2>
      <GlobalChart />
      <hr class="light-separator"><br><br>
      <div class="centered-chart">
  <AppDriehoek />
</div>

    </div><br>
    
  </template>
  
  <script>
  
import AppHeader from '@/components/AppHeader.vue';
import GlobalChart from '@/components/GlobalChart.vue';
import AppNavbar from '@/components/AppNavbar.vue';
import AppDriehoek from '@/components/AppDriehoek.vue';

  
  export default {
    components: {
      AppNavbar,
      AppHeader,
      GlobalChart,
      AppDriehoek
  
     
    }
  }
  </script>
  

  <style scoped>
.centered-chart {
  display: flex;
  justify-content: center;

}
</style>
