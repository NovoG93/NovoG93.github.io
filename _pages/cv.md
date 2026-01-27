---
layout: archive
title: "CV"
permalink: /cv/
author_profile: true
redirect_from:
  - /resume
---

<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf_viewer.min.css">

<style>
.cv-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
  flex-wrap: wrap;
  gap: 15px;
}
.cv-download {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  background: linear-gradient(135deg, #58a6ff, #a371f7);
  color: #fff !important;
  padding: 12px 24px;
  border-radius: 30px;
  text-decoration: none !important;
  font-weight: 600;
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  box-shadow: 0 4px 15px rgba(88, 166, 255, 0.3);
}
.cv-download:hover {
  transform: translateY(-2px) scale(1.02);
  box-shadow: 0 8px 25px rgba(88, 166, 255, 0.4);
}
.cv-controls {
  display: flex;
  gap: 10px;
  align-items: center;
  flex-wrap: wrap;
}
.cv-btn {
  background: #21262d;
  border: 1px solid #30363d;
  color: #c9d1d9;
  padding: 8px 16px;
  border-radius: 8px;
  cursor: pointer;
  font-size: 14px;
  transition: all 0.2s ease;
}
.cv-btn:hover {
  background: #30363d;
  border-color: #58a6ff;
  color: #58a6ff;
}
.page-info {
  color: #8b949e;
  font-size: 14px;
}
#pdf-container {
  background: #161b22;
  border-radius: 8px;
  padding: 20px;
  min-height: 80vh;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 20px;
  overflow-x: auto;
}
.page-wrapper {
  position: relative;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.5);
  border-radius: 4px;
  overflow: visible;
}
.page-wrapper canvas {
  display: block;
}
.textLayer {
  position: absolute;
  left: 0;
  top: 0;
  right: 0;
  bottom: 0;
  overflow: hidden;
  opacity: 0.25;
  line-height: 1.0;
  pointer-events: all;
}
.textLayer > span {
  color: transparent;
  position: absolute;
  white-space: pre;
  cursor: text;
  transform-origin: 0% 0%;
}
.textLayer ::selection {
  background: rgba(88, 166, 255, 0.5);
}
.link-layer {
  position: absolute;
  left: 0;
  top: 0;
  right: 0;
  bottom: 0;
  pointer-events: none;
  z-index: 10;
}
.link-layer a {
  position: absolute;
  pointer-events: all;
  cursor: pointer;
  border-radius: 2px;
  transition: background 0.2s ease;
}
.link-layer a:hover {
  background: rgba(88, 166, 255, 0.25) !important;
}
.loading {
  color: #58a6ff;
  font-size: 18px;
  padding: 40px;
}
</style>

<div class="cv-header">
  <a href="/files/CV.pdf" download class="cv-download">
    📄 Download CV
  </a>
  <div class="cv-controls">
    <button class="cv-btn" id="prev-page">← Prev</button>
    <span class="page-info"><span id="page-num">1</span> / <span id="page-count">-</span></span>
    <button class="cv-btn" id="next-page">Next →</button>
    <button class="cv-btn" id="zoom-out">−</button>
    <button class="cv-btn" id="zoom-in">+</button>
  </div>
</div>

<div id="pdf-container">
  <div class="loading">Loading CV...</div>
</div>

