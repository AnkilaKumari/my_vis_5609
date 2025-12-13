<script lang="ts">
  import { onMount } from "svelte";
  import * as d3 from "d3";
  import type { TMovie } from "../../types";
  import StoryOpen from "./StoryOpen.svelte";
  import Scrolly2D from "./Scrolly2D.svelte";
  import BarChart from "../../lib/BarChart.svelte";

  let movies: TMovie[] = [];

  async function loadCsv() {
    try {
      const csvUrl = "/data/summer_movies.csv";   // ✅ FIXED
      movies = await d3.csv(csvUrl, (d: any) => ({
        tconst: d.tconst,
        title_type: d.title_type,
        primary_title: d.primary_title,
        original_title: d.original_title,
        year: d.year ? new Date(+d.year, 0, 1) : null,
        runtime_minutes: d.runtime_minutes ? +d.runtime_minutes : null,
        genres: d.genres ? d.genres.split(",") : [],   // ✅ FIXED array
        simple_title: d.simple_title,
        average_rating: d.average_rating ? +d.average_rating : null,
        num_votes: d.num_votes ? +d.num_votes : null
      }));

      console.log("CSV loaded:", movies.length);
    } catch (e) {
      console.error("CSV load error:", e);
    }
  }

  onMount(loadCsv);
</script>

<div class="container">
  <StoryOpen movieNum={movies.length} />
  <Scrolly2D {movies} />
  <BarChart {movies} />
</div>

<style>
  .container {
    width: 80vw;
    margin: 10px auto;
  }
</style>
