<script lang="ts">
    import * as d3 from "d3";
    import Scatter from "$lib/Scatter.svelte";
    import Line from "$lib/Line.svelte";
    import { onMount } from "svelte";
    import type { TMovie } from "../../types";

     let movies: TMovie[] = $state([]);
    let yearRange: [Date, Date] | undefined = $state();

    function getYearCountArray(movies: TMovie[]) {
        let yearCount: { [year: number]: number } = {};
        const allYears = [...new Set(movies.map((d) => d.year.getFullYear()))];
        for (let year of allYears) {
            yearCount[year] = movies.filter((d) => d.year.getFullYear() == year).length;
        }
        const yearCountArray = Object.entries(yearCount).map(([year, count]) => ({
            x: new Date(year),
            y: count as number,
        }));
        yearCountArray.sort((a, b) => (a.x < b.x ? -1 : 1));
        return yearCountArray;
    }
    let yearCountArray = $derived(getYearCountArray(movies));

    type TAxisSelection = { x: keyof TMovie; y: keyof TMovie; size: keyof TMovie };
    let axisSelection: TAxisSelection = $state({
        x: "year",
        y: "average_rating",
        size: "num_votes",
    });

    async function loadCsv() {
        try {
            const csvUrl = "./summer_movies.csv";
            movies = await d3.csv(csvUrl, (row) => ({
                ...row,
                num_votes: Number(row.num_votes),
                runtime_minutes: Number(row.runtime_minutes),
                genres: String(row.genres || "").split(","),
                year: new Date(row.year),
                average_rating: Number(row.average_rating),
            })) as unknown as TMovie[];
        } catch (error) {
            console.error("Error loading CSV:", error);
        }
    }
    onMount(loadCsv);

    // dropdown options
    const attrOptionsX = $derived(movies[0] ? Object.keys(movies[0]) : []);
    // remove arrays/long text from Y/Size (you can also filter 'title' fields if present)
    const attrOptionsY = $derived(movies[0] ? Object.keys(movies[0]).filter((d) => d !== "genres") : []);
    const attrOptionsS = $derived(movies[0] ? Object.keys(movies[0]).filter((d) => d !== "genres") : []);

    // ---------- AUTO SIZE LOGIC ----------
    function isBand(attr: keyof TMovie): boolean {
        if (!movies[0]) return false;
        const v = movies[0][attr];
        return typeof v === "string" || typeof v === "object"; // arrays count as categorical
    }

    function uniqueCount(attr: keyof TMovie, rows: TMovie[]) {
        if (!rows.length) return 0;
        if (attr === "genres") {
            // union of all genres
            const set = new Set<string>();
            rows.forEach((m) => (m.genres || []).forEach((g) => set.add(g)));
            return set.size;
        }
        const set = new Set<any>();
        rows.forEach((m) => set.add(m[attr] as any));
        return set.size;
    }

    // base sizes + per-category growth + clamps
    const BASE_W = 600, BASE_H = 500;
    const PER_CAT_W = 28;   // grow x-length ~28 px per category
    const PER_CAT_H = 22;   // grow y-length ~22 px per category
    const MIN_W = 520, MAX_W = 1400;
    const MIN_H = 360, MAX_H = 1000;

    // compute domain sizes for currently visible rows (after brushing)
    const visibleMovies = $derived(
        yearRange
            ? movies.filter((d) => d.year <= yearRange[1] && d.year >= yearRange[0])
            : movies
    );

    const xIsBand = $derived(isBand(axisSelection.x));
    const yIsBand = $derived(isBand(axisSelection.y));
    const xCatCount = $derived(xIsBand ? uniqueCount(axisSelection.x, visibleMovies) : 0);
    const yCatCount = $derived(yIsBand ? uniqueCount(axisSelection.y, visibleMovies) : 0);

    // dynamic width/height (clamped)
    const plotWidth = $derived(
        xIsBand
            ? Math.max(MIN_W, Math.min(MAX_W, BASE_W + Math.max(0, xCatCount - 12) * PER_CAT_W))
            : BASE_W
    );
    const plotHeight = $derived(
        yIsBand
            ? Math.max(MIN_H, Math.min(MAX_H, BASE_H + Math.max(0, yCatCount - 10) * PER_CAT_H))
            : BASE_H
    );
</script>

<div class="container" style="width: {Math.min(plotWidth + 80, 1500)}px;">
    <h1>Summer Movies</h1>
    <p>Here are {movies.length == 0 ? "..." : movies.length + " "} movies</p>

    {#if movies.length > 0}
        <div class="selectors" style="display:flex; gap:12px; align-items:center; flex-wrap: wrap;">
            <label> X Axis:
                <select bind:value={axisSelection.x}>
                    {#each attrOptionsX as key}<option value={key}>{key}</option>{/each}
                </select>
            </label>
            <label> Y Axis:
                <select bind:value={axisSelection.y}>
                    {#each attrOptionsY as key}<option value={key}>{key}</option>{/each}
                </select>
            </label>
            <label> Size:
                <select bind:value={axisSelection.size}>
                    {#each attrOptionsS as key}<option value={key}>{key}</option>{/each}
                </select>
            </label>
        </div>

        <div class="chartWrap" style="overflow:auto;">
            <Scatter
                movies={visibleMovies}
                x={axisSelection.x}
                y={axisSelection.y}
                size={axisSelection.size}
                width={plotWidth}
                height={plotHeight}
            />
        </div>

        <br />

        <div class="chartWrap" style="overflow:auto;">
            <Line data={getYearCountArray(movies)} bind:yearRange width={plotWidth} />
        </div>
    {/if}
</div>

<style>
    .container {
        margin: 10px auto;
        padding: 10px;
    }
    .chartWrap { /* allow big categorical charts without breaking the page */
        max-width: 100%;
    }
</style>