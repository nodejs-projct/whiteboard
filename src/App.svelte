<script>
  import { onMount } from 'svelte';
  
  let canvas;
  let ctx;
  let isDrawing = false;
  let tool = 'pen';
  let color = '#000000';
  let lineWidth = 2;
  let history = [];
  let historyStep = -1;
  let startPos = { x: 0, y: 0 };
  let currentPath = [];
  let shapes = [];
  let texts = [];
  let editingText = null;
  let selectedShape = null;
  let isPanning = false;
  let panStart = { x: 0, y: 0 };
  let offset = { x: 0, y: 0 };
  let scale = 1;
  let stickyNotes = [];
  let arrows = [];
  let fillShape = false;
  let uploadedImages = [];
  let isDragging = false;
  let draggedItem = null;

  const tools = [
    { id: 'select', label: 'Select', icon: '🖱️' },
    { id: 'pen', label: 'Pen', icon: '✏️' },
    { id: 'eraser', label: 'Eraser', icon: '🧹' },
    { id: 'line', label: 'Line', icon: '📏' },
    { id: 'rectangle', label: 'Rectangle', icon: '⬜' },
    { id: 'circle', label: 'Circle', icon: '⭕' },
    { id: 'arrow', label: 'Arrow', icon: '➡️' },
    { id: 'text', label: 'Text', icon: '📝' },
    { id: 'sticky', label: 'Sticky Note', icon: '📌' },
    { id: 'pan', label: 'Pan', icon: '✋' }
  ];

  const colorPresets = [
    '#000000', '#FF0000', '#00FF00', '#0000FF', 
    '#FFFF00', '#FF00FF', '#00FFFF', '#FFA500',
    '#800080', '#008000', '#808080', '#FFFFFF'
  ];

  onMount(() => {
    if (canvas) {
      canvas.width = canvas.offsetWidth;
      canvas.height = canvas.offsetHeight;
      ctx = canvas.getContext('2d');
      saveState();
      redraw();
    }
    
    window.addEventListener('resize', handleResize);
    return () => window.removeEventListener('resize', handleResize);
  });

  function handleResize() {
    if (canvas) {
      const tempCanvas = document.createElement('canvas');
      tempCanvas.width = canvas.width;
      tempCanvas.height = canvas.height;
      const tempCtx = tempCanvas.getContext('2d');
      tempCtx.drawImage(canvas, 0, 0);
      
      canvas.width = canvas.offsetWidth;
      canvas.height = canvas.offsetHeight;
      ctx = canvas.getContext('2d');
      ctx.drawImage(tempCanvas, 0, 0);
    }
  }

  function getMousePos(e) {
    const rect = canvas.getBoundingClientRect();
    return {
      x: (e.clientX - rect.left - offset.x) / scale,
      y: (e.clientY - rect.top - offset.y) / scale
    };
  }

  function saveState() {
    const state = {
      shapes: JSON.parse(JSON.stringify(shapes)),
      texts: JSON.parse(JSON.stringify(texts)),
      stickyNotes: JSON.parse(JSON.stringify(stickyNotes)),
      arrows: JSON.parse(JSON.stringify(arrows)),
      uploadedImages: JSON.parse(JSON.stringify(uploadedImages))
    };
    
    const newHistory = history.slice(0, historyStep + 1);
    newHistory.push(state);
    history = newHistory;
    historyStep = newHistory.length - 1;
  }

  function restoreState(step) {
    if (history[step]) {
      const state = history[step];
      shapes = JSON.parse(JSON.stringify(state.shapes));
      texts = JSON.parse(JSON.stringify(state.texts));
      stickyNotes = JSON.parse(JSON.stringify(state.stickyNotes));
      arrows = JSON.parse(JSON.stringify(state.arrows));
      uploadedImages = JSON.parse(JSON.stringify(state.uploadedImages));
      redraw();
    }
  }

  function redraw() {
    if (!ctx) return;
    
    ctx.save();
    ctx.clearRect(0, 0, canvas.width, canvas.height);
    ctx.translate(offset.x, offset.y);
    ctx.scale(scale, scale);

    // Draw grid
    drawGrid();

    // Draw all shapes
    shapes.forEach(shape => drawShape(shape));
    
    // Draw arrows
    arrows.forEach(arrow => drawArrow(arrow));
    
    // Draw texts
    texts.forEach(text => drawText(text));
    
    // Draw uploaded images
    uploadedImages.forEach(img => {
      if (img.loaded) {
        ctx.drawImage(img.image, img.x, img.y, img.width, img.height);
      }
    });

    ctx.restore();
  }

  function drawGrid() {
    ctx.strokeStyle = '#e0e0e0';
    ctx.lineWidth = 0.5;
    const gridSize = 20;
    
    for (let x = 0; x < canvas.width / scale; x += gridSize) {
      ctx.beginPath();
      ctx.moveTo(x, 0);
      ctx.lineTo(x, canvas.height / scale);
      ctx.stroke();
    }
    
    for (let y = 0; y < canvas.height / scale; y += gridSize) {
      ctx.beginPath();
      ctx.moveTo(0, y);
      ctx.lineTo(canvas.width / scale, y);
      ctx.stroke();
    }
  }

  function drawShape(shape) {
    ctx.strokeStyle = shape.color;
    ctx.fillStyle = shape.color;
    ctx.lineWidth = shape.lineWidth;
    ctx.lineCap = 'round';
    ctx.lineJoin = 'round';

    if (shape.type === 'pen') {
      ctx.beginPath();
      shape.points.forEach((point, index) => {
        if (index === 0) {
          ctx.moveTo(point.x, point.y);
        } else {
          ctx.lineTo(point.x, point.y);
        }
      });
      ctx.stroke();
    } else if (shape.type === 'rectangle') {
      ctx.beginPath();
      ctx.rect(shape.x, shape.y, shape.width, shape.height);
      if (shape.fill) {
        ctx.fill();
      } else {
        ctx.stroke();
      }
    } else if (shape.type === 'circle') {
      ctx.beginPath();
      ctx.arc(shape.x, shape.y, shape.radius, 0, 2 * Math.PI);
      if (shape.fill) {
        ctx.fill();
      } else {
        ctx.stroke();
      }
    } else if (shape.type === 'line') {
      ctx.beginPath();
      ctx.moveTo(shape.x1, shape.y1);
      ctx.lineTo(shape.x2, shape.y2);
      ctx.stroke();
    }
  }

  function drawArrow(arrow) {
    ctx.strokeStyle = arrow.color;
    ctx.fillStyle = arrow.color;
    ctx.lineWidth = arrow.lineWidth;
    
    const headLength = 15;
    const angle = Math.atan2(arrow.y2 - arrow.y1, arrow.x2 - arrow.x1);
    
    ctx.beginPath();
    ctx.moveTo(arrow.x1, arrow.y1);
    ctx.lineTo(arrow.x2, arrow.y2);
    ctx.stroke();
    
    ctx.beginPath();
    ctx.moveTo(arrow.x2, arrow.y2);
    ctx.lineTo(
      arrow.x2 - headLength * Math.cos(angle - Math.PI / 6),
      arrow.y2 - headLength * Math.sin(angle - Math.PI / 6)
    );
    ctx.lineTo(
      arrow.x2 - headLength * Math.cos(angle + Math.PI / 6),
      arrow.y2 - headLength * Math.sin(angle + Math.PI / 6)
    );
    ctx.closePath();
    ctx.fill();
  }

  function drawText(text) {
    ctx.font = `${text.size}px ${text.font}`;
    ctx.fillStyle = text.color;
    ctx.fillText(text.content, text.x, text.y);
  }

  function startDrawing(e) {
    const pos = getMousePos(e);
    isDrawing = true;
    startPos = pos;
    currentPath = [pos];

    if (tool === 'pan') {
      isPanning = true;
      panStart = { x: e.clientX - offset.x, y: e.clientY - offset.y };
      return;
    }

    if (tool === 'select') {
      // Check if clicking on a sticky note
      const clickedNote = stickyNotes.find(note => 
        pos.x >= note.x && pos.x <= note.x + note.width &&
        pos.y >= note.y && pos.y <= note.y + note.height
      );
      
      if (clickedNote) {
        isDragging = true;
        draggedItem = { type: 'sticky', item: clickedNote };
        startPos = pos;
      }
      return;
    }

    if (tool === 'text') {
      editingText = { x: pos.x, y: pos.y, content: '', size: lineWidth * 10, font: 'Arial', color };
      return;
    }

    if (tool === 'sticky') {
      const newSticky = {
        id: Date.now(),
        x: pos.x,
        y: pos.y,
        width: 150,
        height: 150,
        content: '',
        color: '#FFF59D',
        editing: true
      };
      stickyNotes = [...stickyNotes, newSticky];
      saveState();
      return;
    }
  }

  function draw(e) {
    if (!isDrawing) return;
    
    const pos = getMousePos(e);

    if (tool === 'pan' && isPanning) {
      offset = {
        x: e.clientX - panStart.x,
        y: e.clientY - panStart.y
      };
      redraw();
      return;
    }

    if (tool === 'select' && isDragging && draggedItem) {
      const dx = pos.x - startPos.x;
      const dy = pos.y - startPos.y;
      
      if (draggedItem.type === 'sticky') {
        draggedItem.item.x += dx;
        draggedItem.item.y += dy;
      }
      
      startPos = pos;
      redraw();
      return;
    }

    if (tool === 'pen') {
      currentPath.push(pos);
      const shape = {
        type: 'pen',
        points: [...currentPath],
        color,
        lineWidth
      };
      redraw();
      drawShape(shape);
    } else if (tool === 'eraser') {
      shapes = shapes.filter(shape => {
        if (shape.type === 'pen') {
          return !shape.points.some(point => 
            Math.abs(point.x - pos.x) < lineWidth * 2 && 
            Math.abs(point.y - pos.y) < lineWidth * 2
          );
        }
        return true;
      });
      redraw();
    } else if (['rectangle', 'circle', 'line', 'arrow'].includes(tool)) {
      redraw();
      
      if (tool === 'rectangle') {
        const width = pos.x - startPos.x;
        const height = pos.y - startPos.y;
        drawShape({ 
          type: 'rectangle', 
          x: startPos.x, 
          y: startPos.y, 
          width, 
          height, 
          color, 
          lineWidth,
          fill: fillShape 
        });
      } else if (tool === 'circle') {
        const radius = Math.sqrt(Math.pow(pos.x - startPos.x, 2) + Math.pow(pos.y - startPos.y, 2));
        drawShape({ 
          type: 'circle', 
          x: startPos.x, 
          y: startPos.y, 
          radius, 
          color, 
          lineWidth,
          fill: fillShape 
        });
      } else if (tool === 'line') {
        drawShape({ 
          type: 'line', 
          x1: startPos.x, 
          y1: startPos.y, 
          x2: pos.x, 
          y2: pos.y, 
          color, 
          lineWidth 
        });
      } else if (tool === 'arrow') {
        drawArrow({ 
          x1: startPos.x, 
          y1: startPos.y, 
          x2: pos.x, 
          y2: pos.y, 
          color, 
          lineWidth 
        });
      }
    }
  }

  function stopDrawing(e) {
    if (!isDrawing) return;
    
    const pos = getMousePos(e);

    if (tool === 'pan') {
      isPanning = false;
      isDrawing = false;
      return;
    }

    if (tool === 'select' && isDragging) {
      isDragging = false;
      draggedItem = null;
      isDrawing = false;
      saveState();
      return;
    }

    if (tool === 'pen' && currentPath.length > 0) {
      shapes = [...shapes, {
        type: 'pen',
        points: currentPath,
        color,
        lineWidth
      }];
    } else if (tool === 'rectangle') {
      const width = pos.x - startPos.x;
      const height = pos.y - startPos.y;
      if (Math.abs(width) > 2 && Math.abs(height) > 2) {
        shapes = [...shapes, {
          type: 'rectangle',
          x: startPos.x,
          y: startPos.y,
          width,
          height,
          color,
          lineWidth,
          fill: fillShape
        }];
      }
    } else if (tool === 'circle') {
      const radius = Math.sqrt(Math.pow(pos.x - startPos.x, 2) + Math.pow(pos.y - startPos.y, 2));
      if (radius > 2) {
        shapes = [...shapes, {
          type: 'circle',
          x: startPos.x,
          y: startPos.y,
          radius,
          color,
          lineWidth,
          fill: fillShape
        }];
      }
    } else if (tool === 'line') {
      shapes = [...shapes, {
        type: 'line',
        x1: startPos.x,
        y1: startPos.y,
        x2: pos.x,
        y2: pos.y,
        color,
        lineWidth
      }];
    } else if (tool === 'arrow') {
      arrows = [...arrows, {
        x1: startPos.x,
        y1: startPos.y,
        x2: pos.x,
        y2: pos.y,
        color,
        lineWidth
      }];
    }

    isDrawing = false;
    currentPath = [];
    if (tool !== 'text' && tool !== 'sticky') {
      saveState();
      redraw();
    }
  }

  function handleUndo() {
    if (historyStep > 0) {
      historyStep--;
      restoreState(historyStep);
    }
  }

  function handleRedo() {
    if (historyStep < history.length - 1) {
      historyStep++;
      restoreState(historyStep);
    }
  }

  function handleClear() {
    shapes = [];
    texts = [];
    stickyNotes = [];
    arrows = [];
    uploadedImages = [];
    saveState();
    redraw();
  }

  function handleDownload() {
    const link = document.createElement('a');
    link.download = 'whiteboard.png';
    link.href = canvas.toDataURL();
    link.click();
  }

  function handleZoomIn() {
    scale = Math.min(scale * 1.2, 5);
    redraw();
  }

  function handleZoomOut() {
    scale = Math.max(scale / 1.2, 0.1);
    redraw();
  }

  function handleZoomReset() {
    scale = 1;
    offset = { x: 0, y: 0 };
    redraw();
  }

  function handleImageUpload(e) {
    const file = e.target.files[0];
    if (file && file.type.startsWith('image/')) {
      const reader = new FileReader();
      reader.onload = (event) => {
        const img = new Image();
        img.onload = () => {
          const maxWidth = 300;
          const scale = img.width > maxWidth ? maxWidth / img.width : 1;
          uploadedImages = [...uploadedImages, {
            image: img,
            x: 50,
            y: 50,
            width: img.width * scale,
            height: img.height * scale,
            loaded: true
          }];
          saveState();
          redraw();
        };
        img.src = event.target.result;
      };
      reader.readAsDataURL(file);
    }
  }

  function handleExportJSON() {
    const data = {
      shapes,
      texts,
      stickyNotes,
      arrows,
      version: '1.0'
    };
    const blob = new Blob([JSON.stringify(data, null, 2)], { type: 'application/json' });
    const url = URL.createObjectURL(blob);
    const link = document.createElement('a');
    link.href = url;
    link.download = 'whiteboard.json';
    link.click();
    URL.revokeObjectURL(url);
  }

  function handleImportJSON(e) {
    const file = e.target.files[0];
    if (file) {
      const reader = new FileReader();
      reader.onload = (event) => {
        try {
          const data = JSON.parse(event?.target?.result);
          shapes = data.shapes || [];
          texts = data.texts || [];
          stickyNotes = data.stickyNotes || [];
          arrows = data.arrows || [];
          saveState();
          redraw();
        } catch (err) {
          alert('Invalid file format');
        }
      };
      reader.readAsText(file);
    }
  }

  function updateStickyNote(note, content) {
    note.content = content;
    note.editing = false;
    saveState();
  }

  function deleteStickyNote(noteId) {
    stickyNotes = stickyNotes.filter(n => n.id !== noteId);
    saveState();
  }
