<script lang="ts">
  import { Scroll, Bar, RankBar } from "$lib";
  import type { TMovie } from "../../types";
  import * as d3 from "d3";

  let { movies } = $props();
  let myProgress = $state(0);

  let yearRange = $derived(d3.extent(movies.map(d => d.year)));

  function yearList(start?: Date, end?: Date) {
    if (!start || !end) return [];
    return Array.from(
      { length: end.getFullYear() - start.getFullYear() + 1 },
      (_, i) => new Date(start.getFullYear() + i, 0, 1)
    );
  }

  let years = $derived(yearList(yearRange[0], yearRange[1]));
</script>

<h2>2D Bar Chart Scroll Demo</h2>
<p>Scroll to change the year limit. Progress: {Math.round(myProgress)}%</p>

<Scroll bind:progress={myProgress}>
  {#each years as date}
    <div class={date.getFullYear().toString()}>
      <h3 class="year">{date.getFullYear()}</h3>

      {#each movies.filter(m => m.year.getFullYear() === date.getFullYear()) as movie}
        <b class="movie-title">{movie.primary_title}</b> — {movie.genres.join(" | ")} <br>
      {/each}
    </div>
  {/each}

  <svelte:fragment slot="viz">
    <Bar {movies} progress={myProgress} />

    <div style="height:16px;"></div>
    <RankBar {movies} progress={myProgress} width={650} height={450} />
  </svelte:fragment>
</Scroll>

<style>
  .movie-title { color: #449900; }
  h3.year { margin-bottom: 0; }
</style>
