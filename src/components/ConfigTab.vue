<script setup>
import { ref, watch, computed } from 'vue';

const props = defineProps(['templateImage', 'boxes', 'globalFont', 'globalFontWeight', 'globalCss']);
const emit = defineEmits(['upload', 'update-boxes', 'update-config', 'export-settings', 'import-settings']);

const googleFonts = [
  { name: 'Roboto', weights: [100, 300, 400, 500, 700, 900] },
  { name: 'Open Sans', weights: [300, 400, 500, 600, 700, 800] },
  { name: 'Lato', weights: [100, 300, 400, 700, 900] },
  { name: 'Montserrat', weights: [100, 200, 300, 400, 500, 600, 700, 800, 900] },
  { name: 'Source Sans Pro', weights: [200, 300, 400, 600, 700, 900] },
  { name: 'Raleway', weights: [100, 200, 300, 400, 500, 600, 700, 800, 900] },
  { name: 'Oswald', weights: [200, 300, 400, 500, 600, 700] },
  { name: 'Merriweather', weights: [300, 400, 700, 900] },
  { name: 'Playfair Display', weights: [400, 500, 600, 700, 800, 900] },
  { name: 'Noto Sans', weights: [100, 200, 300, 400, 500, 600, 700, 800, 900] }
];

const fontInput = ref(props.globalFont);
const fontWeightInput = ref(props.globalFontWeight || 400);

const selectedFont = computed(() => googleFonts.find(f => f.name === fontInput.value) || googleFonts[0]);

const previewStyle = computed(() => {
  let style = {
    fontFamily: `'${fontInput.value}'`,
    fontWeight: fontWeightInput.value
  };
  
  // Parse global CSS into the style object
  if (props.globalCss) {
    const rules = props.globalCss.split(';');
    rules.forEach(rule => {
      const [prop, value] = rule.split(':').map(s => s.trim());
      if (prop && value) {
        // Simple mapping for common CSS properties to JS style keys
        const camelProp = prop.replace(/-([a-z])/g, (g) => g[1].toUpperCase());
        style[camelProp] = value;
      }
    });
  }
  
  return style;
});

watch(fontInput, (newFont) => {
  const font = googleFonts.find(f => f.name === newFont);
  if (font && !font.weights.includes(fontWeightInput.value)) {
    // Falls das aktuelle Gewicht nicht für die neue Schriftart verfügbar ist,
    // nimm das am nächsten liegende Gewicht (meist 400).
    fontWeightInput.value = font.weights.includes(400) ? 400 : font.weights[0];
  }
});

const cssInput = ref(props.globalCss);

watch(() => props.globalCss, (newCss) => {
  cssInput.value = newCss;
});

watch(() => props.globalFont, (newFont) => {
  fontInput.value = newFont;
});

watch(() => props.globalFontWeight, (newWeight) => {
  fontWeightInput.value = newWeight;
});

const localBoxes = ref([]);
const editLabelIndex = ref(null);
const editLabelValue = ref('');

watch(() => props.boxes, (newBoxes) => {
  localBoxes.value = [...newBoxes];
}, { immediate: true, deep: true });

const selectedBoxIndex = ref(null);
const canvasContainer = ref(null);

// Drawing state
const isDrawing = ref(false);
const startX = ref(0);
const startY = ref(0);
const currentX = ref(0);
const currentY = ref(0);

// Drag/Resize state
const isDragging = ref(false);
const isResizing = ref(false);
const dragTarget = ref(null);
const dragOffset = { x: 0, y: 0 };

const handleFileUpload = (event) => {
  const file = event.target.files[0];
  if (file) {
    const reader = new FileReader();
    reader.onload = (e) => {
      emit('upload', e.target.result);
    };
    reader.readAsDataURL(file);
  }
};

const onMouseDown = (e) => {
  if (!props.templateImage) return;
  
  const rect = canvasContainer.value.getBoundingClientRect();
  const x = e.clientX - rect.left;
  const y = e.clientY - rect.top;

  // Check if clicked on a box or resize handle
  for (let i = localBoxes.value.length - 1; i >= 0; i--) {
    const box = localBoxes.value[i];
    
    // Resize handle (bottom-right corner)
    if (x > box.x + box.width - 15 && x < box.x + box.width + 5 &&
        y > box.y + box.height - 15 && y < box.y + box.height + 5) {
      isResizing.value = true;
      selectedBoxIndex.value = i;
      dragTarget.value = i;
      return;
    }
    
    // Drag handle (edges)
    const margin = 15;
    const nearLeft = Math.abs(x - box.x) < margin;
    const nearRight = Math.abs(x - (box.x + box.width)) < margin;
    const nearTop = Math.abs(y - box.y) < margin;
    const nearBottom = Math.abs(y - (box.y + box.height)) < margin;

    // Check if clicked on or very near any of the four edges
    if ((nearLeft || nearRight || nearTop || nearBottom) && 
        x >= box.x - margin && x <= box.x + box.width + margin && 
        y >= box.y - margin && y <= box.y + box.height + margin) {
      isDragging.value = true;
      selectedBoxIndex.value = i;
      dragTarget.value = i;
      dragOffset.x = x - box.x;
      dragOffset.y = y - box.y;
      return;
    }

    // Select box if clicked inside
    if (x >= box.x && x <= box.x + box.width && y >= box.y && y <= box.y + box.height) {
      selectedBoxIndex.value = i;
      return;
    }
  }

  // If nothing clicked, start drawing new box
  isDrawing.value = true;
  startX.value = x;
  startY.value = y;
  currentX.value = x;
  currentY.value = y;
  selectedBoxIndex.value = null;
};

