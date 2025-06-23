<template>
  <div class="chart-container">
    <!-- Grafiekweergave -->
    <div class="chart-wrapper">
      <canvas ref="chartCanvas"></canvas>
      <div v-if="hoveredStory" class="tooltip" :style="tooltipStyle">{{ hoveredStory }}</div>

      <div v-if="selectedStory" class="modal-overlay" @click.self="selectedStory = null">
        <div class="modal">
          <h3>Verhaal</h3>
          <p>{{ selectedStory }}</p>
          <button @click="selectedStory = null">Sluiten</button>
        </div>
      </div>
    </div>

    <!-- Filtermenu -->
    <div class="filters">
      <h3>Filters</h3>
      <div v-for="filter in filterOptions" :key="filter.name" class="filter-group">
        <label :for="'filter-' + filter.name">{{ filter.label }}</label>
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
import {
  Chart,
  BubbleController,
  LinearScale,
  PointElement,
  Tooltip,
  Title
} from 'chart.js';

Chart.register(BubbleController, LinearScale, PointElement, Tooltip, Title);

export default {
  props: ['score'],
  data() {
    return {
      chart: null,
      allData: [],
      hoveredStory: null,
      tooltipStyle: {},
      selectedStory: null,
      selectedFilters: {
        gemeente: '',
        woningtype: '',
        inkomen: '',
        opleiding: '',
        gender: ''
      },
      filterOptions: [
        { name: 'gemeente', label: 'Gemeente', options: [] },
        { name: 'woningtype', label: 'Woningtype', options: [] },
        { name: 'inkomen', label: 'Inkomen', options: [] },
        { name: 'opleiding', label: 'Opleiding', options: [] },
        { name: 'gender', label: 'Gender', options: [] }
      ]
    };
  },
  async mounted() {
    const res = await fetch('/data.csv');
    const text = await res.text();
    this.allData = this.parseCSV(text).filter(d => d.score === Number(this.score));
    this.extractFilterOptions();
    this.renderChart();
  },
  watch: {
    selectedFilters: {
      handler() {
        this.renderChart();
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
    extractFilterOptions() {
      for (const filter of this.filterOptions) {
        const values = [...new Set(this.allData.map(d => d[filter.name]).filter(Boolean))];
        filter.options = values.sort();
      }
    },
    filterData() {
      return this.allData.filter(d =>
        Object.keys(this.selectedFilters).every(key =>
          !this.selectedFilters[key] || d[key] === this.selectedFilters[key]
        )
      );
    },
    resetFilters() {
      for (const key in this.selectedFilters) {
        this.selectedFilters[key] = '';
      }
    },
    renderChart() {
      if (this.chart) this.chart.destroy();

      const selected = this.filterData();
      const data = selected.map(d => ({
        x: Math.random() * 100,
        y: Math.random() * 100,
        r: 8,
        backgroundColor: '#000',
        story: d.story
      }));

      this.chart = new Chart(this.$refs.chartCanvas, {
        type: 'bubble',
        data: { datasets: [{ data }] },
        options: {
          onHover: (e, elements) => {
            if (elements.length > 0) {
              const index = elements[0].index;
              this.hoveredStory = data[index].story;
              this.tooltipStyle = {
                left: e.clientX + 10 + 'px',
                top: e.clientY + 10 + 'px',
                position: 'fixed',
                backgroundColor: '#000',
                border: '1px solid #ccc',
                padding: '10px',
                maxWidth: '300px',
                zIndex: 1000
              };
            } else {
              this.hoveredStory = null;
            }
          },
          onClick: (e, elements) => {
            if (elements.length > 0) {
              const index = elements[0].index;
              this.selectedStory = data[index].story;
            }
          },
          plugins: { tooltip: { enabled: false },
        datalabels: {
    display: false 
  } 
        },
          scales: {
            x: {
              min: 0,
              max: 100,
              title: { display: false, text: 'Score distributie' },
              ticks: { stepSize: 30, display:false },
              grid: {
                display:false,
                drawBorder:true,

              }
            },
            y: {
              min: 0,
              max: 100,
              title: { display: false, text: 'Waarde' },
              ticks: { stepSize: 20, display:false},
              grid: {
                display:false,
                drawBorder:true,

              }
            }
          }
        }
      });
    },
    downloadChart() {
      const canvas = this.$refs.chartCanvas;
      const link = document.createElement('a');
      const ctx = canvas.getContext('2d');

      // Voeg witte achtergrond toe
      const original = ctx.getImageData(0, 0, canvas.width, canvas.height);
      ctx.save();
      ctx.globalCompositeOperation = 'destination-over';
      ctx.fillStyle = '#ffffff';
      ctx.fillRect(0, 0, canvas.width, canvas.height);
      ctx.restore();

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

.filters {
  width: 250px;
  background-color: #f9f9f9;
  padding: 20px;
  border-radius: 8px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}

.filters h3 {
  font-size: 1.2rem;
  margin-bottom: 15px;
}

.filter-group {
  margin-bottom: 15px;
}

.filter-group label {
  font-size: 1rem;
  margin-bottom: 8px;
  display: block;
}

.filter-group select {
  width: 100%;
  padding: 8px;
  border-radius: 5px;
  border: 1px solid #ddd;
  background-color: #fff;
}
.button-row {
  display: flex;
  gap: 10px;
  margin-top: 20px;
}

.reset-button,
.download-button {
  margin-top: 15px;
  width: 100%;
  padding: 10px;
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
  background-color: #007bff;
  color: #fff;
}

.download-button:hover {
  background-color: #0056b3;
}

.tooltip {
  position: absolute;
  background-color: white;
  padding: 10px;
  border: 1px solid #ccc;
  border-radius: 5px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.2);
}

.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.6);
  display: flex;
  justify-content: center;
  align-items: center;
}

.modal {
  background-color: white;
  padding: 20px;
  border-radius: 8px;
  max-width: 500px;
}

.modal button {
  margin-top: 20px;
  background-color: #007bff;
  color: white;
  padding: 10px 16px;
  border: none;
  border-radius: 6px;
  cursor: pointer;
}

.modal button:hover {
  background-color: #0056b3;
}
</style>
