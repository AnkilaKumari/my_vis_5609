<script lang="ts">
  import * as d3 from "d3";
  import { onMount } from "svelte";
  import type { TMovie } from "../../../types";

  let svg;
  let movies: TMovie[] = [];
  const width = 700;
  const height = 700;
  const margin = { top: 100, right: 20, bottom: 20, left: 100 };

  async function loadCsv() {
    const csvUrl = "/summer_movies.csv";
    const data = await d3.csv(csvUrl, (row: d3.DSVRowString<string>): TMovie => ({
      tconst: row.tconst ?? "",
      title_type: row.title_type ?? "",
      primary_title: row.primary_title ?? "",
      original_title: row.original_title ?? "",
      year: row.year ? +row.year : 0,
      runtime_minutes: row.runtime_minutes ? +row.runtime_minutes : 0,
      genres: row.genres ?? "",
      simple_title: row.simple_title ?? "",
      average_rating: row.average_rating ? +row.average_rating : 0,
      num_votes: row.num_votes ? +row.num_votes : 0,
    }));
    movies = data;
    drawHeatmap();
  }

  function drawHeatmap() {
    const svgEl = d3.select(svg);
    svgEl.selectAll("*").remove();

    // Extract all genre pairs
    const pairs = [];
    movies.forEach((m) => {
      const genres = m.genres ? m.genres.split(",").map((g) => g.trim()) : [];
      for (let i = 0; i < genres.length; i++) {
        for (let j = i + 1; j < genres.length; j++) {
          pairs.push([genres[i], genres[j]]);
        }
      }
    });

    const counts = d3.rollups(
      pairs,
      (v) => v.length,
      (d) => d[0],
      (d) => d[1]
    );

    const genres = Array.from(new Set(pairs.flat()));
    const data = [];
    for (const [g1, inner] of counts) {
      for (const [g2, count] of inner) {
        data.push({ g1, g2, count });
      }
    }

    const x = d3.scaleBand().domain(genres).range([margin.left, width - margin.right]).padding(0.05);
    const y = d3.scaleBand().domain(genres).range([margin.top, height - margin.bottom]).padding(0.05);
    const color = d3.scaleSequential(d3.interpolateBlues).domain([0, d3.max(data, (d) => d.count) || 1]);

    svgEl
      .selectAll("rect")
      .data(data)
      .join("rect")
      .attr("x", (d) => x(d.g1)!)
      .attr("y", (d) => y(d.g2)!)
      .attr("width", x.bandwidth())
      .attr("height", y.bandwidth())
      .attr("fill", (d) => color(d.count))
      .append("title")
      .text((d) => `${d.g1} + ${d.g2}: ${d.count}`);

    svgEl.append("g").attr("transform", `translate(0,${margin.top})`).call(d3.axisTop(x).tickSize(0)).selectAll("text").attr("transform", "rotate(-45)").style("text-anchor", "start");
    svgEl.append("g").attr("transform", `translate(${margin.left},0)`).call(d3.axisLeft(y).tickSize(0));

    svgEl
      .append("text")
      .attr("x", width / 2)
      .attr("y", 40)
      .attr("text-anchor", "middle")
      .style("font-size", "18px")
      .text("Genre Co-occurrence Heatmap");
  }

  onMount(loadCsv);
</script>

<svg bind:this={svg} width={700} height={700}></svg>
