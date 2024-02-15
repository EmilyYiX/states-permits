<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';
  import App from './App.svelte';
  import MultiSelect from 'svelte-multiselect'

  export let data;

  const width = 1528;
  const height = 600;
  const marginTop = 20;
  const marginRight = 30;
  const marginBottom = 30;
  const marginLeft = 60;
  
  const states = ["Alabama", "Alaska", "Arizona", "Arkansas", "California", "Colorado", "Connecticut", "Delaware", "District of Columbia", "Florida", "Georgia", "Guam", "Hawaii", "Idaho", "Illinois", "Indiana", "Iowa", "Kansas", "Kentucky", "Louisiana", "Maine", "Mariana Islands", "Maryland", "Massachusetts", "Michigan", "Minnesota", "Mississippi", "Missouri", "Montana", "Nebraska", "Nevada", "New Hampshire", "New Jersey", "New Mexico", "New York", "North Carolina", "North Dakota", "Ohio", "Oklahoma", "Oregon", "Pennsylvania", "Puerto Rico", "Rhode Island", "South Carolina", "South Dakota", "Tennessee", "Texas", "Utah", "Vermont", "Virgin Islands", "Virginia", "Washington", "West Virginia", "Wisconsin", "Wyoming"];
  let selected = ["California", "Illinois", "North Carolina"];
  let svg;
  let gx;
  let gy;
  let dot;
  let path;

  $: x = d3
    .scaleUtc()
    .domain(d3.extent(data, (d) => d.date))
    .range([marginLeft, width - marginRight]);

  $: y = d3
    .scaleLinear()
    .domain([0, 600000])
    .nice()
    .range([height - marginBottom, marginTop]);

  $: line = d3
    .line()
    .x((d) => x(d.date))
    .y((d) => y(d.permit));

  $: filtered = data.filter(function(d){ return selected.includes(d.state) })

  $: dataByStates = d3.group(
      filtered, 
      d => d.state
    );
  
  
  $: path = d3.select(svg).append("g")
      .attr("fill", "none")
      .attr("stroke", "steelblue")
      .attr("stroke-width", 1.5)
      .attr("stroke-linejoin", "round")
      .attr("stroke-linecap", "round")
    .selectAll("path")
    .data(dataByStates.values())
    .join("path")
      .style("mix-blend-mode", "multiply")
      .attr("d", line);


  $: d3.select(gx).call(d3.axisBottom(x).ticks(width / 80));
  $: d3.select(gy)
    .call(d3.axisLeft(y).ticks(null))
    // zero line
    .call((g) =>
      g
        .selectAll('.tick line')
        .clone()
        .attr('x2', width - marginRight - marginLeft)
        .attr('stroke-opacity', (d) => (d === 0 ? 1 : 0.1)),
    );
  $: points = filtered.map((d) => [x(d.date), y(d.permit), d.state]);

  $: dot = d3.select(svg).append("g")
      .attr("display", "none");

  $: dot.append("circle")
      .attr("r", 2.5);

  $: dot.append("text")
      .attr("text-anchor", "middle")
      .attr("y", -8);

  $: d3.select(svg)
      .on("pointerenter", pointerentered)
      .on("pointermove", pointermoved)
      .on("pointerleave", pointerleft)
      .on("touchstart", event => event.preventDefault());


const formatDate = d3.timeFormat("%Y");
const formatPermit = d3.format(",");

// Modify the pointermoved function
function pointermoved(event) {
  const [xm, ym] = d3.pointer(event, this); // Get the mouse position relative to the SVG element
  let hoveredState = null;
  let closestDistance = Infinity;
  let closestPoint = null;

  // Find the closest point by looping over each state's data
  for (const [state, points] of dataByStates) {
    points.forEach(point => {
      const xCoord = x(point.date);
      const yCoord = y(point.permit);
      const distance = Math.sqrt((xm - xCoord) ** 2 + (ym - yCoord) ** 2);
      if (distance < closestDistance) {
        closestDistance = distance;
        hoveredState = state;
        closestPoint = point;
      }
    });
  }

  if (closestPoint) {
    const xCoord = x(closestPoint.date);
    const yCoord = y(closestPoint.permit);
    dot.attr("transform", `translate(${xCoord},${yCoord})`);
    dot.select("text").text(`${hoveredState} (${formatDate(closestPoint.date)}, ${formatPermit(closestPoint.permit)})`);
    dot.attr("display", null);

    // Highlight the hovered line
    path.style("stroke", d => d[0].state === hoveredState ? "tomato" : "#ddd")
        .style("stroke-width", d => d[0].state === hoveredState ? 3 : 1);
  } else {
    // If no close point is found, hide the dot and reset the lines
    dot.attr("display", "none");
    path.style("stroke", "steelblue").style("stroke-width", 1.5);
  }
}

  function pointerentered() {
    path.style("mix-blend-mode", null).style("stroke", "#ddd");
    dot.attr("display", null);
  }

  function pointerleft() {
    path.style("mix-blend-mode", "multiply").style("stroke", null);
    dot.attr("display", "none");
    d3.select(svg).node().value = null;
    d3.select(svg).dispatch("input", {bubbles: true});
  }

  function redraw() {
  // Assuming dataByStates is correctly updated before this function is called

  // Bind the updated data to the paths
  const paths = d3.select(svg).selectAll("path")
    .data(Array.from(dataByStates.values()).filter(d => d && d.length > 0), d => d ? d[0].state : '');

  // Enter + Update
  paths.enter()
    .append("path")
      .attr("fill", "none")
      .attr("stroke", "steelblue")
      .attr("stroke-width", 1.5)
      .attr("stroke-linejoin", "round")
      .attr("stroke-linecap", "round")
      .attr("d", line)
      .merge(paths) // Merges the enter and update selections
      .style("mix-blend-mode", "multiply");

  // Exit
  paths.exit().remove();
}

  function handleRemove(event) {
    const stateToRemove = event.detail; // Get the state to remove from the event details

    // Update selected states
    selected = selected.filter(item => item !== stateToRemove);

    // Update the dataByStates with the new selection
    const filtered = data.filter(d => selected.includes(d.state));
    dataByStates = d3.group(filtered, d => d.state);

    // Redraw the graph with updated data
    redraw();
  }

</script>

<h1>Permits Per State Over Time</h1>
<MultiSelect bind:selected options={states} on:remove={handleRemove} />
<div class="graph">
    <svg
        bind:this={svg}
        {width}
        {height}
        viewBox="0 0 {width} {height}"
        style="max-width: 100%; height: auto;"
      >
      <g bind:this={gx} transform="translate(0, {height-marginBottom})"/>
      <g bind:this={gy} transform="translate({marginLeft},0)">
        <text
          x="5"
          y={marginTop}
          dy="0.32em"
          fill="#000"
          font-weight="bold"
          text-anchor="start"
        >
          Permits
        </text>
      </g>
    </svg>
</div>