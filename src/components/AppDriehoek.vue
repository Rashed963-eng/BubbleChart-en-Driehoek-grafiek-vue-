<template>
  <div class="visualization-container-wrapper">
    <div ref="container" class="visualization-container">
      <div ref="konvaContainer" class="konva-container"></div>
      <!-- Download knop onder de grafiek -->
      <button @click="downloadImage" class="download-button-below-graph">
        Afbeelding Downloaden
      </button>
      <div v-if="selectedDataPoint" class="selected-data">
        <h3>Verhaal bij Bolletje</h3>
        <template v-if="selectedDataPoint.isLoading">
          <p>Verhaal wordt geladen...</p>
        </template>
        <template v-else>
          <img src="" alt="Speech Bubble" class="speech-bubble-image">
          <pre class="speech-bubble-text">{{ selectedDataPoint.verhaal }}</pre>
        </template>
      </div>
    </div>
    <div class="filter-options">
      <h3>Filter Opties</h3>
      <div v-for="(values, key) in filterOpties" :key="key" class="filter-group">
        <h4>{{ key }}</h4>
        <div v-for="value in values" :key="value" class="filter-item">
          <div
            class="color-indicator"
            :style="{ backgroundColor: filterColors[key]?.[value] || 'gray' }"
            @click="toggleFilter(key, value)"
            :class="{ active: activeFilters[key]?.includes(value) }"
          ></div>
          <label>{{ value }}</label>
        </div>
      </div>
      <button @click="clearAllFilters" v-if="Object.values(activeFilters).flat().length > 0">
        Filters Wissen
      </button>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, computed, watch } from 'vue';
import Konva from 'konva';

const container = ref(null);
const konvaContainer = ref(null);
let stage;
let layer;

const rawData = ref([
  { as1: 0.2, as2: 0.3, as3: 0.5, gemeente: 'Aa en Hunze', woningtype: 'Koopwoning', inkomen: 'Onder modaal', opleiding: 'Basisschool', gender: 'Man', verhaal: '', argument: 'geld' },
  { as1: 0.7, as2: 0.1, as3: 0.2, gemeente: 'Assen', woningtype: 'Huurwoning', inkomen: 'Rondom modaal', opleiding: 'Middelbare school', gender: 'Vrouw', verhaal: '', argument: 'klimaat' },
  { as1: 0.1, as2: 0.8, as3: 0.1, gemeente: 'Emmen', woningtype: 'Appartement', inkomen: 'Boven modaal', opleiding: 'Beroepsonderwijs', gender: 'Non-binair', verhaal: '', argument: 'positief' },
  { as1: 0.4, as2: 0.4, as3: 0.2, gemeente: 'Assen', woningtype: 'Koopwoning', inkomen: 'Onder modaal', opleiding: 'Wetenschappelijk onderwijs', gender: 'Man', verhaal: '', argument: 'anders' },
  ...Array.from({ length: 10000 }, () => {
    let a = Math.random();
    let b = Math.random() * (1 - a);
    let c = 1 - a - b;
    const gemeentes = ['Aa en Hunze', 'Assen', 'Emmen', 'Borger-Odoorn', 'Coevorden', 'De Wolden', 'Hoogeveen', 'Meppel', 'Midden-Drenthe', 'Noordenveld', 'Tynaarlo', 'Westerveld'];
    const woningtypes = ['Koopwoning', 'Huurwoning', 'Appartement', 'Sociale huurwoning', 'Particuliere huurwoning'];
    const inkomens = ['Onder modaal', 'Rondom modaal', 'Boven modaal'];
    const opleidingen = ['Geen', 'Basisschool', 'Middelbare school', 'Beroepsonderwijs', 'Wetenschappelijk onderwijs'];
    const genders = ['Man', 'Vrouw', 'Non-binair', 'Wens niet te beantwoorden'];
    const argumenten = ['geld', 'klimaat', 'positief', 'anders'];
    return {
      as1: a,
      as2: b,
      as3: c,
      gemeente: gemeentes[Math.floor(Math.random() * gemeentes.length)],
      woningtype: woningtypes[Math.floor(Math.random() * woningtypes.length)],
      inkomen: inkomens[Math.floor(Math.random() * inkomens.length)],
      opleiding: opleidingen[Math.floor(Math.random() * inkomens.length)],
      gender: genders[Math.floor(Math.random() * genders.length)],
      verhaal: '',
      argument: argumenten[Math.floor(Math.random() * argumenten.length)],
    };
  }),
]);

const filterOpties = ref({
  Gemeenten: [...new Set(rawData.value.map(item => item.gemeente))].sort(),
  'Type woning': [...new Set(rawData.value.map(item => item.woningtype))].sort(),
  Inkomen: [...new Set(rawData.value.map(item => item.inkomen))].sort(),
  Opleiding: [...new Set(rawData.value.map(item => item.opleiding))].sort(),
  Gender: [...new Set(rawData.value.map(item => item.gender))].sort(),
});

