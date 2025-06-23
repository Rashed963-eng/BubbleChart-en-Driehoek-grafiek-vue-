<template>
    <div>
      <h2>Hoe belangrijk vind jij het gebruik van andere vormen van energie, zoals zon- en windenergie?</h2>
      <DetailChart
        v-if="score && stories.length"
        :score="score"
        :stories="stories"
        :color="scoreColors[score]"
      />
      <p v-else>Geen data gevonden voor score {{ score }}.</p>
    </div>
  </template>
  
  <script>
  import DetailChart from '@/components/DetailChart.vue';
  
  export default {
    components: { DetailChart },
    data() {
      return {
        score: null,
        stories: [],
        scoreColors: {
          1: '#ff9999',
          2: '#ffcc99',
          3: '#ffff99',
          4: '#ccff99',
          5: '#99ffcc'
        }
      };
    },
    async created() {
      this.score = Number(this.$route.params.score);
      // Wijziging hier: CSV-bestand in public/data.csv
      const res = await fetch('/data.csv');  // Pad is nu correct voor jouw indeling
      const text = await res.text();
      const allData = this.parseCSV(text);
      this.stories = allData
        .filter((item) => Number(item.score) === this.score)
        .map((item) => item.story);
    },
    methods: {
      parseCSV(text) {
        const lines = text.trim().split('\n').slice(1);
        return lines.map(row => {
          const [score, story] = row.split(',');
          return { score: Number(score), story };
        });
      }
    }
  };
  </script>
  