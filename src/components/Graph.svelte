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


  function pointermoved(event) {
    const [xm, ym] = d3.pointer(event);
    const i = d3.leastIndex(points, ([x, y]) => Math.hypot(x - xm, y - ym));
    if(!points[i]){
      return;
    }
    const [x, y, k] = points[i];
    path.style("stroke", ({z}) => z === k ? null : "#ddd").filter(({z}) => z === k).raise();
    dot.attr("transform", `translate(${x},${y})`);
    dot.select("text").text(k);
    d3.select(svg).property("value", filtered[i]).dispatch("input", {bubbles: true});
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
  // Clear the SVG content
  d3.select(svg).selectAll("*").remove();

  // Recreate the axes groups to avoid removal on redraw
  d3.select(svg).append("g").attr("transform", `translate(0, ${height - marginBottom})`).attr("class", "x-axis");
  d3.select(svg).append("g").attr("transform", `translate(${marginLeft}, 0)`).attr("class", "y-axis");

  // Redraw the axes
  d3.select('.x-axis').call(d3.axisBottom(x).ticks(width / 80));
  d3.select('.y-axis').call(d3.axisLeft(y));

  // Filter the data based on the current selection and redraw the lines
  const filtered = data.filter(d => selected.includes(d.state));
  const dataByStates = d3.group(filtered, d => d.state);

  d3.select(svg).append('g')
    .attr('class', 'grid')
    .attr('transform', `translate(${marginLeft},0)`)
    .call(d3.axisLeft(y)
      .tickSize(-width + marginLeft + marginRight)
      .tickFormat(''))
    .selectAll('.tick line')
    .style('stroke', '#eee')
    .style('stroke-opacity', 0.7);

    
  // Draw lines
  d3.select(svg)
    .selectAll("path")
    .data(Array.from(dataByStates.values()), d => d[0].state)
    .join("path")
      .attr("fill", "none")
      .attr("stroke", "steelblue")
      .attr("stroke-width", 1.5)
      .attr("stroke-linejoin", "round")
      .attr("stroke-linecap", "round")
      .attr("d", line)
      .style("mix-blend-mode", "multiply");
  }

  function handleRemove(event) {
    selected = selected.filter(item => item !== event.detail);
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