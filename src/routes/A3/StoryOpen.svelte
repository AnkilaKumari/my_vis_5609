<script lang="ts">
  import { Scroll } from "$lib";
  import { slide, fly } from "svelte/transition";

  let { movieNum } = $props();
  let progress = $state(0);
</script>

<Scroll bind:progress --scrolly-story-width="0">
  <div id="virtual"></div>

  <div slot="viz" class="header">
    <h1>Summer Movies</h1>

    {#if progress > 30}
      <p in:slide={{ duration: 800, axis: "x" }}>
        A Deep Dive into {movieNum} Movies Released Between 1945 and 2023
      </p>
    {/if}

    {#if progress > 70}
      <p in:fly={{ duration: 800, y: 200 }}>
        with data visualizations
      </p>
    {/if}
  </div>
</Scroll>

<style>
  .header {
    background-color: #ffe4c1;
    padding: 80px 60px;
    height: 60vh;
    width: 800px;
  }
  #virtual {
    height: 150vh;
  }
  h1 { font-size: 10vh; color: #433417; }
  p { font-size: 3vh; color: #555; }
</style>