const filterColors = ref({
  Gemeenten: {
    'Aa en Hunze': '#e41a1c', 'Assen': '#377eb8', 'Borger-Odoorn': '#4daf4a', 'Coevorden': '#984ea3',
    'De Wolden': '#ff7f00', 'Emmen': '#ffff33', 'Hoogeveen': '#a65628', 'Meppel': '#f781bf',
    'Midden-Drenthe': '#999999', 'Noordenveld': '#1b9e77', 'Tynaarlo': '#d95f02', 'Westerveld': '#7570b3',
  },
  'Type woning': {
    'Koopwoning': '#a6cee3', 'Huurwoning': '#1f78b4', 'Appartement': '#b2df8a',
    'Sociale huurwoning': '#33a02c', 'Particuliere huurwoning': '#fb9a99',
  },
  Inkomen: {
    'Onder modaal': '#fdbf6f', 'Rondom modaal': '#ff7f00', 'Boven modaal': '#cab2d6',
  },
  Opleiding: {
    'Geen': '#6a3d9a', 'Basisschool': '#ffff99', 'Middelbare school': '#b15928',
    'Beroepsonderwijs': '#a6cee3', 'Wetenschappelijk onderwijs': '#1f78b4',
  },
  Gender: {
    'Man': '#e31a1c', 'Vrouw': '#1f78b4', 'Non-binair': '#33a02c', 'Wens niet te beantwoorden': '#ff7f00',
  },
});

const activeFilters = ref({});
const selectedDataPoint = ref(null);

const filteredData = computed(() => {
  return rawData.value.filter(item => {
    return Object.keys(activeFilters.value).every(key => {
      const selectedValues = activeFilters.value[key] || [];
      if (selectedValues.length === 0) {
        return true;
      }
      const dataKey = key === 'Gemeenten' ? 'gemeente' : key === 'Type woning' ? 'woningtype' : key.toLowerCase();
      return selectedValues.includes(item[dataKey]);
    });
  });
});

function projectToTriangle(a, b, c, width, height) {
  const p1 = { x: width * 0.5, y: height * 0.95 };
  const p2 = { x: width * 0.95, y: height * 0.05 };
  const p3 = { x: width * 0.05, y: height * 0.05 };

  const sum = a + b + c;
  const normA = a / sum;
  const normB = b / sum;
  const normC = c / sum;

  const x = normA * p1.x + normB * p2.x + normC * p3.x;
  const y = normA * p1.y + normB * p2.y + normC * p3.y;

  return { x, y };
}

function renderPoints() {
  if (!layer || !stage) return;
  layer.destroyChildren();

  const width = stage.width();
  const height = stage.height();
  const p1 = { x: width * 0.5, y: height * 0.95 };
  const p2 = { x: width * 0.95, y: height * 0.05 };
  const p3 = { x: width * 0.05, y: height * 0.05 };
  const trianglePoints = [p1.x, p1.y, p2.x, p2.y, p3.x, p3.y];

  const triangle = new Konva.Line({
    points: trianglePoints,
    stroke: 'black',
    strokeWidth: 3,
    closed: true,
  });
  layer.add(triangle);

  const labelConfig = {
    fontSize: 14,
    fill: 'black',
    align: 'center',
    verticalAlign: 'middle',
  };
  const labelOffset = 10;

  const labels = [
    { text: 'Het levert geld op / het kost niets', x: p1.x, y: p1.y + labelOffset - 0, offsetX: 210 - ('Het levert geld op / het kost niets'.length * 3), align: 'center' },
    { text: 'Het bijdraagt aan een beter klimaat', x: p2.x, y: p2.y, offsetX: 0, offsetY: - 0, align: 'left' },
    { text: 'Anderen in de buurt moeten er positief over zijn', x: p3.x, y: p3.y, offsetX: 300, offsetY: - 0, align: 'right' },
  ];

  labels.forEach(labelData => {
    const label = new Konva.Text({ ...labelConfig, ...labelData });
    layer.add(label);
  });

  filteredData.value.forEach(d => {
    const { x, y } = projectToTriangle(d.as1, d.as2, d.as3, width, height);
    const fillColor = filterColors.value['Gemeenten']?.[d.gemeente] || 'gray';

    const circle = new Konva.Circle({
      x: x,
      y: y,
      radius: 0.5,
      fill: fillColor,
      opacity: 0.8,
      draggable: false,
      dataPoint: d,
    });

    circle.on('click', async function () {
      const clickedDataPoint = this.attrs.dataPoint;

      // Controleer of het verhaal al bestaat en geladen is
      if (clickedDataPoint.verhaal && clickedDataPoint.verhaal !== 'Laden...' && clickedDataPoint.verhaal !== 'Verhaal niet gevonden.' && clickedDataPoint.verhaal !== 'Er is een technische fout opgetreden bij het genereren van het verhaal.') {
        selectedDataPoint.value = clickedDataPoint;
        return;
      }

      // Toon laadstatus
      selectedDataPoint.value = { ...clickedDataPoint, verhaal: 'Laden...', isLoading: true };

      try {
        const generatedText = await generateStoryForDataPoint(clickedDataPoint);
        const index = rawData.value.findIndex(item => item === clickedDataPoint);
        if (index !== -1) {
          rawData.value[index].verhaal = generatedText;
        }
        selectedDataPoint.value = { ...clickedDataPoint, verhaal: generatedText, isLoading: false };
      } catch (error) {
        console.error("Fout bij genereren verhaal:", error);
        const errorText = "Er is een technische fout opgetreden bij het genereren van het verhaal.";
        const index = rawData.value.findIndex(item => item === clickedDataPoint);
        if (index !== -1) {
          rawData.value[index].verhaal = errorText;
        }
        selectedDataPoint.value = { ...clickedDataPoint, verhaal: errorText, isLoading: false };
      }
    });

    circle.on('mouseover', function () {
      document.body.style.cursor = 'pointer';
    });
    circle.on('mouseout', function () {
      document.body.style.cursor = 'default';
    });

    layer.add(circle);
  });

  layer.draw();
}

