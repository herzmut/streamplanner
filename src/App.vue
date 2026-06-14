<script setup>
import { ref, watch, onMounted } from 'vue';
import AJV from 'ajv';
import ConfigTab from './components/ConfigTab.vue';
import PlannerTab from './components/PlannerTab.vue';

const ajv = new AJV();

const configSchema = {
  type: 'object',
  properties: {
    templateImage: { type: ['string', 'null'] },
    boxes: {
      type: 'array',
      items: {
        type: 'object',
        properties: {
          x: { type: 'number' },
          y: { type: 'number' },
          width: { type: 'number' },
          height: { type: 'number' },
          label: { type: 'string' }
        },
        required: ['x', 'y', 'width', 'height']
      }
    },
    globalFont: { type: 'string' },
    globalFontWeight: { type: 'number' },
    globalCss: { type: 'string' },
    version: { type: 'number' }
  },
  additionalProperties: true
};

const validate = ajv.compile(configSchema);

const STORAGE_KEY = 'streamplanner_settings';

const activeTab = ref('planner');
const templateImage = ref(null);
const boxes = ref([]);
const globalFont = ref('Roboto');
const globalFontWeight = ref(400);
const globalCss = ref('');

// Load settings from localStorage
onMounted(() => {
  const saved = localStorage.getItem(STORAGE_KEY);
  if (saved) {
    try {
      const config = JSON.parse(saved);
      if (config.templateImage) templateImage.value = config.templateImage;
      if (config.boxes) boxes.value = config.boxes;
      if (config.globalFont) globalFont.value = config.globalFont;
      if (config.globalFontWeight) globalFontWeight.value = config.globalFontWeight;
      if (config.globalCss) globalCss.value = config.globalCss;
    } catch (e) {
      console.error('Failed to load settings from localStorage', e);
    }
  }
});

// Watch for changes and save to localStorage
watch(
  [templateImage, boxes, globalFont, globalFontWeight, globalCss],
  () => {
    const config = {
      templateImage: templateImage.value,
      boxes: boxes.value,
      globalFont: globalFont.value,
      globalFontWeight: globalFontWeight.value,
      globalCss: globalCss.value,
    };
    try {
      localStorage.setItem(STORAGE_KEY, JSON.stringify(config));
    } catch (e) {
      console.error('Failed to save settings to localStorage', e);
      // Handle quota exceeded error if image is too large
      if (e.name === 'QuotaExceededError') {
        alert('Speicherlimit im Browser überschritten. Das Hintergrundbild ist möglicherweise zu groß.');
      }
    }
  },
  { deep: true }
);

const handleImageUpload = (imageData) => {
  templateImage.value = imageData;
};

const updateBoxes = (newBoxes) => {
  boxes.value = newBoxes;
};

const updateConfig = (config) => {
  globalFont.value = config.font;
  globalFontWeight.value = config.fontWeight;
  globalCss.value = config.css;
};

const exportToClipboard = async () => {
  const config = {
    templateImage: templateImage.value,
    boxes: boxes.value,
    globalFont: globalFont.value,
    globalFontWeight: globalFontWeight.value,
    globalCss: globalCss.value,
    version: 1
  };
  try {
    await navigator.clipboard.writeText(JSON.stringify(config));
    alert('Einstellungen wurden in die Zwischenablage kopiert.');
  } catch (err) {
    console.error('Fehler beim Kopieren in die Zwischenablage:', err);
    alert('Fehler beim Exportieren.');
  }
};

const importFromClipboard = async () => {
  try {
    const text = await navigator.clipboard.readText();
    const config = JSON.parse(text);


    // Validierung mit JSON Schema
    const valid = validate(config);
    if (!valid) {
      const errors = validate.errors.map(err => `${err.instancePath} ${err.message}`).join(', ');
      console.error('Fehler beim Validieren:', errors);
      alert('Ungültiges Format');
    }
    
    // Anwenden der Werte
    if (config.templateImage !== undefined) templateImage.value = config.templateImage;
    if (config.boxes !== undefined) boxes.value = config.boxes;
    if (config.globalFont !== undefined) globalFont.value = config.globalFont;
    if (config.globalFontWeight !== undefined) globalFontWeight.value = config.globalFontWeight;
    if (config.globalCss !== undefined) globalCss.value = config.globalCss;
    
    alert('Einstellungen erfolgreich importiert.');
  } catch (err) {
    console.error('Fehler beim Importieren:', err);
    alert('Fehler beim Importieren: ' + err.message);
  }
};
</script>

<template>
  <div class="app-container">
    <nav class="tabs">
      <button 
        :class="{ active: activeTab === 'planner' }" 
        @click="activeTab = 'planner'"
      >
        Planer
      </button>
      <button 
        :class="{ active: activeTab === 'config' }" 
        @click="activeTab = 'config'"
      >
        Konfiguration
      </button>
    </nav>

    <link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=Roboto:wght@100;300;400;500;700;900&family=Open+Sans:wght@300;400;500;600;700;800&family=Lato:wght@100;300;400;700;900&family=Montserrat:wght@100;200;300;400;500;600;700;800;900&family=Source+Sans+Pro:wght@200;300;400;600;700;900&family=Raleway:wght@100;200;300;400;500;600;700;800;900&family=Oswald:wght@200;300;400;500;600;700&family=Merriweather:wght@300;400;700;900&family=Playfair+Display:wght@400;500;600;700;800;900&family=Noto+Sans:wght@100;200;300;400;500;600;700;800;900&display=swap">

    <main class="tab-content">
      <PlannerTab 
        v-if="activeTab === 'planner'"
        :template-image="templateImage"
        :boxes="boxes"
        :global-font="globalFont"
        :global-font-weight="globalFontWeight"
        :global-css="globalCss"
      />
      <ConfigTab 
        v-if="activeTab === 'config'"
        :template-image="templateImage"
        :boxes="boxes"
        :global-font="globalFont"
        :global-font-weight="globalFontWeight"
        :global-css="globalCss"
        @upload="handleImageUpload"
        @update-boxes="updateBoxes"
        @update-config="updateConfig"
        @export-settings="exportToClipboard"
        @import-settings="importFromClipboard"
      />
    </main>
  </div>
</template>

<style>
:root {
  --primary-color: #42b983;
  --bg-color: #f8f9fa;
  --text-color: #2c3e50;
}

body {
  margin: 0;
  font-family: Inter, system-ui, Avenir, Helvetica, Arial, sans-serif;
  background-color: var(--bg-color);
  color: var(--text-color);
}

.app-container {
  width: 70vw;
  margin: 0 auto;
  padding: 20px;
  min-height: 100vh;
  box-sizing: border-box;
  display: flex;
  flex-direction: column;
}

.tabs {
  display: flex;
  gap: 10px;
  margin-bottom: 20px;
  border-bottom: 2px solid #ddd;
  padding-bottom: 10px;
}

.tabs button {
  padding: 10px 20px;
  border: none;
  background: none;
  cursor: pointer;
  font-size: 1.1rem;
  font-weight: bold;
  color: #666;
  border-radius: 4px;
}

.tabs button.active {
  color: var(--primary-color);
  background-color: rgba(66, 185, 131, 0.1);
}

.tab-content {
  background: white;
  padding: 20px;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
  flex-grow: 1;
  display: flex;
  flex-direction: column;
}
</style>
