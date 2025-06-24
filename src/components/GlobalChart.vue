<template>
  <div class="chart-container">
    
    <!-- Grafiek -->
    <div class="chart-wrapper">
      <canvas ref="chartCanvas"></canvas>
      <div class="legend">
        <div v-for="score in [1, 2, 3, 4, 5, 6, 7]" :key="score" class="legend-item">
          <div class="legend-color" :style="{ backgroundColor: scoreColors[score] }"></div>
          <span>{{ scoreLabels[score] }}</span>
        </div>
      </div>
    </div>

    <!-- Filtermenu 1-->
    <div class="filter-sidebar">
      <h3>Filters</h3>
      <div v-for="filter in filterOptions" :key="filter.name" class="filter-group">
        <label for="filter-{{ filter.name }}">{{ filter.label }}</label>
        <select v-model="selectedFilters[filter.name]" :id="'filter-' + filter.name">
          <option value="">Alles</option>
          <option v-for="option in filter.options" :key="option" :value="option">{{ option }}</option>
        </select>
      </div>

      <div class="button-row">
        <button class="reset-button" @click="resetFilters">Reset filters</button>
        <button @click="downloadChart" class="download-button">Download als afbeelding</button>
      </div>
    </div>
  </div>
</template>

<script>
import ChartDataLabels from 'chartjs-plugin-datalabels';
import {
  Chart,
  BubbleController,
  LinearScale,
  PointElement,
  Tooltip,
  Title
} from 'chart.js';

Chart.register(ChartDataLabels, BubbleController, LinearScale, PointElement, Tooltip, Title);

export default {
  name: 'GlobalChart',
  data() {
    return {
      chart: null,
      allData: [],
      groupedData: {},
      selectedFilters: {
        gemeente: '',
        woningtype: '',
        inkomen: '',
        opleiding: '',
        gender: ''
      },
      filterOptions: [
        { name: 'gemeente', label: 'Gemeente', options: [
          'Westerveld', 'Assen', 'Borger-Odoorn', 'De Wolden', 'Tynaarlo', 'Coevorden', 'Aa en Hunze', 
          'Hoogeveen', 'Emmen'
        ]},
        { name: 'woningtype', label: 'Woningtype', options: [
          'Huurwoning', 'Particuliere huurwoning', 'Koopwoning', 'Sociale huurwoning'
        ]},
        { name: 'inkomen', label: 'Inkomen', options: [
          'Onder modaal', 'Boven modaal', 'Rondom modaal'
        ]},
        { name: 'opleiding', label: 'Opleiding', options: [
          'Middelbare school', 'Beroepsonderwijs', 'Basisschool', 'Wetenschappelijk onderwijs'
        ]},
        { name: 'gender', label: 'Gender', options: [
          'Man', 'Vrouw','Anders','Geen antwoord'
        ]}
      ],
       //      1: '#ff9999',
  //      2: '#ffcc99',
  //      3: '#ffff99',
  //      4: '#ccff99',
  //      5: '#99ffcc' 
      scoreColors: {
  1: '#cc0000', // rood
  2: '#e67e22', // oranje
  3: '#f1c40f', // geel
  4: '#b2d732', // lichtgroen
  5: '#2ecc71', // groen
  6: '#27ae60', // donkergroen
  7: '#1e8449'  // nog donkerder groen
},
scoreLabels: {
  1: 'Absoluut niet belangrijk',
  2: 'Niet belangrijk',
  3: 'Een beetje belangrijk',
  4: 'Neutraal',
  5: 'Redelijk belangrijk',
  6: 'Belangrijk',
  7: 'Enorm belangrijk',
},


    };
  },
  async mounted() {
    const res = await fetch('/data.csv');
    const text = await res.text();
    this.allData = this.parseCSV(text);
    this.updateChartData();
  },
  watch: {
    selectedFilters: {
      handler() {
        this.updateChartData();
      },
      deep: true
    }
  },
  methods: {
    parseCSV(text) {
      const lines = text.trim().split('\n');
      const headers = lines[0].split(',');
      return lines.slice(1).map(line => {
        const values = line.split(',');
        const obj = {};
        headers.forEach((h, i) => {
          obj[h.trim()] = values[i] ? values[i].trim() : '';
        });
        obj.score = Number(obj.score);
        return obj;
      });
    },
    filterData() {
      return this.allData.filter(d => {
        return Object.keys(this.selectedFilters).every(key => {
          return !this.selectedFilters[key] || d[key] === this.selectedFilters[key];
        });
      });
    },
    groupByScore(data) {
      return data.reduce((acc, item) => {
        if (!acc[item.score]) acc[item.score] = [];
        acc[item.score].push(item);
        return acc;
      }, {});
    },
    updateChartData() {
      const filtered = this.filterData();
      this.groupedData = this.groupByScore(filtered);
      this.renderChart();
    },
    resetFilters() {
      for (const key in this.selectedFilters) {
        this.selectedFilters[key] = '';
      }
    },
    renderChart() {
  if (this.chart) this.chart.destroy();

  const scores = [1, 2, 3, 4, 5, 6, 7];
  const filtered = this.filterData();
  const total = filtered.length;
  const maxCount = Math.max(...scores.map(s => (this.groupedData[s] || []).length)) || 1;
  const spacing = 100 / scores.length;

  const datasets = scores.map((score, i) => { 
    const count = (this.groupedData[score] || []).length;
    const percentage = total > 0 ? Math.round((count / total) * 100) : 0;
    const relativeSize = 10 + (count / maxCount) * 100;
    return {
      label: this.scoreLabels[score],
      data: [{
        x: (i + 1) * spacing,
        y: 50,
        r: relativeSize,
        percentage,
        score
      }],
      backgroundColor: this.scoreColors[score]
    };
  });

  this.chart = new Chart(this.$refs.chartCanvas, {
    type: 'bubble',
    data: { datasets },
    options: {
      plugins: {
        tooltip: { enabled: false },
        datalabels: {
          color: '#ffffff',
          font: {
            weight: 'bold',
            size: 14
          },
          formatter: (value) => `${value.percentage}%`
        }
      },
      onClick: (e, elements) => {
        if (elements.length > 0) {
          const clicked = elements[0];
          const score = datasets[clicked.datasetIndex].data[0].score;
          this.$router.push({ path: `/detail/${score}`, query: this.selectedFilters });
        }
      },
      scales: {
        x: {
          min: 0,
          max: 120,
          title: { display: true, text: 'Score distributie' },
          ticks: { stepSize: 30, display: false },
          grid: { drawBorder: true, display: false }
        },
        y: {
          min: 0,
          max: 100,
          title: { display: false },
          ticks: { stepSize: 20, display: false },
          grid: { display: false, drawBorder: true }
        }
      }
    },
    plugins: [ChartDataLabels]
  });
}
,
    downloadChart() {
      const canvas = this.$refs.chartCanvas;
      const ctx = canvas.getContext('2d');

      const original = ctx.getImageData(0, 0, canvas.width, canvas.height);

      ctx.save();
      ctx.globalCompositeOperation = 'destination-over';
      ctx.fillStyle = '#ffffff';
      ctx.fillRect(0, 0, canvas.width, canvas.height);
      ctx.restore();

      const link = document.createElement('a');
      link.download = 'bubblechart.png';
      link.href = canvas.toDataURL('image/png');
      link.click();

      ctx.putImageData(original, 0, 0);
    }
  }
};
</script>