function toggleFilter(key, value) {
  if (!activeFilters.value[key]) {
    activeFilters.value[key] = [];
  }
  if (activeFilters.value[key].includes(value)) {
    activeFilters.value[key] = activeFilters.value[key].filter(v => v !== value);
    if (activeFilters.value[key].length === 0) {
      delete activeFilters.value[key];
    }
  } else {
    activeFilters.value[key].push(value);
  }
}

function clearAllFilters() {
  activeFilters.value = {};
}

async function generateStoryForDataPoint(dataPoint) {
  const prompt = `Genereer een kort, empathisch en relevant verhaal (ongeveer 50-70 woorden) over de ervaring van een persoon in Drenthe die spreekt over onderwerpen als geld, klimaat en gemeenschap, gebaseerd op de volgende kenmerken:
    Gemeente: ${dataPoint.gemeente}
    Woningtype: ${dataPoint.woningtype}
    Inkomen: ${dataPoint.inkomen}
    Opleiding: ${dataPoint.opleiding}
    Geslacht: ${dataPoint.gender}
    Relevant argument/focus: ${dataPoint.argument || 'geen specifieke focus'}

    Het verhaal moet de persoonlijke beleving weerspiegelen, geen feitenoverzicht zijn. Begin direct met het verhaal.`;

  let chatHistory = [];
  chatHistory.push({ role: "user", parts: [{ text: prompt }] });
  const payload = { contents: chatHistory };
  const apiKey = ""; // Canvas zal dit automatisch invullen

  try {
    const apiUrl = `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent?key=${apiKey}`;
    const response = await fetch(apiUrl, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(payload)
    });
    const result = await response.json();

    if (result.candidates && result.candidates.length > 0 &&
        result.candidates[0].content && result.candidates[0].content.parts &&
        result.candidates[0].content.parts.length > 0) {
      return result.candidates[0].content.parts[0].text;
    } else {
      console.warn("Geen geldige respons van de Gemini API:", result);
      // Controleer op promptFeedback voor meer specifieke foutmelding
      if (result.promptFeedback && result.promptFeedback.blockReason) {
          return `Verhaal geblokkeerd: ${result.promptFeedback.blockReason}.`;
      }
      return "Kon geen verhaal genereren. Probeer opnieuw.";
    }
  } catch (error) {
    console.error("Fout bij aanroepen Gemini API:", error);
    return "Er is een technische fout opgetreden bij het genereren van het verhaal.";
  }
}

function downloadImage() {
  if (!stage) {
    console.error("Konva stage is niet geïnitialiseerd.");
    return;
  }
  stage.toDataURL({
    mimeType: 'image/png',
    quality: 1,
    callback: function(dataUrl) {
      const link = document.createElement('a');
      link.download = 'ternaire_visualisatie.png';
      link.href = dataUrl;
      document.body.appendChild(link);
      link.click();
      document.body.removeChild(link);
    }
  });
}