<script src="https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.min.js"></script>
<script>
document.addEventListener('DOMContentLoaded', function() {
  const url = '/files/CV.pdf';
  const container = document.getElementById('pdf-container');
  const pageNumSpan = document.getElementById('page-num');
  const pageCountSpan = document.getElementById('page-count');
  
  let pdfDoc = null;
  let currentPage = 1;
  let scale = 1.5;
  const pageWrappers = [];

  pdfjsLib.GlobalWorkerOptions.workerSrc = 'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.worker.min.js';

  async function renderPage(pageNum, wrapper) {
    const page = await pdfDoc.getPage(pageNum);
    const viewport = page.getViewport({ scale: scale });
    
    wrapper.innerHTML = '';
    wrapper.style.width = viewport.width + 'px';
    wrapper.style.height = viewport.height + 'px';
    
    /* Canvas layer */
    const canvas = document.createElement('canvas');
    canvas.height = viewport.height;
    canvas.width = viewport.width;
    wrapper.appendChild(canvas);
    
    const ctx = canvas.getContext('2d');
    await page.render({
      canvasContext: ctx,
      viewport: viewport
    }).promise;
    
    /* Text layer */
    const textLayer = document.createElement('div');
    textLayer.className = 'textLayer';
    textLayer.style.width = viewport.width + 'px';
    textLayer.style.height = viewport.height + 'px';
    wrapper.appendChild(textLayer);
    
    const textContent = await page.getTextContent();
    pdfjsLib.renderTextLayer({
      textContent: textContent,
      container: textLayer,
      viewport: viewport,
      textDivs: []
    });
    
    /* Link layer for clickable hyperlinks */
    const linkLayer = document.createElement('div');
    linkLayer.className = 'link-layer';
    wrapper.appendChild(linkLayer);
    
    const annotations = await page.getAnnotations();
    console.log('Page', pageNum, 'annotations:', annotations);
    
    annotations.forEach(annotation => {
      if (annotation.subtype === 'Link') {
        const link = document.createElement('a');
        
        /* Handle different link types */
        if (annotation.url) {
          link.href = annotation.url;
          link.target = '_blank';
          link.rel = 'noopener noreferrer';
        } else if (annotation.dest) {
          /* Internal link - skip for now */
          return;
        } else if (annotation.action && annotation.action.uri) {
          link.href = annotation.action.uri;
          link.target = '_blank';
          link.rel = 'noopener noreferrer';
        } else {
          return;
        }
        
        /* Position the link */
        const rect = annotation.rect;
        if (rect) {
          const [x1, y1, x2, y2] = viewport.convertToViewportRectangle(rect);
          
          const left = Math.min(x1, x2);
          const top = Math.min(y1, y2);
          const width = Math.abs(x2 - x1);
          const height = Math.abs(y2 - y1);
          
          link.style.left = left + 'px';
          link.style.top = top + 'px';
          link.style.width = width + 'px';
          link.style.height = height + 'px';
          
          linkLayer.appendChild(link);
        }
      }
    });
  }

  async function renderAllPages() {
    container.innerHTML = '';
    pageWrappers.length = 0;
    
    for (let i = 1; i <= pdfDoc.numPages; i++) {
      const wrapper = document.createElement('div');
      wrapper.className = 'page-wrapper';
      container.appendChild(wrapper);
      pageWrappers.push(wrapper);
      await renderPage(i, wrapper);
    }
  }

  async function init() {
    try {
      pdfDoc = await pdfjsLib.getDocument(url).promise;
      pageCountSpan.textContent = pdfDoc.numPages;
      await renderAllPages();
    } catch (error) {
      console.error('PDF load error:', error);
      container.innerHTML = '<div class="loading">Error loading PDF. <a href="/files/CV.pdf" style="color: #58a6ff;">Download instead</a></div>';
    }
  }

  document.getElementById('zoom-in').addEventListener('click', async () => {
    scale += 0.25;
    await renderAllPages();
  });

  document.getElementById('zoom-out').addEventListener('click', async () => {
    if (scale > 0.5) {
      scale -= 0.25;
      await renderAllPages();
    }
  });

  document.getElementById('prev-page').addEventListener('click', () => {
    if (currentPage > 1) {
      currentPage--;
      pageNumSpan.textContent = currentPage;
      pageWrappers[currentPage - 1].scrollIntoView({ behavior: 'smooth', block: 'start' });
    }
  });

  document.getElementById('next-page').addEventListener('click', () => {
    if (currentPage < pdfDoc.numPages) {
      currentPage++;
      pageNumSpan.textContent = currentPage;
      pageWrappers[currentPage - 1].scrollIntoView({ behavior: 'smooth', block: 'start' });
    }
  });

  init();
});
</script>