const onMouseMove = (e) => {
  if (!props.templateImage) return;
  const rect = canvasContainer.value.getBoundingClientRect();
  const x = e.clientX - rect.left;
  const y = e.clientY - rect.top;

  if (isDrawing.value) {
    currentX.value = x;
    currentY.value = y;
  } else if (isDragging.value && dragTarget.value !== null) {
    const box = localBoxes.value[dragTarget.value];
    box.x = x - dragOffset.x;
    box.y = y - dragOffset.y;
  } else if (isResizing.value && dragTarget.value !== null) {
    const box = localBoxes.value[dragTarget.value];
    box.width = Math.max(20, x - box.x);
    box.height = Math.max(20, y - box.y);
  }
};

const onMouseUp = () => {
  if (isDrawing.value) {
    const width = Math.abs(currentX.value - startX.value);
    const height = Math.abs(currentY.value - startY.value);
    if (width > 10 && height > 10) {
      localBoxes.value.push({
        x: Math.min(startX.value, currentX.value),
        y: Math.min(startY.value, currentY.value),
        width,
        height,
        text: ''
      });
      emit('update-boxes', [...localBoxes.value]);
    }
  } else if (isDragging.value || isResizing.value) {
    emit('update-boxes', [...localBoxes.value]);
  }
  isDrawing.value = false;
  isDragging.value = false;
  isResizing.value = false;
  dragTarget.value = null;
};

const removeBox = (index) => {
  localBoxes.value.splice(index, 1);
  selectedBoxIndex.value = null;
  emit('update-boxes', [...localBoxes.value]);
};

const originalLabelValue = ref('');

const startEditLabel = (index) => {
  editLabelIndex.value = index;
  const currentLabel = localBoxes.value[index].label || `Box ${index + 1}`;
  editLabelValue.value = currentLabel;
  originalLabelValue.value = currentLabel;
};

const updateLabelLive = (index) => {
  localBoxes.value[index].label = editLabelValue.value;
  emit('update-boxes', [...localBoxes.value]);
};

const handleBlur = (index) => {
  if (!editLabelValue.value.trim()) {
    localBoxes.value[index].label = ''; // Set to empty so getter/computed can show "Box N"
    // Or explicitly set "Box N" if preferred. The description says "wieder eingesetzt werden".
    // Actually, localBoxes[index].label = '' will trigger the fallback in template.
    // But let's be explicit if the user wants it to be "Box N" again.
    localBoxes.value[index].label = `Box ${index + 1}`;
    emit('update-boxes', [...localBoxes.value]);
  }
  editLabelIndex.value = null;
};

const cancelEditLabel = (index) => {
  localBoxes.value[index].label = originalLabelValue.value;
  editLabelValue.value = originalLabelValue.value;
  editLabelIndex.value = null;
  emit('update-boxes', [...localBoxes.value]);
};

watch([fontInput, fontWeightInput, cssInput], () => {
  emit('update-config', { font: fontInput.value, fontWeight: fontWeightInput.value, css: cssInput.value });
});

const vFocus = {
  mounted: (el) => el.focus()
};
</script>

