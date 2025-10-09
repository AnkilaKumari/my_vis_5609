
<script lang="ts">
  import * as d3 from "d3";
  import { onMount } from "svelte";
  import type { TMovie } from "../../types";
  import Bar from "$lib/Bar.svelte"; 


  // Reactive variable for storing the data
  let movies: TMovie[] = [];

  // Function to load the CSV
  async function loadCsv() {
    try {
      const csvUrl = "./summer_movies.csv";
      // @igno
      movies = await d3.csv(csvUrl, (row) => {
        // TIP: in row, all values are strings, so we need to use a row conversion function here to format them
        return {
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
        };
      });
      
      console.log("Loaded CSV Data:", movies);
    } catch (error) {
      console.error("Error loading CSV:", error);
    }
  }
  // Call the loader when the component mounts
  onMount(loadCsv);
</script>

<h1>Summer Movies</h1>

<p>Here are {movies.length == 0 ? "..." : movies.length + " "} movies</p>

 {#if movies.length > 0}
  <Bar {movies} />
{/if}