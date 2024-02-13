<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';
  import App from './App.svelte';

  export let data;

  const width = 928;
  const height = 600;
  const marginTop = 20;
  const marginRight = 30;
  const marginBottom = 30;
  const marginLeft = 60;
  
  
  
  let states = [];
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

  $: dataByStates = d3.group(
      data, 
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
  $: points = data.map((d) => [x(d.date), y(d.permit), d.state]);

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
    d3.select(svg).property("value", data[i]).dispatch("input", {bubbles: true});
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

  function onSubmit(e) {
    const formData = new FormData(e.target);

    states = [];
    for (let field of formData) {
      const [key, value] = field;
      states.push(value);
    }
  }
</script>

<style>
  .state-checkboxes {
    column-count: 2;
    column-gap: 20px; /* Adjust the gap to your liking */
  }
</style>

<h1>Permits Per State Over Time</h1>
<div style="display: flex">
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
<form on:submit|preventDefault={onSubmit} class="state-checkboxes">
  {#each dataByStates.keys() as key}
  <input type="checkbox" id={key} name={key} value={key} checked>
  <label for=key>{key}</label><br>
  {/each}
  <input type="submit" value="Submit">
</form>
</div>