<template>
  <div class="config-tab">
    <div class="sidebar">
      <section>
        <h3>Allgemein</h3>
        <div class="input-group">
          <label>Vorlage hochladen:</label>
          <input type="file" @change="handleFileUpload" accept="image/*" />
        </div>
        <div class="transfer-actions">
          <button @click="emit('export-settings')" class="action-btn">Export</button>
          <button @click="emit('import-settings')" class="action-btn">Import</button>
        </div>
      </section>

      <section>
        <h3>Schriftart</h3>
        <div class="selected-font-preview" :style="previewStyle">
          AaBbQqZz
        </div>
        
        <div class="font-selector">
          <div 
            v-for="font in googleFonts" 
            :key="font.name"
            class="font-option"
            :class="{ active: fontInput === font.name }"
            @click="fontInput = font.name"
            :style="{ fontFamily: `'${font.name}'` }"
          >
            <div class="font-name">{{ font.name }}</div>
            <div class="font-preview">AaBbQqZz</div>
          </div>
        </div>

        <div class="weight-selector" v-if="selectedFont">
          <label>Schriftdicke: {{ fontWeightInput }}</label>
          <input 
            type="range" 
            v-model.number="fontWeightInput" 
            :min="selectedFont.weights[0]" 
            :max="selectedFont.weights[selectedFont.weights.length - 1]" 
            step="100"
          />
        </div>
      </section>

      <section>
        <h3>CSS Regeln</h3>
        <textarea 
          v-model="cssInput" 
          placeholder="z.B. color: white; font-weight: bold; text-align: center;"
          rows="10"
        ></textarea>
        <p class="hint">Diese Regeln gelten für alle Textboxen.</p>
      </section>
      
      <section v-if="localBoxes.length > 0">
        <h3>Boxen ({{ localBoxes.length }})</h3>
        <ul class="box-list">
          <li v-for="(box, index) in localBoxes" :key="index" class="box-item">
            <div v-if="editLabelIndex === index" class="edit-label-container">
              <input 
                v-model="editLabelValue" 
                @input="updateLabelLive(index)"
                @keyup.enter="editLabelIndex = null" 
                @keyup.esc="cancelEditLabel(index)"
                @blur="handleBlur(index)"
                v-focus
                class="edit-label-input"
              />
            </div>
            <div v-else class="box-label-display" @click="startEditLabel(index)">
              <span class="box-name">{{ box.label || `Box ${index + 1}` }}</span>
              <button @click.stop="removeBox(index)" class="btn-small">Löschen</button>
            </div>
          </li>
        </ul>
      </section>
    </div>

    <div class="main-preview" :class="{ 'centered-content': templateImage }">
      <div 
        ref="canvasContainer" 
        class="canvas-container"
        :style="{ cursor: isDrawing ? 'crosshair' : 'default' }"
        @mousedown="onMouseDown"
        @mousemove="onMouseMove"
        @mouseup="onMouseUp"
      >
        <img v-if="templateImage" :src="templateImage" class="template-img" draggable="false" alt="Vorlage" />
        <div v-else class="empty-state">
          Bitte eine Vorlage hochladen, um Boxen zu definieren.
        </div>

        <!-- Drawing Box -->
        <div 
          v-if="isDrawing" 
          class="drawing-box"
          :style="{
            left: Math.min(startX, currentX) + 'px',
            top: Math.min(startY, currentY) + 'px',
            width: Math.abs(currentX - startX) + 'px',
            height: Math.abs(currentY - startY) + 'px'
          }"
        ></div>

        <!-- Existing Boxes -->
        <div 
          v-for="(box, index) in localBoxes" 
          :key="index"
          class="box-overlay"
          :class="{ selected: selectedBoxIndex === index }"
          :style="[
            {
              left: box.x + 'px',
              top: box.y + 'px',
              width: box.width + 'px',
              height: box.height + 'px'
            },
            previewStyle
          ]"
        >
          <span class="box-label">{{ box.label || (index + 1) }}</span>
          
          <div v-if="box.text" class="box-content-preview">
            <div v-for="(line, i) in box.text.split('\n')" :key="i">
              {{ line }}
            </div>
          </div>
          
          <!-- Delete Icon (bottom left) -->
          <button 
            v-if="selectedBoxIndex === index" 
            class="delete-btn" 
            @click.stop="removeBox(index)"
            title="Löschen"
          >
            🗑️
          </button>
          
          <!-- Resize Handle (bottom right) -->
          <div class="resize-handle"></div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.config-tab {
  display: flex;
  gap: 20px;
  flex-grow: 1;
}

.sidebar {
  width: 300px;
  flex-shrink: 0;
}

section {
  margin-bottom: 20px;
  padding-bottom: 20px;
  border-bottom: 1px solid #eee;
}

h3 {
  margin-top: 0;
  margin-bottom: 10px;
  font-size: 1rem;
}

.box-list {
  list-style: none;
  padding: 0;
}

.box-item {
  margin-bottom: 8px;
  padding: 4px;
  border-radius: 4px;
  background: #f9f9f9;
}

.box-label-display {
  display: flex;
  justify-content: space-between;
  align-items: center;
  cursor: pointer;
}

.box-name {
  flex-grow: 1;
  font-size: 0.9rem;
}

.edit-label-container {
  display: flex;
  gap: 5px;
}

.edit-label-input {
  flex-grow: 1;
  padding: 2px 4px;
  font-size: 0.9rem;
}

