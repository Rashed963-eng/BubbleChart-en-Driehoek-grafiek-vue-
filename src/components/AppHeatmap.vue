<template>
  <div class="main-content">
    <div class="heatmap-wrapper">
      <apexchart
        type="heatmap"
        :options="chartOptions"
        :series="series"
        width="100%"
        height="600"
      />
    </div>

    <div v-if="selectedCell" class="responses-list">
      <h3>
        Respondenten voor:
        "{{ selectedCell.yLabel }}" &amp; "{{ selectedCell.xLabel }}"
      </h3>
      <button @click="selectedCell = null" class="close-list">Sluit lijst</button>
      <ul>
        <li v-for="(resp, idx) in matchingResponses" :key="idx">
          <p><strong>Verhaal:</strong> {{ resp['01. Verhaalvraag'] }}</p>
          <p><strong>Gemeente:</strong> {{ resp['22. Gemeenten'] }}</p>
          <p><strong>Woningtype:</strong> {{ resp['19. Huur koop'] }}</p>
          <p><strong>Inkomen:</strong> {{ resp['21. Inkomen'] }}</p>
          <p><strong>Opleiding:</strong> {{ resp['20. Opleiding'] }}</p>
          <p><strong>Gender:</strong> {{ resp['18. Gender'] }}</p>
          <hr />
        </li>
      </ul>
    </div>

    <aside class="filters-panel">
      <div class="filter-group" v-for="(filter, key) in filters" :key="key">
        <details>
          <summary>{{ filter.label }}</summary>
          <div class="dropdown">
            <div
              v-for="option in filter.options"
              :key="option"
              class="dropdown-option"
              :class="{ selected: filter.selected.includes(option) }"
              @click="toggleOption(key, option)"
            >
              {{ option }}
            </div>
          </div>
        </details>
      </div>
      <button class="reset-button" @click="resetFilters">Reset filters</button>
    </aside>
  </div>
</template>

<script setup>
import { ref, onMounted, watch } from 'vue'
import Papa from 'papaparse'

const chartOptions = ref({})
const series = ref([])
const rawData = ref([])

const selectedCell = ref(null)
const matchingResponses = ref([])

const bins = ref([0, 0.10, 0.30, 0.60, 0.80, 1.00])
const labels1 = ref(['zeer gerust', 'gerust', 'neutraal', 'bezorgd', 'zeer bezorgd'])
const labels2 = ref(['heel goed idee', 'goed idee', 'neutraal', 'slecht idee', 'heel slecht idee'])

const filters = ref({
  gemeenten: {
    label: 'Gemeente',
    options: [
      'Aa en Hunze', 'Assen', 'Borger-Odoorn', 'Coevorden', 'De Wolden',
      'Emmen', 'Hoogeveen', 'Meppel', 'Midden-Drenthe', 'Noordenveld',
      'Tynaarlo', 'Westerveld'
    ],
    selected: []
  },
  woning: {
    label: 'Woningtype',
    options: ['Koopwoning', 'Sociale huurwoning', 'Particuliere huurwoning'],
    selected: []
  },
  inkomen: {
    label: 'Inkomen',
    options: [
      'Onder modaal',
      'Rondom modaal (€2350 netto per maand)',
      'Boven modaal'
    ],
    selected: []
  },
  opleiding: {
    label: 'Opleiding',
    options: [
      'Geen', 'Basisschool', 'Middelbare school',
      'Beroepsonderwijs (MBO / HBO)', 'Wetenschappelijk onderwijs (WO)'
    ],
    selected: []
  },
  gender: {
    label: 'Gender',
    options: ['Man', 'Vrouw', 'Non-binair', 'Wens niet te beantwoorden'],
    selected: []
  }
})

const toggleOption = (key, option) => {
  const selected = filters.value[key].selected
  const index = selected.indexOf(option)
  if (index >= 0) {
    selected.splice(index, 1)
  } else {
    selected.push(option)
  }
}

const resetFilters = () => {
  Object.values(filters.value).forEach(f => (f.selected = []))
}

const getCategory = (value, bins, labels) => {
  for (let i = 0; i < bins.length - 1; i++) {
    if (value >= bins[i] && value <= bins[i + 1]) return labels[i]
  }
  return null
}

const filterRow = row => {
  const match = (selected, value) => selected.length === 0 || selected.includes(value)
  return (
    match(filters.value.gemeenten.selected, row['22. Gemeenten']) &&
    match(filters.value.woning.selected, row['19. Huur koop']) &&
    match(filters.value.inkomen.selected, row['21. Inkomen']) &&
    match(filters.value.opleiding.selected, row['20. Opleiding']) &&
    match(filters.value.gender.selected, row['18. Gender'])
  )
}

const handleHeatmapClick = (event, chartContext, opts) => {
  const { seriesIndex, dataPointIndex } = opts
  if (seriesIndex == null || dataPointIndex == null) return

  const xLabel = series.value[seriesIndex]?.data[dataPointIndex]?.x
  const yLabel = series.value[seriesIndex]?.name


  selectedCell.value = { xLabel, yLabel }

  matchingResponses.value = rawData.value
    .filter(filterRow)
    .filter(row => {
      const x = row['08. Zorg lezen_Gerust']
      const y = row['10. In jouw dorp goed idee_Dat vind ik een heel goed idee']
      if (
        typeof x !== 'number' || typeof y !== 'number' ||
        x < 0 || x > 100 || y < 0 || y > 100
      ) return false

      const xCat = getCategory(x, bins.value, labels1.value)
      const yCat = getCategory(y, bins.value, labels2.value)

      return xCat === yLabel && yCat === xLabel
    })
}

