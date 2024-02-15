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
    const verticalLineClass = 'vertical-line';

    if (d3.select(svg).select("." + verticalLineClass).empty()) {
      d3.select(svg)
        .append("line") 
        .attr("class", verticalLineClass)
        .attr("x1", marginLeft) 
        .attr("y1", marginBottom-10) 
        .attr("x2", marginLeft) 
        .attr("y2", height-30) 
        .attr("stroke", "black") 
        .attr("stroke-width", 1); 
    }
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

    paths.exit().remove();
  }

  function removeState(stateToRemove) {
    // Update selected states
    selected = selected.filter(item => item !== stateToRemove);

    // Update the dataByStates with the new selection
    const filtered = data.filter(d => selected.includes(d.state));
    dataByStates = d3.group(filtered, d => d.state);

    // Redraw the graph with updated data
    redraw();
  }

  function handleRemove(event) {
    const stateToRemove = event.detail; // Get the state to remove from the event details
    removeState(stateToRemove)
  }

  function handleRemoveAll(event) {
    event.detail.options.forEach((stateToRemove) => removeState(stateToRemove));
  }
  
onMount(() => {
  // Existing setup for drawing the chart goes here...

  // Append a text element for the x-axis label positioned at the rightmost part
  d3.select(svg)
    .append("text") // Append a new text element
    .attr("class", "x-axis-label") // Optional: Assign a class for styling
    .attr("text-anchor", "end") // Anchor the text at the end for right alignment
    .attr("x", width - marginRight- 700) // Position horizontally at the right edge of the SVG, considering the margin
    .attr("y", height) // Position vertically just above the bottom of the SVG, adjust as needed
    .text("Year"); // Set the text for the label
});

</script>

<h1>Permits Per State Over Time</h1>
<MultiSelect bind:selected options={states} on:remove={handleRemove} on:removeAll={handleRemoveAll} />
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
          font-size=15px
        >
          Permits
        </text>
      </g>
    </svg>
</div>
<div class="write-up">
  <h1>Writeup</h1>
  <p>
    This interactive data visualization enables users to explore and compare the trends of gun permit issuance across all states some territories of the USA over a period of more than two decades. The visualization was constructed using <span class="highlight">Svelte</span> and <span class="highlight">D3.js</span>.
  </p>
  
  <h2>Rationale for Design Decisions</h2>
  <p>
    Our design decisions were guided by the goal to provide a clear and comparative view. A multi-line chart was chosen for its representation of time series data, which helps in learning trends over time. The distinct lines for each state allow for quick visual differentiation and comparison. The x-axis shows the timeline in years from 1999 to 2023. The y-axis displays the number of permits issued with the intervals being of 50,000 each. Each point on each line represents the number of permits issued for that state in a particular year.
  </p>
  <p>
    A hover tooltip was implemented to provided detailed information: the state name, year, and number of permits issued. Looking at the data and the possibility of having multiple lines at once on the graph, we decided to make the hovered line turn bold and red. This was done to help focus on the state that the user wants to examine and compare with other states on the graph. This hover tooltip was implemented in a way that it is not necessary for the user to put the cursor exactly on the line as that can be cumbersome. Instead, the implementation is such that it calculates the nearest point on the nearest line on the graph. Then it highlights that state red and displays the information close to the cursor for easy reading. In the background, if more states are present, those states become a lighter shade compared to the highlighted state to facilitate easy comparison.
  <p>
    The multi-select feature allows users to customize the data displayed by selecting all or either a subset of the states that the user wants to visualize. This approach was favored over checkboxes, as we thought it is easier to type in or select a state in a search bar than going through a long list of checkboxes and clicking small boxes to include or exclude. The multiselct bar shows all the states in alphabetical order and when selected shows them side by side on the bar itself with a remove option on it. This bar also has a clear all option in the form of a cross at the right side of the bar to reset the graph if the user wants to start over.
  </p>
  <p>
    We considered various alternatives for visual encodings, including area and stacked line charts. However, we decided against these to prevent individual state data visualization or a subset of states. Our priority was to maintain the clarity of individual state trends while allowing for comparisons with other states.
  </p>

  <h2>Contributions</h2>
  <p>Emily set up the Svelte project and the basic line chart. It took a lot of time to set up the Svelte project because I did it before the template was posted on EdStem to use. I think it took about 5 hours for me to get a GitHub pages site set up initially. Housheng participated in and optimized the interactive functionality of visualization proposed by Emily, and designed the functionality of mouseover on lines that shows more information for each state. During this process, in order to solve the problem that the active function only draw plots but cannot clean any of them, Housheng spent a lot of time on designing a suitable redraw function and solving the related debugging problems, which took about 6hrs in total. Siddhant was tasked with doing the write up and a zoom functionality. However, I could not completely implement the zoom functionality for the visualization. Attempting the zoom functionality took about 4 hours but could not be presented for the submission. Siddhant then added some CSS styling for the svg, styling for the tooltip, and fixed bugs with axis, axis labels, and axis ticks. This took about 1.5 hours.</p>
</div>

<style>
  svg {
  display: block;
  max-width: 100%;
  height: auto;
  margin: 0 auto;
}
path.line {
  fill: none;
  stroke: steelblue;
  stroke-width: 2;
  transition: all 0.3s ease-out;
}
.axis path,
.axis line {
  fill: none;
  stroke: #ccc;
  shape-rendering: crispEdges;
}

.axis text {
  font: 12px sans-serif;
  color: #666;
}

.axis .domain {
  stroke: none;
}

div.tooltip {
  position: absolute;
  text-align: center;
  width: auto;
  padding: 8px;
  font: 12px sans-serif;
  background: rgba(255, 255, 255, 0.8);
  border: 0;
  border-radius: 8px;
  pointer-events: none;
  box-shadow: 0 2px 4px rgba(0,0,0,0.2);
}

</style>