<style scoped>
.chart-container {
  display: flex;
  gap: 40px;
  margin-top: 20px;
}

.chart-wrapper {
  flex: 1;
  position: relative;
}

.legend {
  display: flex;
  justify-content: center;
  margin-top: 20px;
}

.legend-item {
  display: flex;
  align-items: center;
  margin: 0 10px;
}

.legend-color {
  width: 15px;
  height: 15px;
  margin-right: 6px;
  border-radius: 50%;
}

.filter-sidebar {
  display: flex;
  flex-direction: column;
  width: 250px;
  background-color: #f9f9f9;
  padding: 20px;
  border-radius: 8px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}

.filter-sidebar h3 {
  font-size: 1.2rem;
  margin-bottom: 15px;
  color: #333;
}

.filter-group {
  margin-bottom: 15px;
}
.filter-group label {
  font-size: 1rem;
  margin-bottom: 8px;
  display: block;
}

label {
  font-size: 1rem;
  margin-bottom: 8px;
  display: block;
  color: #555;
}

select {
  width: 100%;
  padding: 8px;
  border-radius: 5px;
  border: 1px solid #ddd;
  background-color: #fff;
  font-size: 1rem;
}

.button-row {
  display: flex;
  gap: 10px;
  margin-top: 20px;
}

.reset-button,
.download-button {
  flex: 1;
  padding: 10px;
  width:100%;
  font-size: 1rem;
  border: none;
  border-radius: 5px;
  cursor: pointer;
}

.reset-button {
  background-color: #ff4444;
  color: #fff;
}

.reset-button:hover {
  background-color: #ff0000;
}

.download-button {
  background-color: #0080ff;
  color: white;
}

.download-button:hover {
  background-color: #005fcc;
}



</style>
