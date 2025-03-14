<script>
  /*
    Imports
  */
  import "./app.css"
  import { onDestroy, onMount } from "svelte"
  import * as d3 from 'd3'
  import Particles from "./lib/Particles.svelte"
  import Meteors from "./lib/Metors.svelte"

  /*
    Non-constant Variables
  */
  let counters = { red: 0, green: 0, blue: 0, purple: 0 }
  let margin = { top: 50, right: 120, bottom: 50, left: 150 }
  let data = []
  let event_source_url = 'http://localhost:8080/updates'
  let width = 800
  let height = 500
  let delay = 200
  let svg
  let event_source

  /*
    Constant Variables
  */
  const gradients = {
    red: { start: '#ff6b6b', end: '#ff4757' },
    green: { start: '#4CAF50', end: '#388E3C' },
    blue: { start: '#2196F3', end: '#1976D2' },
    purple: { start: '#9C27B0', end: '#7B1FA2' }
  };
  const color_order = ["red", "green", "blue", "purple"]
  const chartWidth = width - margin.left - margin.right
  const chartHeight = height - margin.top - margin.bottom

  /*
    Counter Functions
  */
  async function increment(color) {
    try {
      await fetch(`http://localhost:8080/increment/${color}`, { method: "POST" })
    } catch (error) {
      console.error("Error - (Svelte)increment - Failed to Increment:", error)
    }
  }

  /*
    Bar Graph Functions
  */
  function chart_init() {
    svg = d3.select("#chart")
      .append("svg")
      .attr("viewBox", [0, 0, width, height])
      .attr("width", width)
      .attr("height", height)
      .attr("style", "max-width: 100%; height: auto;")
    color_order.forEach(color => {
      const gradient = (svg.append("defs")).append("linearGradient")
        .attr("id", `gradient-${color}`)
        .attr("x1", "0%")
        .attr("y1", "0%")
        .attr("x2", "100%")
        .attr("y2", "0%");
      gradient.append("stop")
        .attr("offset", "0%")
        .attr("stop-color", gradients[color].start);
      gradient.append("stop")
        .attr("offset", "100%")
        .attr("stop-color", gradients[color].end);
    });
    start_live()
  }

  function start_live() {
    try {
      event_source = new EventSource(event_source_url)
      event_source.onmessage = (e) => {
        try {
          counters = JSON.parse(e.data)
          data = Object.entries(counters).map(([color, count]) => (
            {
             color, 
             count, 
            }))
            update_chart()
        } catch (parseError) {
          console.error("Error - (Svelte)updates - Failed to Parse Updates:", parseError)
        }
      }
      event_source.onerror = (error) => {
        console.error("Error - (Svelte)updates - Failed to Fetch Updates:", error)
      }
      console.log("EventSource connection established")
    } catch (error) {
      console.error("Error - (Svelte)updates - Failed to Fetch Updates:", error)
    }
  }

  function update_chart() {
    data.sort((a, b) => b.count - a.count)
    const xScale = d3.scaleLinear()
      .domain([0, d3.max(data, d => d.count) * 1.1])
      .range([0, chartWidth])
    const yScale = d3.scaleBand()
      .domain(data.map(d => d.color))
      .range([0, chartHeight])
      .padding(0.1)
    const bars = svg.selectAll(".bar")
      .data(data, d => d.color)
    bars.exit()
      .transition()
      .duration(delay)
      .attr("width", 0)
      .remove()
    const newBars = bars.enter()
      .append("g")
      .attr("class", "bar")
      .attr("transform", d => `translate(${margin.left},${margin.top + yScale(d.color)})`)
    newBars.append("rect")
      .attr("height", yScale.bandwidth())
      .attr("width", 0)
      .attr("fill", d => `url(#gradient-${d.color})`)
      .transition()
      .duration(delay)
      .attr("width", d => xScale(d.count));
    newBars.append("text")
      .attr("class", "value-label")
      .attr("x", d => xScale(d.count) + 5)
      .attr("y", yScale.bandwidth() / 2)
      .attr("dy", "0.35em")
      .style("font-size", "12px")
      .text(d => d.count.toLocaleString())
    bars.transition()
      .duration(delay)
      .attr("transform", d => `translate(${margin.left},${margin.top + yScale(d.color)})`)
    bars.select("rect")
      .transition()
      .duration(delay)
      .attr("width", d => xScale(d.count))
    bars.select(".value-label")
      .transition()
      .duration(delay)
      .attr("x", d => xScale(d.count) + 5)
      .text(d => d.count.toLocaleString())
  }

  /*
    Startup
  */
  onMount(async () => {
    try {
      const response = await fetch("http://localhost:8080/counters")
      counters = await response.json()
      data = Object.entries(counters).map(([color, count]) => ({
        category: color,
        value: count,
      }))
      chart_init()
    } catch (error) {
      console.error("Error - (Svelte)onMount - Failed to Fetch Counters:", error)
    }
  })

  /*
    Clean Up
  */
  onDestroy(() => {
    if (event_source) {
      event_source.close()
    }
  })
</script>

<main>
  <div style="background-color: black; width: 100vw; height: 100vh; position: fixed; top: 0; left: 0; overflow: hidden;">
    <Particles className="absolute inset-0 z-[-1]" refresh={true} quantity={1000}/>
    <Meteors number={30} />
    <div class="chart-container">
      <div id="chart"></div>
    </div>
    <div class="buttons">
      {#each color_order.map(color => [color, counters[color]]) as [color, count]}
        <div class="button-wrapper">
          <div class="button-background" style="background-color: #E8E9EB; border-color: #424342;"></div>
          <button 
            style="border-color: #424342; color: {gradients[color]['start']}; background-color: {gradients[color]['start']};"
            on:click={() => increment(color)}
          >
            {""}
          </button>
        </div>
      {/each}
    </div>
  </div>
</main>

<style>
  .chart-container {
    width: 100%;
    height: 100%;
    display: flex;
    justify-content: center;
    align-items: center;
  }

  .buttons {
    position: fixed;
    bottom: 2rem;
    left: 50%;
    transform: translateX(-50%);
    display: flex;
    gap: 1rem;
    padding: 0;
  }

  .button-wrapper {
    position: relative;
    margin: 0 0.5rem;
  }

  .button-background {
    position: absolute;
    top: 2px;
    left: 0;
    right: 0;
    height: calc(100% + 3px);
    border: 1.5px solid black;
    border-radius: 0.35rem;
    z-index: -1;
  }
  
  button {
    padding-left: 1rem;
    font-size: 0.75rem;
    border: 1.5px solid black;
    border-radius: 0.35rem;
    font-weight: bold;
    text-transform: capitalize;
    background-color: #fff;
    width: 4rem;
    height: 3em;
  }

  button:hover {
    transform: translateY(-2px);
  }

  button:focus {
    outline: none;
  }

  button:active {
    transform: translateY(1px);
  }
  
  :global(.bar:hover rect) {
    opacity: 0.8;
  }

</style>
