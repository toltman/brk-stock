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
    left: 30,
    top: 0,
    right: 0,
    bottom: 0,
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
    .domain(specifiedXDomain || d3.extent(data, xAccessor))
    .range(specifiedXRange || [0, boundsWidth]);

  let specifiedYDomain = null;
  $: yScale = d3
    .scaleLinear()
    .domain(
      specifiedYDomain || [d3.min(data, spAccessor), d3.max(data, brkAccessor)]
    )
    .range([boundsHeight, 0])
    .nice(true);

  // draw data
  $: lineD = d3
    .line()
    .x((d) => xScale(xAccessor(d)))
    .y((d) => yScale(brkAccessor(d)))(data);

  // set up interactions
  let hoveredPoint = null;
  $: quadtree = d3
    .quadtree()
    .x((d) => xScale(xAccessor(d)))
    .y((d) => yScale(brkAccessor(d)))
    .addAll(data);
</script>

<div
  class="scroll-section"
  use:inview
  on:enter={(event) => {
    // show the min_temp property on the y axis
  }}
>
  Minimum temperature {d3.min(data, brkAccessor)}
</div>

<div
  class="scroll-section"
  use:inview
  on:enter={(event) => {
    // show the max_temp property on the y axis
  }}
>
  Maximum temperature: {d3.max(data, brkAccessor)}
</div>

<div
  class="scroll-section"
  use:inview
  on:enter={(event) => {
    // set the tooltip to show the first data point
    hoveredPoint = data[d3.minIndex(data, xAccessor)];
  }}
>
  First data point
</div>

<div
  class="scroll-section"
  use:inview
  on:enter={(event) => {
    // make the bounds 2000px wide
    // hint: you might need to make a new variable within the <script/>
    // hint: you also have access to a "leave" event
    let subset = data.slice(0, 10);
    specifiedXDomain = [
      xAccessor(subset[0]),
      xAccessor(subset[subset.length - 1]),
    ];
    specifiedYDomain = [
      0,
      Math.max(d3.max(subset, brkAccessor), d3.max(subset, spAccessor)),
    ];
  }}
>
  Zoom in on the start of the data
</div>

<div class="fixed">
  <h2>Berkshire Hathawway vs the Market</h2>

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
        <g transform="translate({margin.left}, 0)">
          <AxisVertical count="5" scale={yScale} />
        </g>
      </svg>
      <div
        class="label"
        style="transform: translate({margin.left + 6}px, -10px)"
      >
        BRK
      </div>

      {#if hoveredPoint}
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
      {/if}
    </div>
  </figure>
</div>

0

<style>
  .fixed {
    position: fixed;
    top: 0;
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
    overflow: visible;
  }

  h2 {
    text-align: center;
    font-family: sans-serif;
  }

  .label {
    position: absolute;
    top: 0;
    left: 0;
    max-width: 4em;
    font-size: 0.6em;
  }
</style>
