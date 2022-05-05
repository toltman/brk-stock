<script>
  import * as d3 from "d3";
  import AxisHorizontal from "./AxisHorizontal.svelte";
  import AxisVertical from "./AxisVertical.svelte";
  import Tooltip from "./Tooltip.svelte";
  import { inview } from "svelte-inview";

  // access data
  let data = [];
  d3.csv(
    "https://raw.githubusercontent.com/toltman/brk-stock/main/data/Berkshire%20Hathaway%20vs%20SP500.csv"
  ).then((res) => {
    data = res;
  });

  const xAccessor = (d) => new Date(d.year);
  const brkAccessor = (d) => parseFloat(d.brk);
  const spAccessor = (d) => parseFloat(d.sp500);

  // create chart dimensions
  let margin = {
    left: 70,
    top: 10,
    right: 0,
    bottom: 20,
  };
  let width = 400;
  let height = 300;
  $: boundsWidth = width - margin.left - margin.right;
  $: boundsHeight = height - margin.top - margin.bottom;

  // create scales
  let specifiedXRange = null;
  let specifiedXDomain = null;
  $: xScale = d3
    .scaleTime()
    .domain(specifiedXDomain || d3.extent(data.slice(0, 5), xAccessor))
    .range(specifiedXRange || [0, boundsWidth]);

  let specifiedYDomain = null;
  $: yScale = d3
    .scaleLinear()
    .domain(specifiedYDomain || [0, d3.max(data.slice(0, 5), brkAccessor)])
    .range([boundsHeight, 0])
    .nice(true);

  // draw data
  $: lineD = d3
    .line()
    .x((d) => xScale(xAccessor(d)))
    .y((d) => yScale(brkAccessor(d)))(data);

  $: lineSP500 = d3
    .line()
    .x((d) => xScale(xAccessor(d)))
    .y((d) => yScale(spAccessor(d)))(data);

  // set up interactions
  let hoveredPoint = null;
  $: quadtree = d3
    .quadtree()
    .x((d) => xScale(xAccessor(d)))
    .y((d) => yScale(brkAccessor(d)))
    .addAll(data);

  // update chart based on new data
  function updateChart(data) {
    if (data.length) {
      specifiedXDomain = [xAccessor(data[0]), xAccessor(data[data.length - 1])];
      specifiedYDomain = [
        0,
        Math.max(d3.max(data, brkAccessor), d3.max(data, spAccessor)),
      ];
    }
  }
</script>

<div class="scroll-section" use:inview on:enter={updateChart(data.slice(0, 5))}>
  Getting started
</div>

<div
  class="scroll-section"
  use:inview
  on:enter={updateChart(data.slice(0, 15))}
>
  The 70's
</div>

<div
  class="scroll-section"
  use:inview
  on:enter={updateChart(data.slice(0, 25))}
>
  The 80's
</div>

<div
  class="scroll-section"
  use:inview
  on:enter={updateChart(data.slice(0, 35))}
>
  The 90's
</div>

<div
  class="scroll-section"
  use:inview
  on:enter={updateChart(data.slice(0, 45))}
>
  The 2000's
</div>

<div class="scroll-section" use:inview on:enter={updateChart(data)}>
  The 2010's
</div>

<div class="fixed">
  <h2>
    <span style="font-size: 1.5em">B</span>ERKSHIRE
    <span style="font-size: 1.5em">H</span>ATHAWAY
  </h2>

  <figure>
    <div class="wrapper" bind:clientWidth={width} bind:clientHeight={height}>
      <svg {width} {height}>
        <g transform="translate({margin.left}, {margin.top})">
          <path
            d={lineD}
            fill="none"
            stroke="sienna"
            stroke-width="5"
            style="transition: 3s"
          />
          <path
            d={lineSP500}
            fill="none"
            stroke="green"
            stroke-width="5"
            style="transition: 3s"
          />
          {#each data as d}
            <circle
              cx={xScale(xAccessor(d))}
              cy={yScale(brkAccessor(d))}
              r="5"
              style="
                transition: all 3s;
              "
              fill={hoveredPoint === d ? "skyblue" : "sienna"}
              stroke={hoveredPoint === d ? "black" : "none"}
              visibility="hidden"
            />
          {/each}

          <rect
            width={boundsWidth}
            height={boundsHeight}
            fill="transparent"
            on:mousemove={(e) => {
              const pos = d3.pointer(e);
              const x = pos[0];
              const y = pos[1];
              const closestPoint = quadtree.find(x, y);
              if (!closestPoint) return;
              const hoveredPointPosition = [
                xScale(xAccessor(closestPoint)),
                yScale(brkAccessor(closestPoint)),
              ];
              // don't highlight if too far away
              // a^2 + b^2 = c^2
              const distance = Math.sqrt(
                (x - hoveredPointPosition[0]) ** 2 +
                  (y - hoveredPointPosition[1]) ** 2
              );
              if (distance < 50) {
                hoveredPoint = closestPoint;
              } else {
                hoveredPoint = null;
              }
            }}
          />
        </g>
        <g transform="translate({margin.left}, {boundsHeight + margin.top})">
          <AxisHorizontal
            scale={xScale}
            count="5"
            format={d3.timeFormat("%Y")}
          />
        </g>
        <g transform="translate({margin.left}, {margin.top})">
          <AxisVertical count="5" scale={yScale} format={(d) => "$" + d} />
        </g>
      </svg>

      <!-- {#if hoveredPoint}
        <Tooltip
          x={margin.left + xScale(xAccessor(hoveredPoint))}
          y={margin.top + yScale(brkAccessor(hoveredPoint))}
          placement={[
            xScale(xAccessor(hoveredPoint)) > boundsWidth - 50 ? -90 : -50,
            yScale(brkAccessor(hoveredPoint)) < 80 ? 0 : -100,
          ]}
        >
          <strong>
            {d3.timeFormat("%Y")(xAccessor(hoveredPoint))}
          </strong>
          <div>
            ${d3.format(",.0f")(hoveredPoint.brk)}
          </div>
        </Tooltip>
      {/if} -->
    </div>
  </figure>
</div>

<style>
  .fixed {
    position: fixed;
    top: 25%;
    right: 0;
    bottom: 0;
    left: 0;
    z-index: 1;
  }
  .scroll-section {
    position: relative;
    background: white;
    margin-top: 90vh;
    margin-right: auto;
    margin-bottom: 100vh;
    margin-left: auto;
    border: 1px solid;
    width: 10em;
    padding: 0.5em 0.6em;
    z-index: 10;
  }

  .wrapper {
    position: relative;
    margin: auto;
    font-family: sans-serif;
    width: 100%;
    height: 300px;
    max-width: 600px;
    min-width: 0;
  }

  svg {
    overflow: hidden;
  }

  h2 {
    text-align: center;
    font-family: "Times New Roman", Times, serif;
    color: rgb(0, 0, 128);
  }
</style>