onMounted(() => {
  const width = container.value.offsetWidth;
  const height = container.value.offsetHeight;

  stage = new Konva.Stage({
    container: konvaContainer.value,
    width: width,
    height: height,
    draggable: true,
  });
  layer = new Konva.Layer();
  stage.add(layer);

  renderPoints();

  watch(filteredData, renderPoints);

  stage.on('wheel', (event) => {
    event.evt.preventDefault();
    const scaleBy = 1.05;
    const oldScale = stage.scaleX();
    const pointer = stage.getPointerPosition();
    const newScale = event.evt.deltaY < 0 ? oldScale * scaleBy : oldScale / scaleBy;
    stage.scale({ x: newScale, y: newScale });
    const newPos = {
      x: pointer.x - (pointer.x - stage.x()) * newScale / oldScale,
      y: pointer.y - (pointer.y - stage.y()) * newScale / oldScale,
    };
    stage.position(newPos);
    stage.batchDraw();
  });
});
</script>

<style scoped>
.centered-chart {
  display: flex;
  justify-content: center;
  align-items: center;
}




.visualization-container-wrapper {
  display: flex;
  align-items: flex-start;
}

.visualization-container {
  width: 800px;
  height: auto;
  border: 1px solid lightgray;
  margin-right: 20px;
  position: relative;
}

.konva-container {
  width: 100%;
  height: 600px;
  cursor: grab;
  border-radius: 8px;
  overflow: hidden;
}

.konva-container.konva-dragging {
  cursor: grabbing;
}

.filter-options {
  padding: 15px;
  border: 1px solid lightgray;
  background-color: #f9f9f9;
  width: 250px;
  box-sizing: border-box;
  border-radius: 8px;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
}

.filter-group {
  margin-bottom: 20px;
  padding-bottom: 15px;
  border-bottom: 1px dashed #e0e0e0;
}

.filter-group:last-child {
  border-bottom: none;
  margin-bottom: 0;
  padding-bottom: 0;
}

.filter-group h4 {
  margin-top: 0;
  margin-bottom: 10px;
  color: #333;
  font-size: 1.1em;
}

.filter-item {
  display: flex;
  align-items: center;
  margin-bottom: 8px;
  cursor: pointer;
}

.filter-item:hover .color-indicator {
    border-color: #555;
}


.color-indicator {
  width: 16px;
  height: 16px;
  border-radius: 50%;
  margin-right: 10px;
  cursor: pointer;
  border: 1px solid #ccc;
  transition: border-color 0.2s ease-in-out;
}

.color-indicator.active {
  border: 2px solid #000;
  box-shadow: 0 0 5px rgba(0, 0, 0, 0.3);
}

.filter-options label {
  font-size: 14px;
  color: #555;
  cursor: pointer;
}

.filter-options button {
  margin-top: 15px;
  padding: 10px 15px;
  cursor: pointer;
  background-color: #007bff;
  color: white;
  border: none;
  border-radius: 5px;
  font-size: 14px;
  transition: background-color 0.2s ease-in-out, transform 0.1s ease;
  width: 100%;
  box-sizing: border-box;
}

.filter-options button:hover {
  background-color: #0056b3;
  transform: translateY(-1px);
}

.filter-options button:active {
  transform: translateY(1px);
}

.download-button-below-graph {
    display: block;
    margin: 20px auto 0 auto;
    padding: 10px 20px;
    background-color: #28a745;
    color: white;
    border: none;
    border-radius: 5px;
    font-size: 16px;
    cursor: pointer;
    transition: background-color 0.2s ease-in-out;
    width: fit-content;
}

.download-button-below-graph:hover {
    background-color: #218838;
}

.selected-data {
  margin-top: 20px;
  padding: 20px;
  border: 1px solid #e0e0e0;
  background-color: #ffffff;
  border-radius: 10px;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.05);
  display: flex;
  flex-direction: column;
  align-items: center;
  text-align: center;
  max-width: 760px;
  margin-left: auto;
  margin-right: auto;
}

.selected-data h3 {
  margin-top: 0;
  color: #333;
}

.speech-bubble-text {
  white-space: pre-wrap;
  word-wrap: break-word;
  max-width: 100%;
  font-size: 0.95em;
  line-height: 1.5;
  color: #333;
  padding: 15px 20px;
  background-color: #e0f7fa;
  border: 1px solid #b2ebf2;
  border-radius: 15px;
  position: relative;
  margin-bottom: 15px;
}

.speech-bubble-text::after {
  content: '';
  position: absolute;
  bottom: -15px;
  left: 50%;
  transform: translateX(-50%);
  width: 0;
  height: 0;
  border-left: 15px solid transparent;
  border-right: 15px solid transparent;
  border-top: 15px solid #e0f7fa;
}
.speech-bubble-text::before {
  content: '';
  position: absolute;
  bottom: -16px;
  left: 50%;
  transform: translateX(-50%);
  width: 0;
  height: 0;
  border-left: 16px solid transparent;
  border-right: 16px solid transparent;
  border-top: 16px solid #b2ebf2;
}
</style>
