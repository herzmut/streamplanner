<script setup>
import { ref, computed, watch } from 'vue';

const props = defineProps(['templateImage', 'boxes', 'globalFont', 'globalFontWeight', 'globalColor', 'globalCss']);

const localTexts = ref([]);

watch(() => props.boxes, (newBoxes) => {
  localTexts.value = newBoxes.map(b => b.text || '');
}, { immediate: true, deep: true });

const updateText = (index, value) => {
  localTexts.value[index] = value;
  props.boxes[index].text = value; // Update the shared reference
};

const combinedStyle = computed(() => {
  let style = {
    fontFamily: `'${props.globalFont}'`,
    fontWeight: props.globalFontWeight,
    color: props.globalColor
  };

  if (props.globalCss) {
    const rules = props.globalCss.split(';');
    rules.forEach(rule => {
      const [prop, value] = rule.split(':').map(s => s.trim());
      if (prop && value) {
        const camelProp = prop.replace(/-([a-z])/g, (g) => g[1].toUpperCase());
        style[camelProp] = value;
      }
    });
  }

  return style;
});

const downloadImage = (format) => {
  const canvas = document.createElement('canvas');
  const img = document.querySelector('.template-preview-img');
  
  if (!img) return;

  canvas.width = img.naturalWidth;
  canvas.height = img.naturalHeight;
  const ctx = canvas.getContext('2d');

  // Draw background image
  ctx.drawImage(img, 0, 0);

  // Get scaling factor (if displayed size differs from natural size)
  const scaleX = img.naturalWidth / img.clientWidth;
  const scaleY = img.naturalHeight / img.clientHeight;

  // Draw text boxes
  props.boxes.forEach((box, index) => {
    const text = localTexts.value[index] || '';
    const lines = text.split('\n');
    
    ctx.save();
    
    // Parse font size from globalCss with support for various units
    let fontSizeInPx = 20;
    const sizeMatch = props.globalCss.match(/font-size:\s*([\d.]+)([a-zA-Z%]*)/);
    if (sizeMatch) {
      const value = parseFloat(sizeMatch[1]);
      const unit = sizeMatch[2] || 'px';

      // Convert units to pixels (rough estimation for some units)
      if (unit === 'px') {
        fontSizeInPx = value;
      } else if (unit === 'em' || unit === 'rem') {
        fontSizeInPx = value * 16; // Assuming base font size is 16px
      } else if (unit === 'pt') {
        fontSizeInPx = value * 1.333;
      } else if (unit === '%') {
        fontSizeInPx = (value / 100) * 16;
      } else {
        fontSizeInPx = value; // Default to value if unit is unknown
      }
    }
    
    // Scaling font size for natural resolution
    const scaledFontSize = fontSizeInPx * scaleY;
    
    // Set font weight from globalFontWeight if not specified in globalCss
    let fontWeight = props.globalFontWeight || 400;
    const weightMatch = props.globalCss.match(/font-weight:\s*([^;]+)/);
    if (weightMatch) fontWeight = weightMatch[1].trim();
    
    ctx.font = `${fontWeight} ${scaledFontSize}px '${props.globalFont}'`;
    
    // Setze die Schriftfarbe: Priorität hat CSS, dann globalColor, dann Schwarz
    let color = props.globalColor || '#000000';
    const colorMatch = props.globalCss.match(/color:\s*([^;]+)/);
    if (colorMatch) color = colorMatch[1].trim();
    ctx.fillStyle = color;

    // Parse text-align
    let textAlign = 'left';
    const alignMatch = props.globalCss.match(/text-align:\s*([^;]+)/);
    if (alignMatch) textAlign = alignMatch[1].trim();
    ctx.textAlign = textAlign === 'center' ? 'center' : (textAlign === 'right' ? 'right' : 'left');

    const boxX = box.x * scaleX;
    const boxY = box.y * scaleY;
    const boxW = box.width * scaleX;

    let startX = boxX;
    if (ctx.textAlign === 'center') startX = boxX + boxW / 2;
    if (ctx.textAlign === 'right') startX = boxX + boxW;

    const lineHeight = scaledFontSize * 1.68;
    lines.forEach((line, i) => {
      ctx.fillText(line, startX, boxY + scaledFontSize + (i * lineHeight), boxW);
    });

    ctx.restore();
  });

  const link = document.createElement('a');
  link.download = `wochenplan.${format}`;
  link.href = canvas.toDataURL(`image/${format === 'jpg' ? 'jpeg' : 'png'}`);
  link.click();
};

