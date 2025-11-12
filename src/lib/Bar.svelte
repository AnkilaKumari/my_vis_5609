<script lang="ts">
  import * as d3 from "d3";
  import { onMount } from "svelte";
  import type { TMovie } from "../types";

  export let movies: TMovie[] = [];

  let svg;
  const width = 800;
  const height = 500;
  const margin = { top: 40, right: 30, bottom: 90, left: 60 };

  <g
  class={`${genre} genre`}
  transform={`translate(${usableArea.left}, ${yScale(genre)})`}
  transition:fade
>


  onMount(() => {
    if (movies.length === 0) return;

    const genreCounts = d3.rollups(
      movies.flatMap((d) => (d.genres ? d.genres.split(",") : [])),
      (v) => v.length,
      (genre) => genre.trim()
    );

    const data = genreCounts
      .map(([genre, count]) => ({ genre, count }))
      .sort((a, b) => b.count - a.count);

    const x = d3
      .scaleBand()
      .domain(data.map((d) => d.genre))
      .range([margin.left, width - margin.right])
      .padding(0.2);

    const y = d3
      .scaleLinear()
      .domain([0, d3.max(data, (d) => d.count) || 0])
      .nice()
      .range([height - margin.bottom, margin.top]);

    const svgEl = d3.select(svg);
    svgEl.selectAll("*").remove();

    svgEl
      .selectAll("rect")
      .data(data)
      .join("rect")
      .attr("x", (d) => x(d.genre)!)
      .attr("y", (d) => y(d.count))
      .attr("width", x.bandwidth())
      .attr("height", (d) => y(0) - y(d.count))
      .attr("fill", "#69b3a2")
      .on("mouseover", function (event, d) {
        d3.select(this).attr("fill", "#2171b5");
      })
      .on("mouseout", function (event, d) {
        d3.select(this).attr("fill", "#69b3a2");
      })
      .append("title")
      .text((d) => `${d.genre}: ${d.count}`);

    svgEl
      .append("g")
      .attr("transform", `translate(0,${height - margin.bottom})`)
      .call(d3.axisBottom(x))
      .selectAll("text")
      .attr("transform", "rotate(-45)")
      .style("text-anchor", "end");
      .genre {
  transition: transform 0.8s ease;
}
.bar {
  transition: width 0.8s ease, fill 0.3s ease;
}


    svgEl
      .append("g")
      .attr("transform", `translate(${margin.left},0)`)
      .call(d3.axisLeft(y));

    svgEl
      .append("text")
      .attr("x", width / 2)
      .attr("y", 30)
      .attr("text-anchor", "middle")
      .style("font-size", "18px")
      .genre {
  transition: transform 0.8s ease;
}
.bar {
  transition: width 0.8s ease, fill 0.3s ease;
}

      .text("Genre Distribution of Summer Movies");
  });
</script>

<svg bind:this={svg} width={width} height={height}></svg>
