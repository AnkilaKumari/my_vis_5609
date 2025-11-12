<script lang="ts">
    import type { TMovie } from "../types";
    import * as d3 from "d3";

    type TProps = {
        movies: TMovie[];
        x: keyof TMovie;
        y: keyof TMovie;
        size?: keyof TMovie;
        width?: number;
        height?: number;
    };
    const { movies, x, y, size, height = 500, width = 600 }: TProps = $props();

    let selectedMovie: TMovie | undefined = $state();

    // channel type detection (guarded for empty data)
    const xSample = $derived(movies[0]?.[x]);
    const ySample = $derived(movies[0]?.[y]);
    const isBandX = $derived(typeof xSample === "string" || typeof xSample === "object");
    const isBandY = $derived(typeof ySample === "string" || typeof ySample === "object");

    // dynamic margins (more room for long categorical labels)
    const margin = $derived({
        top: 15,
        bottom: isBandX ? 90 : 50,
        left: isBandY ? 120 : 30,
        right: 10,
    });

    const usableArea = $derived({
        top: margin.top,
        right: width - margin.right,
        bottom: height - margin.bottom,
        left: margin.left,
    });

    const sizeRange = [3, 15];

    function getScale(attrName: keyof TMovie, axis: "x" | "y" | "size", rows: TMovie[]) {
        if (rows.length === 0) return null;

        let range: number[] = [0, 0];
        if (axis === "x") range = [usableArea.left, usableArea.right];
        else if (axis === "y") range = [usableArea.bottom, usableArea.top];
        else if (axis === "size") range = sizeRange;

        const v0 = rows[0][attrName];

        if (typeof v0 === "string") {
            return d3.scaleBand()
                .domain(rows.map((d) => d[attrName] as string))
                .range(range)
                .padding(0.2);
        } else if (typeof v0 === "number") {
            return d3.scaleLinear()
                .domain(d3.extent(rows, (d) => d[attrName] as number) as [number, number])
                .nice()
                .range(range);
        } else if (v0 instanceof Date) {
            return d3.scaleTime()
                .domain(d3.extent(rows, (d) => d[attrName] as Date) as [Date, Date])
                .nice()
                .range(range);
        } else if (typeof v0 === "object") {
            // array (e.g., genres)
            const all = rows.map((d) => d[attrName] as string[]).reduce((a, v) => a.concat(v || []), []);
            return d3.scaleBand().domain([...new Set(all)]).range(range).padding(0.2);
        }
        return null;
    }

    const xScale = $derived(getScale(x, "x", movies));
    const yScale = $derived(getScale(y, "y", movies));
    const sizeScale = $derived(size ? getScale(size, "size", movies) : null);

    let xAxis: SVGGElement | null = null;
    let yAxis: SVGGElement | null = null;

    // center circles on band scales
    function posOnScale(scale: any, v: any) {
        if (!scale) return undefined;
        const base = scale(v);
        if (base == null) return undefined;
        return typeof scale.bandwidth === "function" ? base + scale.bandwidth() / 2 : base;
    }

    function updateAxis() {
        if (!xScale || !yScale) return;

        const gx = d3.select(xAxis).call(d3.axisBottom(xScale as any));
        gx.selectAll("text")
            .attr("transform", isBandX ? "rotate(45)" : null)
            .style("text-anchor", isBandX ? "start" : "middle");

        d3.select(yAxis).call(d3.axisLeft(yScale as any));
    }

    // tip1: click to select/deselect
    function handleMovieClick(movie: TMovie) {
        selectedMovie = selectedMovie === movie ? undefined : movie;
    }

    $effect(() => { updateAxis(); });

    // unique clipPath id
    const clipId = `clip-${Math.random().toString(36).slice(2)}`;
</script>

<svg {width} {height}>
    <defs>
        <clipPath id={clipId}>
            <rect
                x={usableArea.left}
                y={usableArea.top}
                width={usableArea.right - usableArea.left}
                height={usableArea.bottom - usableArea.top}
            />
        </clipPath>
    </defs>

    <!-- CLIPPED plotting layer -->
    <g class="points" clip-path={"url(#" + clipId + ")"}>
        {#each movies as movie}
            {#if x === "genres"}
                {#each movie["genres"] as genre}
                    {#if xScale && yScale}
                        <!-- svelte-ignore a11y_click_events_have_key_events -->
                        <!-- svelte-ignore a11y_no_static_element_interactions -->
                        <circle
                            cx={posOnScale(xScale, genre)}
                            cy={(yScale as any)(movie[y])}
                            r={sizeScale ? (sizeScale as any)(movie[size]) ?? sizeRange[0] : sizeRange[0]}
                            fill={movie === selectedMovie ? "steelblue" : "transparent"}
                            stroke="steelblue"
                            stroke-width="2"
                            opacity={movie === selectedMovie ? 1 : 0.5}
                            onclick={() => handleMovieClick(movie)}
                        />
                    {/if}
                {/each}
            {:else}
                {#if xScale && yScale}
                    <!-- svelte-ignore a11y_click_events_have_key_events -->
                    <!-- svelte-ignore a11y_no_static_element_interactions -->
                    <circle
                        cx={posOnScale(xScale, movie[x])}
                        cy={(yScale as any)(movie[y])}
                        r={sizeScale ? (sizeScale as any)(movie[size]) ?? sizeRange[0] : sizeRange[0]}
                        fill={movie === selectedMovie ? "steelblue" : "transparent"}
                        stroke="steelblue"
                        stroke-width="2"
                        opacity={movie === selectedMovie ? 1 : 0.5}
                        onclick={() => handleMovieClick(movie)}
                    />
                {/if}
            {/if}
        {/each}
    </g>

    <!-- axes (not clipped) -->
    <g transform={`translate(0, ${usableArea.bottom})`} bind:this={xAxis} />
    <g transform={`translate(${usableArea.left}, 0)`} bind:this={yAxis} />
</svg>

<div class="selectedInfo">
    {selectedMovie
        ? JSON.stringify(selectedMovie, null, 2)
        : "Click on a point to see details"}
</div>

<style>
    .points circle {
        transition: r 0.5s, cx 0.5s, cy 0.5s;
        cursor: pointer;
    }
    svg { display: inline-block; vertical-align: top; }
    .selectedInfo {
        display: inline-block; vertical-align: top; width: 250px;
        color: #555; font-family: monospace; white-space: pre-wrap;
        background-color: #f9f9f9; padding: 10px; border: 1px solid #ccc; border-radius: 4px; overflow: auto;
    }
</style>
