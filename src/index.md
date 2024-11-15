---
theme: dashboard
title: Navigation Prototype
toc: false
---

<!-- The source data currently used for this project is from the CosMx SMI
NSCLC FFPE Dataset: 
https://nanostring.com/products/cosmx-spatial-molecular-imager/ffpe-dataset/nsclc-ffpe-dataset/ -->

<style>
.legend {
  font-family: var(--sans-serif);
  font-size: 12px;
  fill: white;
}

.legend rect {
  stroke-width: 1;
  stroke: #ccc;
}

.axis text,
.axis path,
.axis line {
  fill: white;
  stroke: none;
}

.tooltip {
  position: absolute;
  text-align: center;
  width: auto;
  height: auto;
  padding: 5px;
  font: 12px sans-serif;
  background: white;
  border: 0;
  border-radius: 3px;
  pointer-events: none;
  opacity: 0;
}
</style>

<div>${chart()}</div>

```js
// Prevent default page zoom on Ctrl + scroll and pinch gestures
document.addEventListener('wheel', function(event) {
  if (event.ctrlKey) {
    event.preventDefault();
  }
}, { passive: false });

document.addEventListener('gesturestart', function(event) {
  event.preventDefault();
});

function setupCanvas(width, height) {
  return d3.select("body").append("canvas")
    .attr("width", width)
    .attr("height", height)
    .style("max-width", "100%")
    .style("height", "auto")
    .node();
}

function loadData() {
  return FileAttachment("cell-annotations.csv").csv();
}

function createScales(data, width, height) {
  const xExtent = d3.extent(data, d => +d["sdimx"]);
  const yExtent = d3.extent(data, d => +d["sdimy"]);

  const xscaling = (xExtent[1] - xExtent[0]) / (yExtent[1] - yExtent[0]);

  const x = d3.scaleLinear()
    .domain(xExtent)
    .range([0, xscaling * width]);

  const y = d3.scaleLinear()
    .domain(yExtent)
    .range([height, 0]);

  return { x, y };
}

function loadSprites() {
  const isLocal = window.location.hostname === 'localhost' || window.location.hostname === '127.0.0.1';
  const imagePath = isLocal ? 'http://localhost:3000/images/' : './images/';  // Modify this path for GitHub Pages deployment

  const spriteMapping = {
    "plasma": imagePath + "b-cell.png",
    "B.naive": imagePath + "b-cell.png",
    "fibroblasts": imagePath + "caf.png",
    "Cancer": imagePath + "cancer.png",
    "pDCs": imagePath + "dc.png",
    "mDCs": imagePath + "dc.png",
    "macrophages": imagePath + "mac.png",
    "monocytes.NC.I": imagePath + "mono.png",
    "monocytes.NC.I": imagePath + "mono.png",
    "mast": imagePath + "mono.png",
    "NK": imagePath + "tcell.png",
    "T.CD8.naive": imagePath + "tcell.png",
    "T.CD4.memory": imagePath + "tcell.png",
    "T.CD4.naive": imagePath + "tcell.png",
    "Treg": imagePath + "tcell.png",
    "endothelial.cells": imagePath + "no-mapping.png",
    "B.memory": imagePath + "b-cell.png",
  };

  const spriteImages = {};
  const loadImages = Object.entries(spriteMapping).map(([label, url]) => {
    return new Promise((resolve, reject) => {
      const img = new Image();
      img.src = url;
      img.onload = () => {
        spriteImages[label] = img;
        resolve();
      };
      img.onerror = () => {
        console.error(`Failed to load image for ${label} from ${url}`);
        reject();
      };
    });
  });

  return { spriteImages, loadImages };
}

function renderChart(context, data, x, y, color, spriteImages, defaultDotRadius, dotScale, defaultImageRadius, imageScale, maxZoom, width, height, marginTop, marginRight) {
  function render(transform) {
    context.clearRect(0, 0, width, height);
    context.save();
    context.translate(transform.x, transform.y);
    context.scale(transform.k, transform.k);

    const dotRadius = defaultDotRadius + (dotScale * (transform.k - 1) / (maxZoom - 1));
    const imageRadius = defaultImageRadius + (imageScale * (transform.k - 1) / (maxZoom - 1));

    if (transform.k > 8) {
      data.forEach(d => {
        const img = spriteImages[d["immune_cell_labels"]];
        if (img) {
          context.drawImage(img, x(d["sdimx"]) - (imageRadius), y(d["sdimy"]) - (imageRadius), imageRadius * 2, imageRadius * 2);
        }
      });
    } else {
      data.forEach(d => {
        context.beginPath();
        context.arc(x(d["sdimx"]), y(d["sdimy"]), dotRadius, 0, 2 * Math.PI);
        context.fillStyle = color(d["immune_cell_labels"]);
        context.fill();
      });
    }

    context.restore();

    drawAxes(context, x, y);
    drawLegend(context, color, width, marginTop, marginRight);
  }

  render(d3.zoomIdentity);

  d3.select(context.canvas)
    .call(d3.zoom()
      .scaleExtent([1, maxZoom])
      .translateExtent([[0, 0], [width, height]])
      .extent([[0, 0], [width, height]])
      .on("zoom", event => render(event.transform))
    );
}

function drawAxes(context, x, y) {
  // Axes drawing logic should be implemented here.
}

function drawLegend(context, color, width, marginTop, marginRight) {
  const legendData = color.domain();
  legendData.forEach((d, i) => {
    context.fillStyle = color(d);
    context.fillRect(width - marginRight + 20, marginTop + i * 20, 18, 18);
    context.fillStyle = "#FFF";
    context.fillText(d, width - marginRight + 44, marginTop + i * 20 + 14);
  });
}

function chart() {
  // Configurable parameters
  const defaultDotRadius = 0.8;
  const dotScale = 0.3;

  const defaultImageRadius = 1.2;
  const imageScale = 0.3;

  const maxZoom = 40;

  // Parameters
  const width = window.innerWidth;
  const height = window.innerHeight;
  const marginTop = 20;
  const marginRight = 150;
  const marginBottom = 30;
  const marginLeft = 40;

  const canvas = setupCanvas(width, height);
  const context = canvas.getContext("2d");

  loadData().then(data => {
    if (!data || data.length === 0) {
      console.error("Data not loaded correctly or empty.");
      return;
    }

    const { x, y } = createScales(data, width, height);
    const color = d3.scaleOrdinal()
      .domain(data.map(d => d["cell_labels_level1"]))
      .range(d3.schemeCategory10);

    const { spriteImages, loadImages } = loadSprites();

    Promise.all(loadImages).then(() => {
      renderChart(context, data, x, y, color, spriteImages, defaultDotRadius, dotScale, defaultImageRadius, imageScale, maxZoom, width, height, marginTop, marginRight);
    }).catch(error => {
      console.error("Error loading images:", error);
    });
  }).catch(error => {
    console.error("Error loading data:", error);
  });

  return canvas;
}



