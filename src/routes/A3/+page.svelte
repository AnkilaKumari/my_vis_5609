<script>
  import { Scroll } from "$lib";
  import Bar from "$lib/Bar.svelte";
  import { onMount } from "svelte";
  import * as d3 from "d3";
  import type { TMovie } from "../../../types";

  let movies: TMovie[] = [];
  let myProgress = $state(0);

  async function loadCsv() {
    movies = await d3.csv("./summer_movies.csv", d => ({
      ...d,
      year: +d.year,
      runtime_minutes: +d.runtime_minutes,
      average_rating: +d.average_rating,
      num_votes: +d.num_votes,
      genres: d.genres.split(","),
    }));
  }

  onMount(loadCsv);
</script>

<Scroll bind:progress={myProgress}>
  <div>
    <h2>Genre changes over time</h2>
    <p>Scroll down to see how movie genres rise and fall through the years.</p>
  </div>

  <svelte:fragment slot="viz">
    <Bar {movies} progress={myProgress} />
  </svelte:fragment>
</Scroll>
