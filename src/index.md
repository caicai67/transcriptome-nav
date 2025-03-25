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

<div id="container" style="position:relative; width:100vw; height:100vh;">
    <div id="openseadragon-viewer" style="position:absolute; top:0; left:0; width:100%; height:100%;"></div>
    <div style="position:absolute; top:0; left:0; width:100%; height:100%; pointer-events:none;">${chart()}</div>
    <!-- <canvas id="overlayCanvas" style="position:absolute; top:0; left:0; width:100%; height:100%; pointer-events:none;"></canvas> -->
</div>

<script src="https://openseadragon.github.io/openseadragon/openseadragon.min.js"></script>
<!-- <script src="/openseadragon/openseadragon.min.js"></script> TODO: switch to this and fix server endpoint -->
<script type="text/javascript">
    var viewer = OpenSeadragon({
        id: "openseadragon-viewer",
        prefixUrl: "https://openseadragon.github.io/openseadragon/images/",
        tileSources: "http://localhost:3000/Lung5-3_image2.dzi",
        crossOriginPolicy: "Anonymous", // Allow cross-origin image loading
    });
</script>

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
    "neutrophils": imagePath + "neutro.png",
    "plasma": imagePath + "b-cell.png",
    "B.naive": imagePath + "b-cell.png",
    "fibroblasts": imagePath + "caf.png",
    "Cancer": imagePath + "cancer.png",
    "pDCs": imagePath + "dc.png",
    "mDCs": imagePath + "dc.png",
    "macrophages": imagePath + "mac.png",
    "monocytes.NC.I": imagePath + "mac.png",
    "monocytes.NC.I": imagePath + "mac.png",
    "mast": imagePath + "mast.png",
    "NK": imagePath + "nk.png",
    "T.CD8.memory": imagePath + "cd8.png",
    "T.CD8.naive": imagePath + "cd8.png",
    "T.CD4.memory": imagePath + "cd4.png",
    "T.CD4.naive": imagePath + "cd4.png",
    "Treg": imagePath + "cd4.png",
    "endothelial.cells": imagePath + "endo.png",
    "B.memory": imagePath + "b-cell.png",

    "stainImg": imagePath + "small-Lung5-3_image2.png",
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

function renderChart(context, data, x, y, color, spriteImages, defaultDotRadius, dotScale, defaultImageRadius, imageScale, maxZoom, width, height, marginTop, marginRight, viewRadius) {
    const spriteZoomLevel = 8; // Define the zoom level at which sprites replace dots

    function render(transform) {
        context.clearRect(0, 0, width, height);
        context.save();
        context.translate(transform.x, transform.y);
        context.scale(transform.k, transform.k);
        context.globalAlpha = 0.5;

        //context.drawImage(spriteImages["stainImg"], 0, 0, width+360, height);
        context.restore();

        // Draw the inner circle using a clipping mask
        context.save();
        context.beginPath();
        context.arc(width/2, height/2, viewRadius, 0, Math.PI * 2);
        context.clip();

        //draw sprites
        context.translate(transform.x, transform.y);
        context.scale(transform.k, transform.k);


        const zoomFactor = transform.k; // Current zoom level

        // Calculate the dot radius based on zoom level
        let dotRadius;
        if (zoomFactor < spriteZoomLevel) {
            dotRadius = defaultDotRadius * (1.5 - (0.5 * (zoomFactor - 1) / (spriteZoomLevel - 1)));
        } else {
            dotRadius = defaultDotRadius; // for images, use default size
        }

        // Calculate image radius for sprites
        const imageRadius = defaultImageRadius + (imageScale * (zoomFactor - spriteZoomLevel) / (maxZoom - spriteZoomLevel));

        // Render points or sprites based on zoom level
        if (zoomFactor >= spriteZoomLevel) {
            data.forEach(d => {
                const img = spriteImages[d["immune_cell_labels"]]; //TODO: put this string in the config on the chart and pass in to this method
                if (img) {
                    context.drawImage(img, x(d["sdimx"]) - imageRadius, y(d["sdimy"]) - imageRadius, imageRadius * 2, imageRadius * 2);
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

        context.save();
        drawAxes(context, x, y);
        drawLegend(context, color, width, marginTop, marginRight);

        context.restore();
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
    let legendData = color.domain(); // Get the unique labels from the color domain

    // Sort the legend items by color to group similar colors together
    legendData = legendData.sort((a, b) => d3.ascending(color(a), color(b)));

    // Adjust the position and dimensions of the legend
    const legendWidth = 150; // Adjust to fit your text
    const legendHeight = legendData.length * 24 + 10; // Add spacing for all rows
    const legendX = width - marginRight + 50; // Move further to the right
    const legendY = marginTop - 10;

    // Draw the white background for the entire legend
    context.fillStyle = "#FFFFFF"; // White background
    context.fillRect(legendX, legendY, legendWidth, legendHeight);

    // Set font style for the legend text
    context.font = "bold 14px sans-serif";
    context.textBaseline = "middle";

    // Loop through the legend data to render each text item
    legendData.forEach((d, i) => {
        const textX = legendX + 10; // Padding inside the legend
        const textY = marginTop + i * 24 + 5;

        // Draw the text in the corresponding color
        context.fillStyle = color(d); // Match text color to legend color
        context.fillText(d, textX, textY);
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
  const viewRadius = 200;

  const canvas = setupCanvas(width, height);
  const context = canvas.getContext("2d");

  loadData().then(data => {
    if (!data || data.length === 0) {
      console.error("Data not loaded correctly or empty.");
      return;
    }

    const { x, y } = createScales(data, width, height);

    const color = d3.scaleOrdinal()
    .domain(["B.memory", "B.naive", "Cancer", "endothelial.cells", "fibroblasts", "macrophages", "mast", "mDCs", "monocytes.C", "monocytes.NC.I", "neutrophils", "NK", "pDCs", "plasma", "T.CD4.memory", "T.CD4.naive", "T.CD8.memory", "T.CD8.naive", "Treg"])
    .range(["#5755fe", "#5755fe", "#dbb295", "#6a7a8a", "#181818", "#44af5f", "#fbad27", "#d2d429", "#44af5f", "#44af5f", "#44af5f", "#f6a9ed", "#d2d429", "#5755fe", "#e53902", "#e53902", "#2ed7d5", "#2ed7d5", "#e53902"]);

    const { spriteImages, loadImages } = loadSprites();

    Promise.all(loadImages).then(() => {
      renderChart(context, data, x, y, color, spriteImages, defaultDotRadius, dotScale, defaultImageRadius, imageScale, maxZoom, width, height, marginTop, marginRight, viewRadius);
    }).catch(error => {
      console.error("Error loading images:", error);
    });

  }).catch(error => {
    console.error("Error loading data:", error);
  });


  return canvas;
}



