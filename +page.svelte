<script lang="ts">
  import * as d3 from "d3";
  import { onMount } from "svelte";
  import type { TMovie } from "../../../types";

  let svg;
  let movies: TMovie[] = [];
  const width = 800;
  const height = 500;
  const margin = { top: 50, right: 40, bottom: 60, left: 60 };

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
    drawChart();
  }

  function drawChart() {
    const svgEl = d3.select(svg);
    svgEl.selectAll("*").remove();

    // --- Count number of movies per genre per year ---
    const grouped = d3.rollups(
      movies.flatMap((m) =>
        (m.genres ? m.genres.split(",") : []).map((g) => ({ year: m.year, genre: g.trim() }))
      ),
      (v) => v.length,
      (d) => d.year,
      (d) => d.genre
    );

    // Convert nested structure to flat array
    const flatData = grouped.flatMap(([year, map]) =>
      Array.from(map, ([genre, count]) => ({ year, genre, count }))
    );

    // Get all genres and years
    const allGenres = Array.from(new Set(flatData.map((d) => d.genre)));
    const allYears = Array.from(new Set(flatData.map((d) => d.year))).sort();

    // Limit to top 5 genres overall for visibility
    const topGenres = Array.from(
      d3
        .rollups(flatData, (v) => d3.sum(v, (d) => d.count), (d) => d.genre)
        .sort((a, b) => b[1] - a[1])
        .slice(0, 5),
      ([genre]) => genre
    );

    const data = d3.group(flatData.filter((d) => topGenres.includes(d.genre)), (d) => d.genre);

    // --- Create scales ---
    const x = d3
      .scaleLinear()
      .domain(d3.extent(allYears) as [number, number])
      .range([margin.left, width - margin.right]);

    const y = d3
      .scaleLinear()
      .domain([0, d3.max(flatData, (d) => d.count) || 0])
      .nice()
      .range([height - margin.bottom, margin.top]);

    const color = d3.scaleOrdinal(d3.schemeSet2).domain(topGenres);

    // --- Draw axes ---
    svgEl
      .append("g")
      .attr("transform", `translate(0,${height - margin.bottom})`)
      .call(d3.axisBottom(x).tickFormat(d3.format("d")));

    svgEl
      .append("g")
      .attr("transform", `translate(${margin.left},0)`)
      .call(d3.axisLeft(y));

    // --- Draw lines ---
    const line = d3
      .line<{ year: number; count: number }>()
      .x((d) => x(d.year))
      .y((d) => y(d.count));

    for (const [genre, values] of data.entries()) {
      svgEl
        .append("path")
        .datum(values)
        .attr("fill", "none")
        .attr("stroke", color(genre))
        .attr("stroke-width", 2)
        .attr("d", line)
        .append("title")
        .text(genre);
    }

    // --- Add title ---
    svgEl
      .append("text")
      .attr("x", width / 2)
      .attr("y", 30)
      .attr("text-anchor", "middle")
      .style("font-size", "18px")
      .text("Top 5 Genres Over Time");
  }

  onMount(loadCsv);
</script>

<svg bind:this={svg} width={800} height={500}></svg>
