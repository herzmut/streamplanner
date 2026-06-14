<script setup>
import { ref, watch, onMounted, computed } from 'vue';
import AJV from 'ajv';
import { marked } from 'marked';
import privacyRaw from './assets/privacy.md?raw';
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

const showPrivacyModal = ref(false);
const privacyMarkdown = ref(privacyRaw);

const renderedPrivacy = computed(() => marked(privacyMarkdown.value));

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
      <div class="main-tabs">
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
      </div>
      <div class="meta-links">
        <button class="privacy-link" @click="showPrivacyModal = true">
          Datenschutz
        </button>
      </div>
    </nav>

    <!-- Modal für Datenschutz -->
    <div v-if="showPrivacyModal" class="modal-overlay" @click.self="showPrivacyModal = false">
      <div class="modal-content">
        <div class="modal-header">
          <h2>Datenschutz</h2>
          <button class="close-btn" @click="showPrivacyModal = false">&times;</button>
        </div>
        <div class="modal-body markdown-body" v-html="renderedPrivacy"></div>
        <div class="modal-footer">
          <button class="action-btn" @click="showPrivacyModal = false">Schließen</button>
        </div>
      </div>
    </div>

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
.app-container {
  width: 90vw;
  margin: 0;
  padding: 20px;
  min-height: 100vh;
  box-sizing: border-box;
  display: flex;
  flex-direction: column;
}

.tabs {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
  border-bottom: 2px solid var(--color-border);
  padding-bottom: 10px;
}

.main-tabs {
  display: flex;
  gap: 10px;
}

.privacy-link {
  font-size: 0.9rem !important;
  opacity: 0.5 !important;
  font-weight: normal !important;
}

.privacy-link:hover {
  opacity: 1 !important;
  color: var(--primary-color) !important;
}

/* Modal Styles */
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background: rgba(0, 0, 0, 0.7);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
  backdrop-filter: blur(4px);
}

.modal-content {
  background: var(--color-background-soft);
  width: 800px;
  max-width: 90vw;
  max-height: 80vh;
  border-radius: 8px;
  display: flex;
  flex-direction: column;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.5);
  border: 1px solid var(--color-border);
}

.modal-header {
  padding: 15px 20px;
  border-bottom: 1px solid var(--color-border);
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.modal-header h2 {
  margin: 0;
  font-size: 1.5rem;
  color: var(--primary-color);
}

.close-btn {
  background: none;
  border: none;
  color: var(--color-text);
  font-size: 2rem;
  cursor: pointer;
  line-height: 1;
  padding: 0;
}

.modal-body {
  padding: 20px;
  overflow-y: auto;
  flex-grow: 1;
  color: var(--color-text);
  line-height: 1.6;
}

.modal-footer {
  padding: 15px 20px;
  border-top: 1px solid var(--color-border);
  display: flex;
  justify-content: flex-end;
}

.action-btn {
  padding: 8px 20px;
  background-color: var(--night-owl-button-bg);
  color: var(--night-owl-text);
  border: 1px solid var(--color-border);
  border-radius: 4px;
  cursor: pointer;
  font-weight: bold;
  transition: all 0.2s;
}

.action-btn:hover {
  background-color: var(--night-owl-button-hover);
  border-color: var(--primary-color);
}

/* Markdown Styling */
.markdown-body h1, .markdown-body h2, .markdown-body h3 {
  color: var(--color-heading);
  margin-top: 1.5em;
  margin-bottom: 0.5em;
}

.markdown-body p {
  margin-bottom: 1em;
}

.markdown-body ul {
  margin-bottom: 1em;
  padding-left: 1.5em;
}

.tabs button {
  padding: 10px 20px;
  border: none;
  background: none;
  cursor: pointer;
  font-size: 1.1rem;
  font-weight: bold;
  color: var(--color-text);
  border-radius: 4px;
  opacity: 0.7;
  transition: all 0.2s;
}

.tabs button:hover {
  opacity: 1;
  background-color: var(--color-background-soft);
}

.tabs button.active {
  color: var(--primary-color);
  background-color: rgba(127, 219, 202, 0.1);
  opacity: 1;
}

.tab-content {
  background: var(--color-background-soft);
  padding: 20px;
  border-radius: 8px;
  box-shadow: 0 4px 12px rgba(0,0,0,0.3);
  flex-grow: 1;
  display: flex;
  flex-direction: column;
  color: var(--color-text);
}
</style>