const processData = (data) => {
  const xField = '08. Zorg lezen_Gerust'
  const yField = '10. In jouw dorp goed idee_Dat vind ik een heel goed idee'

  const matrix = {}
  for (const xLabel of labels1.value) {
    matrix[xLabel] = {}
    for (const yLabel of labels2.value) {
      matrix[xLabel][yLabel] = 0
    }
  }

  data.filter(filterRow).forEach(row => {
    const x = row[xField]
    const y = row[yField]
    if (x == null || y == null || x > 100 || y > 100) return

    const xCat = getCategory(x, bins.value, labels1.value)
    const yCat = getCategory(y, bins.value, labels2.value)

    if (xCat && yCat) {
      matrix[xCat][yCat]++
    }
  })

  series.value = labels1.value.map(xLabel => ({
    name: xLabel,
    data: labels2.value.map(yLabel => ({
      x: yLabel,
      y: matrix[xLabel][yLabel]
    }))
  }))

  chartOptions.value = {
    chart: {
      type: 'heatmap',
      toolbar: { show: false },
      events: {
        click: handleHeatmapClick
      }
    },
    dataLabels: {
      enabled: true,
      style: { colors: ['#000'] }
    },
    plotOptions: {
      heatmap: {
        shadeIntensity: 0.5,
        radius: 0,
        useFillColorAsStroke: true,
        distributed: false,
        colorScale: {
          ranges: [
            { from: 0, to: 9, color: '#bbd5fa' },
            { from: 10, to: 19, color: '#86a8d9' },
            { from: 20, to: 29, color: '#4b70a6' },
            { from: 30, to: 39, color: '#214273' },
            { from: 40, to: 50, color: '#071d3d' }
          ]
        }
      }
    },
    title: {
      text: 'Gerustheid t.o.v. de energietransitie vs. lokale energieplannen',
      align: 'center',
      style: { fontSize: '16px' }
    },
    xaxis: {
      type: 'category',
      title: { text: 'Zon- en windenergie in jouw dorp' }
    },
    yaxis: {
      title: { text: 'Gerustheid energietransitie' }
    }
  }
}

onMounted(() => {
  fetch('/ZonWindDrenthe.csv')
    .then(response => response.text())
    .then(csvText => {
      Papa.parse(csvText, {
        header: true,
        dynamicTyping: true,
        complete: (results) => {
          rawData.value = results.data
          processData(rawData.value)
        }
      })
    })
})

watch(
  [filters, bins, labels1, labels2],
  () => processData(rawData.value),
  { deep: true }
)
</script>



<style scoped>
.main-content {
  display: flex;
  gap: 10px;
  align-items: flex-start;
  justify-content: space-between;
  flex-wrap: wrap;
}

.heatmap-wrapper {
  flex: 1 1 600px;
}

.filters-panel {
  width: 250px;
  padding-top: 70px;
  padding-right: 30px;
  display: flex;
  flex-direction: column;
  gap: 16px;
  position: sticky;
  top: 20px;
  max-height: 90vh;
  overflow-y: auto;
}

@media (max-width: 768px) {
  .main-content:not(:has(.responses-list)) {
    flex-direction: column;
  }

  .filters-panel {
    width: 100%;
    border-left: none;
    border-top: 1px solid #ccc;
    padding-top: 20px;
  }
}

.filter-group details {
  border: 1px solid #bbb;
  border-radius: 8px;
  background-color: #f2f2f2;
  padding: 10px;
  font-size: 14px;
}

.filter-group summary {
  font-weight: 600;
  cursor: pointer;
}

.dropdown {
  margin-top: 8px;
  display: flex;
  flex-direction: column;
  gap: 6px;
  max-height: 200px;
  overflow-y: auto;
}

.dropdown-option {
  padding: 6px 10px;
  border-radius: 4px;
  cursor: pointer;
  background-color: #fff;
  border: 1px solid transparent;
  transition: background-color 0.2s, border 0.2s;
}

.dropdown-option:hover {
  background-color: #e6f0ff;
  border-color: #99c2ff;
}

.dropdown-option.selected {
  background-color: #d0e6ff;
  font-weight: bold;
  border-color: #6699ff;
}


.reset-button {
  height: 40px;
  background-color: #ddd;
  border: none;
  padding: 6px 12px;
  cursor: pointer;
  border-radius: 4px;
}

.responses-list {
  margin-top: 30px;
  padding: 20px;
  border-radius: 8px;
  flex: 1 1 100%;
}

.responses-list h3 {
  margin-top: 0;
  font-size: 18px;
}

.responses-list ul {
  list-style: none;
  padding-left: 0;
}

.responses-list li {
  margin-bottom: 16px;
  background-color: #fff;
  border: 1px solid #ccc;
  padding: 12px;
  border-radius: 6px;
  box-shadow: 0 2px 4px rgba(0,0,0,0.05);
}

.responses-list p {
  margin: 4px 0;
  font-size: 14px;
}

.close-list {
  background-color: #ccc;
  border: none;
  padding: 6px 12px;
  cursor: pointer;
  border-radius: 4px;
  margin-bottom: 12px;
}

</style>