label {
  display: block;
  margin-bottom: 5px;
}

.input-group {
  margin-bottom: 10px;
}

.transfer-actions {
  display: flex;
  gap: 10px;
  margin-top: 10px;
}

.action-btn {
  flex: 1;
  padding: 8px;
  background-color: var(--primary-color, #42b983);
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-weight: bold;
}

.action-btn:hover {
  filter: brightness(0.9);
}

input[type="file"], select, textarea {
  width: 100%;
  padding: 8px;
  border: 1px solid #ccc;
  border-radius: 4px;
  box-sizing: border-box;
}

textarea {
  font-family: monospace;
  resize: vertical;
}

.hint {
  font-size: 0.8rem;
  color: #666;
  margin-top: 5px;
}

.main-preview {
  flex-grow: 1;
  overflow: auto;
  background: #eee;
  padding: 20px;
  border-radius: 4px;
  min-height: 600px;
  display: flex;
  flex-direction: column;
}

.main-preview.centered-content {
  align-items: center;
}

.canvas-container {
  position: relative;
  display: inline-block;
  box-shadow: 0 0 10px rgba(0,0,0,0.2);
  background: white;
}

.template-img {
  display: block;
  max-width: none;
  user-select: none;
}

.empty-state {
  flex-grow: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #999;
  border: 2px dashed #ccc;
  background: white;
}

.drawing-box, .box-overlay {
  position: absolute;
  box-sizing: border-box;
  background-image: linear-gradient(to right, #000 50%, transparent 50%), 
                    linear-gradient(to right, #000 50%, transparent 50%), 
                    linear-gradient(to bottom, #000 50%, transparent 50%), 
                    linear-gradient(to bottom, #000 50%, transparent 50%);
  background-position: left top, left bottom, left top, right top;
  background-repeat: repeat-x, repeat-x, repeat-y, repeat-y;
  background-size: 10px 1px, 10px 1px, 1px 10px, 1px 10px;
  animation: marquee 0.4s infinite linear;
}

@keyframes marquee {
  0% { background-position: 0 0, 10px 100%, 0 10px, 100% 0; }
  100% { background-position: 10px 0, 0 100%, 0 0, 100% 10px; }
}

.box-overlay {
  background: rgba(66, 185, 131, 0.1);
}

.box-overlay.selected {
  background: rgba(66, 185, 131, 0.3);
  border-color: #42b983;
  z-index: 10;
}

.box-label {
  position: absolute;
  top: -20px;
  left: 0;
  background: #42b983;
  color: white;
  padding: 0 5px;
  font-size: 0.8rem;
  border-radius: 2px;
  z-index: 2;
}

.box-content-preview {
  width: 100%;
  height: 100%;
  overflow: hidden;
  white-space: pre-wrap;
  word-break: break-word;
  pointer-events: none;
  font-size: inherit; /* Nutze die Schriftgröße von .box-overlay (previewStyle) */
}

.delete-btn {
  position: absolute;
  bottom: -15px;
  left: -15px;
  background: white;
  border: 1px solid #ddd;
  border-radius: 50%;
  width: 30px;
  height: 30px;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: 0 2px 4px rgba(0,0,0,0.2);
}

.delete-btn:hover {
  background: #fde8e8;
}

.resize-handle {
  position: absolute;
  bottom: -5px;
  right: -5px;
  width: 10px;
  height: 10px;
  background: #42b983;
  cursor: nwse-resize;
  border-radius: 50%;
}

.selected-font-preview {
  padding: 15px;
  border: 1px solid #eee;
  border-radius: 4px;
  margin-bottom: 15px;
  text-align: center;
  font-size: 2rem;
  background: #f9f9f9;
}

.font-selector {
  display: flex;
  flex-direction: column;
  gap: 10px;
  max-height: 300px;
  overflow-y: auto;
  border: 1px solid #ddd;
  padding: 10px;
  border-radius: 4px;
  margin-bottom: 15px;
}

.font-option {
  padding: 8px;
  border: 1px solid transparent;
  border-radius: 4px;
  cursor: pointer;
  transition: all 0.2s;
}

.font-option:hover {
  background: #f0f0f0;
}

.font-option.active {
  border-color: #42b983;
  background: rgba(66, 185, 131, 0.1);
}

.font-name {
  font-size: 0.8rem;
  color: #666;
  margin-bottom: 2px;
}

.font-preview {
  font-size: 1.2rem;
}

.weight-selector {
  margin-top: 10px;
}

.weight-selector input {
  width: 100%;
}

.btn-small {
  padding: 2px 5px;
  font-size: 0.7rem;
  margin-left: 5px;
}

ul {
  padding-left: 20px;
}

li {
  margin-bottom: 5px;
}
</style>
