<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';
  import { nest } from 'd3-collection';

  let accidentsData;
  let filteredData;
  const svgWidth = 800;
  const svgHeight = 400;
  const margin = { top: 20, right: 30, bottom: 50, left: 60 };
  let startYear = null;
  let endYear = null;

  const filterDataByYear = () => {
    filteredData = accidentsData.filter(d => {
      const year = new Date(d.date_time).getFullYear();
      return year >= startYear && year <= endYear;
    });
    drawChart(filteredData);
    drawPieChart(filteredData);
  }

  onMount(async () => {
    const response = await fetch('pd_collisions_details_datasd.csv');
    const accidentsCsv = await response.text();
    accidentsData = d3.csvParse(accidentsCsv, d3.autoType);
    startYear = d3.min(accidentsData, d => new Date(d.date_time).getFullYear());
    endYear = d3.max(accidentsData, d => new Date(d.date_time).getFullYear());
    filterDataByYear();
  });

  const drawChart = (data) => {
    const nestedData = nest()
    .key(d => d.veh_make || "Unknown") // Handle missing values for veh_make
    .key(d => new Date(d.date_time).getFullYear())
    .rollup(v => d3.sum(v, d => d.injured + d.killed))
    .entries(data);

  nestedData.sort((a, b) => {
    const sumA = d3.sum(a.values, d => d.value);
    const sumB = d3.sum(b.values, d => d.value);
    return sumB - sumA;
  });

  const topCategories = nestedData.slice(0, 10);

  // Filter out top categories
  const otherData = nestedData.slice(10);

  // Aggregate the rest into the "Other" category, year by year
  const otherAggregatedData = otherData.reduce((acc, cur) => {
    cur.values.forEach(d => {
      const existingYear = acc.find(yearData => yearData.key === d.key);
      if (existingYear) {
        existingYear.value += d.value;
      } else {
        acc.push({ key: d.key, value: d.value });
      }
    });
    return acc;
  }, []);

  otherAggregatedData.sort((a, b) => a.key - b.key); // Ensure the data is sorted by year

  // Create aggregated data array
  let aggregatedData = [...topCategories, { key: "Other", values: otherAggregatedData }];

    
  const xScale = d3.scaleTime()
    .domain([new Date(startYear, 0), new Date(endYear, 11)])
    .range([margin.left, svgWidth - margin.right]);

  const yScale = d3.scaleLinear()
    .domain([0, d3.max(aggregatedData.flatMap(d => d.values), d => d.value)])
    .nice()
    .range([svgHeight - margin.bottom, margin.top]);

  const line = d3.line()
    .x(d => xScale(new Date(d.key, 0)))
    .y(d => yScale(d.value));

  const color = d3.scaleOrdinal()
    .domain(aggregatedData.map(d => d.key))
    .range(d3.schemePaired);

  const svg = d3.select("#line-chart")
    .attr("width", svgWidth)
    .attr("height", svgHeight);

  svg.selectAll(".line")
    .data(aggregatedData)
    .enter()
    .append("path")
    .attr("class", "line")
    .attr("fill", "none")
    .attr("stroke", d => color(d.key))
    .attr("stroke-width", 2)
    .attr("d", d => line(d.values));

  // Add axes
  svg.append("g")
    .attr("transform", `translate(0, ${svgHeight - margin.bottom})`)
    .call(d3.axisBottom(xScale).tickFormat(d3.timeFormat("%Y")));

  svg.append("g")
    .attr("transform", `translate(${margin.left}, 0)`)
    .call(d3.axisLeft(yScale));

    const tooltip = d3.select("body")
      .append("div")
      .attr("class", "tooltip")
      .style("opacity", 0);
  }

  const drawPieChart = (data) => {
    const groupedData = d3.rollup(
        data,
        v => v.length,
        d => d.veh_make ?? "Unknown"
    );

    const pieData = Array.from(groupedData, ([category, count]) => ({ category, count }))
        .sort((a, b) => b.count - a.count);

    const topCategories = pieData.slice(0, 10);
    const otherCount = pieData.slice(10).reduce((acc, cur) => acc + cur.count, 0);
    const aggregatedData = [...topCategories, { category: "Other", count: otherCount }];

    // Increased SVG dimensions for more space
    const width = 600;
    const height = 600;
    const radius = Math.min(width, height) / 2.7 - 40; // Added margin for labels

    const color = d3.scaleOrdinal()
        .domain(aggregatedData.map(d => d.category))
        .range(d3.schemePaired);

    const pie = d3.pie()
        .value(d => d.count)
        .sort(null);

    const arc = d3.arc()
        .innerRadius(0)
        .outerRadius(radius);

    const svg = d3.select("#pie-chart")
        .attr("width", width)
        .attr("height", height)
        .append("g")
        .attr("transform", `translate(${width / 2}, ${height / 2})`);

    const arcs = svg.selectAll("arc")
        .data(pie(aggregatedData))
        .enter()
        .append("g")
        .attr("class", "arc");

    arcs.append("path")
        .attr("d", arc)
        .attr("fill", d => color(d.data.category));

    // Enhanced label positioning
    arcs.append("text")
        .attr("transform", d => {
            const c = arc.centroid(d);
            const x = c[0];
            const y = c[1];
            const h = Math.sqrt(x*x + y*y); // Pythagorean theorem for hypotenuse
            return `translate(${(x/h * (radius + 10))}, ${(y/h * (radius + 10))})`; // Move labels outside arc
        })
        .attr("dy", ".35em")
        .style("text-anchor", d => (d.endAngle + d.startAngle) / 2 > Math.PI ? "end" : "start")
        .text(d => `${d.data.category}: ${d.data.count}`);
    
        arcs
        .on("mouseenter", handleMouseOver)
        .on("mouseleave", handleMouseOut);

  function handleMouseOver(event, d) {
      d3.select(this).style("opacity", 0.7);

      const hoveredCategory = d.data.category;
      const lines = d3.select("#line-chart").selectAll(".line");

      if (hoveredCategory !== "Other") {
        // Apply gray color to all lines except the hovered category
        lines.each(function(lineData) {
          if (lineData.key !== hoveredCategory) {
            d3.select(this).attr("stroke", "#ccc").attr("stroke-width", 2);
          } else {
            // Highlight the hovered category line
            d3.select(this).attr("original-stroke", d3.select(this).attr("stroke")).attr("stroke-width", 6);
          }
        });
      } else {
        // When hovering over "Other", highlight only the "Other" line
        lines.each(function(lineData) {
          if (lineData.key === "Other") {
            d3.select(this).attr("original-stroke", d3.select(this).attr("stroke")).attr("stroke-width", 6);
          } else {
            d3.select(this).attr("stroke", "#ccc").attr("stroke-width", 2);
          }
        });
      }
    }


    function handleMouseOut() {
      d3.select(this).style("opacity", 1);

      // Assuming each line has data bound to it that contains a 'key' property
      d3.select("#line-chart").selectAll(".line")
        .attr("stroke", d => color(d.key)) // Use the color scale to set the stroke based on the key
        .attr("stroke-width", 2);
    }

  }
</script>

<div>
  <label>Start Year:
    <input type="number" bind:value={startYear} on:input={filterDataByYear}>
  </label>
  <label>End Year:
    <input type="number" bind:value={endYear} on:input={filterDataByYear}>
  </label>
</div>

<svg id="line-chart"></svg>
<svg id="pie-chart"></svg>

<style>
  .tooltip {
    position: absolute;
    background-color: white;
    border: 1px solid #ddd;
    border-radius: 4px;
    padding: 6px;
    font-size: 12px;
    pointer-events: none;
  }
</style>