</script>

<template>
  <div class="planner-tab">
    <div class="sidebar">
      <h3>Programm eintragen</h3>
      <div v-if="boxes.length === 0" class="no-boxes">
        Keine Boxen definiert. Gehe zum Reiter "Konfiguration".
      </div>
      <div v-for="(box, index) in boxes" :key="index" class="text-input-group">
        <label>{{ box.label || `Box ${index + 1}` }}</label>
        <textarea 
          :value="localTexts[index]" 
          @input="updateText(index, $event.target.value)"
          rows="3"
        ></textarea>
      </div>

      <div class="download-actions" v-if="templateImage && boxes.length > 0">
        <button @click="downloadImage('png')">Als PNG herunterladen</button>
        <button @click="downloadImage('jpg')">Als JPG herunterladen</button>
      </div>
    </div>

    <div class="main-preview">
      <div v-if="templateImage" class="preview-container">
        <img :src="templateImage" class="template-preview-img" alt="Vorschau" />
        
        <div 
          v-for="(box, index) in boxes" 
          :key="index"
          class="text-overlay-box"
          :style="[
            {
              left: box.x + 'px',
              top: box.y + 'px',
              width: box.width + 'px',
              height: box.height + 'px'
            },
            combinedStyle
          ]"
        >
          <div class="content-display" :style="combinedStyle">
            <div v-for="(line, i) in (localTexts[index] || '').split('\n')" :key="i">
              {{ line }}
            </div>
          </div>
        </div>
      </div>
      <div v-else class="empty-state">
        Keine Vorlage vorhanden. Bitte im Reiter "Konfiguration" hochladen.
      </div>
    </div>
  </div>
</template>

<style scoped>
.planner-tab {
  display: flex;
  gap: 20px;
  flex-grow: 1;
}

.sidebar {
  width: 300px;
  flex-shrink: 0;
}

.text-input-group {
  margin-bottom: 15px;
}

.text-input-group label {
  display: block;
  font-weight: bold;
  margin-bottom: 5px;
  font-size: 0.9rem;
  color: var(--color-heading);
}

.text-input-group textarea {
  width: 100%;
  padding: 8px;
  background-color: var(--color-background-mute);
  color: var(--color-text);
  border: 1px solid var(--color-border);
  border-radius: 4px;
  box-sizing: border-box;
  transition: border-color 0.2s;
}

.text-input-group textarea:focus {
  border-color: var(--primary-color);
  outline: none;
}

.download-actions {
  margin-top: 20px;
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.download-actions button {
  padding: 10px;
  background-color: var(--night-owl-button-bg);
  color: var(--night-owl-text);
  border: 1px solid var(--color-border);
  border-radius: 4px;
  cursor: pointer;
  font-weight: bold;
  transition: all 0.2s;
}

.download-actions button:hover {
  background-color: var(--night-owl-button-hover);
  border-color: var(--primary-color);
}

.main-preview {
  flex-grow: 1;
  background: var(--color-background);
  padding: 20px;
  border-radius: 4px;
  overflow: auto;
  display: flex;
  flex-direction: column;
}


.preview-container {
  position: relative;
  display: inline-block;
  background: #222;
  box-shadow: 0 4px 20px rgba(0,0,0,0.5);
}

.template-preview-img {
  display: block;
  max-width: none;
}

.text-overlay-box {
  position: absolute;
  pointer-events: none;
  overflow: hidden;
  white-space: pre-wrap;
  word-break: break-word;
}

.content-display {
  width: 100%;
  height: 100%;
  font-size: 20px; /* Match default from downloadImage for consistent preview */
}

.no-boxes, .empty-state {
  color: var(--color-text);
  opacity: 0.6;
  font-style: italic;
  padding: 20px;
  text-align: center;
}

.empty-state {
  background: var(--color-background-mute);
  flex-grow: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  border: 2px dashed var(--color-border);
}
</style>
