---
theme: dashboard
title: Navigation Prototype
toc: false
---

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
function chart() {
  const width = 640;
  const height = 640;
  const marginTop = 20;
  const marginRight = 150; // Increased margin to accommodate legend
  const marginBottom = 30;
  const marginLeft = 40;
  const defaultRadius = 0.8;

  // Zoom configuration
  const maxZoom = 10; // Maximum zoom
  const dotScale = 3; // How much dots expand as the user zooms in

  // Create the SVG container.
  const svg = d3.create("svg")
    .attr("viewBox", [0, 0, width, height])
    .style("max-width", "100%")
    .style("height", "auto")
    .property("value", []);

  FileAttachment("cell-annotations.csv").csv().then(data => {
    const xExtent = d3.extent(data, d => +d["sdimx"]);
    const yExtent = d3.extent(data, d => +d["sdimy"]);

    const xscaling = (xExtent[1] - xExtent[0]) / (yExtent[1] - yExtent[0]);

    const x = d3.scaleLinear()
      .domain(xExtent)
      .range([0, xscaling * width]);

    const y = d3.scaleLinear()
      .domain(yExtent)
      .range([width, 0]);

    // Define the mapping between categories and their corresponding sprite images
    const spriteMapping = {
      "CAF": "assets/images/caf.png",
      "Cancer": "assets/images/cancer.png",
      // Add more categories and their corresponding sprite images
    };

    console.log("Sprite Mapping:", spriteMapping);

    const color = d3.scaleOrdinal()
      .domain(data.map(d => d["cell_labels_level1"]))
      .range(d3.schemeCategory10);

    // Append the dots.
    const dots = svg.append("g")
      .attr("clip-path", "url(#clip)")
      .selectAll("circle")
      .data(data)
      .join("circle")
      .attr("fill", d => color(d["cell_labels_level1"]))
      .attr("transform", d => `translate(${x(d["sdimx"])},${y(d["sdimy"])})`)
      .attr("r", defaultRadius)
      .on("mouseover", function (event, d) {
        d3.select(this).attr("r", 4).attr("stroke", "black");
        tooltip.transition()
          .duration(200)
          .style("opacity", .9);
        tooltip.html(`Label: ${d.cell_labels_level1}<br/>X: ${d.sdimx}<br/>Y: ${d.sdimy}`)
          .style("left", (event.pageX + 5) + "px")
          .style("top", (event.pageY - 28) + "px");
      })
      .on("mouseout", function (d) {
        d3.select(this).attr("r", defaultRadius).attr("stroke", null);
        tooltip.transition()
          .duration(500)
          .style("opacity", 0);
      });

    // Create tooltip div
    const tooltip = d3.select("body").append("div")
      .attr("class", "tooltip");

    // Append the axes.
    const xAxis = svg.append("g")
      .attr("transform", `translate(0,${height - marginBottom})`)
      .attr("class", "axis")
      .call(d3.axisBottom(x))
      .call(g => g.select(".domain").remove())
      .call(g => g.append("text")
        .attr("x", width - marginRight)
        .attr("y", -4)
        .attr("fill", "#fff")
        .attr("font-weight", "bold")
        .attr("text-anchor", "end")
        .text("x"));

    const yAxis = svg.append("g")
      .attr("transform", `translate(${marginLeft},0)`)
      .attr("class", "axis")
      .call(d3.axisLeft(y))
      .call(g => g.select(".domain").remove())
      .call(g => g.select(".tick:last-of-type text").clone()
        .attr("x", 4)
        .attr("text-anchor", "start")
        .attr("font-weight", "bold")
        .attr("fill", "#fff")
        .text("y"));

    // Create legend items
    const legend = svg.selectAll(".legend")
      .data(color.domain())
      .enter().append("g")
      .attr("class", "legend")
      .attr("transform", (d, i) => `translate(${width - marginRight + 20}, ${i * 20 + marginTop})`);

    // Append rectangles for the color boxes
    legend.append("rect")
      .attr("width", 18)
      .attr("height", 18)
      .style("fill", d => color(d));

    // Append text labels
    legend.append("text")
      .attr("x", 24)
      .attr("y", 9)
      .attr("dy", ".35em")
      .text(d => d);

    // Zoom behavior
    const zoom = d3.zoom()
      .scaleExtent([1, maxZoom])
      .translateExtent([[0, 0], [width, height]])
      .extent([[0, 0], [width, height]])
      .on("zoom", zoomed);

    svg.call(zoom);

    function zoomed(event) {
      const transform = event.transform;
      const newX = transform.rescaleX(x);
      const newY = transform.rescaleY(y);
      xAxis.call(d3.axisBottom(newX));
      yAxis.call(d3.axisLeft(newY));

      dots.attr("transform", d => `translate(${newX(d["sdimx"])},${newY(d["sdimy"])})`);

      if (transform.k > 8) {
        dots.attr("r", 0); // Hide dots
        const sprites = svg.selectAll(".sprite")
          .data(data);

        sprites.enter().append("image")
          .attr("class", "sprite")
          .attr("xlink:href", d => {
            const url = spriteMapping[d["cell_labels_level1"]];
            console.log(`Using sprite for ${d["cell_labels_level1"]}: ${url}`);
            return url;
          }) // Use the sprite mapping
          .attr("width", 10)
          .attr("height", 10)
          .attr("transform", d => `translate(${newX(d["sdimx"]) - 5},${newY(d["sdimy"]) - 5})`)
          .merge(sprites)
          .attr("transform", d => `translate(${newX(d["sdimx"]) - 5},${newY(d["sdimy"]) - 5})`);

        sprites.exit().remove();
      } else {
        dots.attr("r", defaultRadius + (dotScale * (transform.k - 1) / (maxZoom - 1))); // Adjust the radius based on the zoom level
        svg.selectAll(".sprite").remove(); // Remove sprites
      }
    }
  });

  return svg.node();
}