</script>

<div class="whiteboard-container">
  <!-- Toolbar -->
  <div class="toolbar">
    <div class="toolbar-section">
      <h1>Whiteboard</h1>
    </div>

    <!-- Tools -->
    <div class="toolbar-section">
      {#each tools as t}
        <button
          class="tool-btn"
          class:active={tool === t.id}
          on:click={() => tool = t.id}
          title={t.label}
        >
          {t.icon}
        </button>
      {/each}
    </div>

    <!-- Color Picker -->
    <div class="toolbar-section">
      <input
        type="color"
        bind:value={color}
        class="color-picker"
        title="Color"
      />
      <div class="color-presets">
        {#each colorPresets as preset}
          <button
            class="color-preset"
            style="background: {preset}"
            on:click={() => color = preset}
          />
        {/each}
      </div>
    </div>

    <!-- Line Width -->
    <div class="toolbar-section">
      <label>
        Width: {lineWidth}px
        <input
          type="range"
          min="1"
          max="20"
          bind:value={lineWidth}
          class="slider"
        />
      </label>
    </div>

    <!-- Fill Toggle -->
    <div class="toolbar-section">
      <label>
        <input type="checkbox" bind:checked={fillShape} />
        Fill Shape
      </label>
    </div>

    <!-- History Controls -->
    <div class="toolbar-section">
      <button
        on:click={handleUndo}
        disabled={historyStep <= 0}
        title="Undo"
      >
        ↶ Undo
      </button>
      <button
        on:click={handleRedo}
        disabled={historyStep >= history.length - 1}
        title="Redo"
      >
        ↷ Redo
      </button>
    </div>

    <!-- Zoom Controls -->
    <div class="toolbar-section">
      <button on:click={handleZoomOut}>🔍-</button>
      <button on:click={handleZoomReset}>{Math.round(scale * 100)}%</button>
      <button on:click={handleZoomIn}>🔍+</button>
    </div>

    <!-- File Operations -->
    <div class="toolbar-section">
      <label class="file-btn">
        📷 Image
        <input type="file" accept="image/*" on:change={handleImageUpload} style="display: none" />
      </label>
      <button on:click={handleDownload}>💾 Export PNG</button>
      <button on:click={handleExportJSON}>📤 Export JSON</button>
      <label class="file-btn">
        📥 Import
        <input type="file" accept=".json" on:change={handleImportJSON} style="display: none" />
      </label>
      <button on:click={handleClear} class="danger">🗑️ Clear</button>
    </div>
  </div>

  <!-- Canvas Area -->
  <div class="canvas-container">
    <canvas
      bind:this={canvas}
      on:mousedown={startDrawing}
      on:mousemove={draw}
      on:mouseup={stopDrawing}
      on:mouseleave={stopDrawing}
      class="canvas"
      style="cursor: {tool === 'pan' ? 'grab' : tool === 'select' ? 'default' : 'crosshair'}"
    />

    <!-- Sticky Notes Overlay -->
    {#each stickyNotes as note (note.id)}
      <div
        class="sticky-note"
        style="left: {note.x * scale + offset.x}px; top: {note.y * scale + offset.y}px; 
               width: {note.width * scale}px; height: {note.height * scale}px;
               background: {note.color}"
      >
        {#if note.editing}
          <textarea
            bind:value={note.content}
            on:blur={() => updateStickyNote(note, note.content)}
            placeholder="Type here..."
            class="sticky-textarea"
          />
        {:else}
          <div on:dblclick={() => note.editing = true} class="sticky-content">
            {note.content || 'Double-click to edit'}
          </div>
        {/if}
        <button class="sticky-delete" on:click={() => deleteStickyNote(note.id)}>×</button>
      </div>
    {/each}

    <!-- Text Input Overlay -->
    {#if editingText}
      <input
        type="text"
        bind:value={editingText.content}
        on:keydown={(e) => {
          if (e.key === 'Enter' && editingText.content) {
            texts = [...texts, editingText];
            editingText = null;
            saveState();
            redraw();
          }
        }}
        on:blur={() => editingText = null}
        placeholder="Type text..."
        class="text-input"
        style="left: {editingText.x * scale + offset.x}px; 
               top: {(editingText.y - 30) * scale + offset.y}px;
               font-size: {editingText.size * scale}px;
               color: {editingText.color}"
        autofocus
      />
    {/if}
  </div>
</div>

<style>
  :global(body) {
    margin: 0;
    padding: 0;
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  }

  .whiteboard-container {
    display: flex;
    flex-direction: column;
    height: 100vh;
    background: #f5f5f5;
  }

  .toolbar {
    display: flex;
    align-items: center;
    gap: 16px;
    padding: 12px 16px;
    background: white;
    border-bottom: 1px solid #e0e0e0;
    box-shadow: 0 2px 4px rgba(0,0,0,0.1);
    flex-wrap: wrap;
  }

  .toolbar-section {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 0 12px;
    border-right: 1px solid #e0e0e0;
  }

  .toolbar-section:last-child {
    border-right: none;
  }

  h1 {
    margin: 0;
    font-size: 20px;
    font-weight: 600;
    color: #333;
  }

  .tool-btn {
    padding: 8px 12px;
    border: 1px solid #ddd;
    background: white;
    border-radius: 6px;
    cursor: pointer;
    font-size: 18px;
    transition: all 0.2s;
  }

  .tool-btn:hover {
    background: #f0f0f0;
  }

  .tool-btn.active {
    background: #2196F3;
    color: white;
    border-color: #2196F3;
  }

  .color-picker {
    width: 40px;
    height: 40px;
    border: 2px solid #ddd;
    border-radius: 6px;
    cursor: pointer;
  }

  .color-presets {
    display: flex;
    gap: 4px;
  }

  .color-preset {
    width: 24px;
    height: 24px;
    border: 2px solid #ddd;
    border-radius: 4px;
    cursor: pointer;
    transition: transform 0.2s;
  }

  .color-preset:hover {
    transform: scale(1.1);
  }

  .slider {
    width: 100px;
  }

  button {
    padding: 8px 16px;
    border: 1px solid #ddd;
    background: white;
    border-radius: 6px;
    cursor: pointer;
    font-size: 14px;
    transition: all 0.2s;
  }

  button:hover:not(:disabled) {
    background: #f0f0f0;
  }

  button:disabled {
    opacity: 0.4;
    cursor: not-allowed;
  }

  button.danger {
    background: #f44336;
    color: white;
    border-color: #f44336;
  }

  button.danger:hover {
    background: #d32f2f;
  }

  .file-btn {
    padding: 8px 16px;
    border: 1px solid #ddd;
    background: white;
    border-radius: 6px;
    cursor: pointer;
    font-size: 14px;
    transition: all 0.2s;
  }

  .file-btn:hover {
    background: #f0f0f0;
  }

  .canvas-container {
    flex: 1;
    position: relative;
    overflow: hidden;
  }

  .canvas {
    width: 100%;
    height: 100%;
    background: white;
  }

  .sticky-note {
    position: absolute;
    padding: 12px;
    box-shadow: 0 4px 8px rgba(0,0,0,0.2);
    border-radius: 4px;
    font-family: 'Comic Sans MS', cursive;
  }

  .sticky-textarea {
    width: 100%;
    height: 100%;
    border: none;
    background: transparent;
    resize: none;
    font-family: inherit;
    font-size: 14px;
    outline: none;
  }

  .sticky-content {
    width: 100%;
    height: 100%;
    overflow: auto;
    cursor: text;
    font-size: 14px;
  }

  .sticky-delete {
    position: absolute;
    top: 4px;
    right: 4px;
    width: 24px;
    height: 24px;
    padding: 0;
    background: rgba(0,0,0,0.1);
    border: none;
    border-radius: 50%;
    font-size: 18px;
    line-height: 1;
    cursor: pointer;
  }

  .sticky-delete:hover {
    background: rgba(0,0,0,0.2);
  }

  .text-input {
    position: absolute;
    border: 2px solid #2196F3;
    border-radius: 4px;
    padding: 4px 8px;
    outline: none;
    background: white;
  }

  label {
    display: flex;
    align-items: center;
    gap: 8px;
    font-size: 14px;
    cursor: pointer;
  }
</style>