<script>
    import { onMount } from 'svelte';
    import { dev } from '$app/environment';
    import * as d3 from 'd3';
    import Graph from './Graph.svelte';

    let data = [];

    onMount(async () => {
        const csvPath = `nics-firearm-background-checks.csv`
        const res = await fetch(
            csvPath
        );
        const csv = await res.text();
        await d3.csvParse(csv, (d) => {
            let arr = d["month"].split("-")
            data.push({
                date: new Date(Date.UTC(arr[0], arr[1] - 1)),
                state: d["state"],
                permit: parseInt(d["permit"]),
            });
        });
        data = data;
    });


</script>

<main>
    <Graph {data} />
</main>

<style>
</style>
