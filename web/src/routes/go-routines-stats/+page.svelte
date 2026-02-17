<script lang="ts">
	import { onMount } from 'svelte';
	import * as echarts from 'echarts';
	import * as Card from '$lib/components/ui/card/index.js';
	import { Button } from '$lib/components/ui/button/index.js';
	import { Skeleton } from '$lib/components/ui/skeleton/index.js';
	import { fetchGoRoutinesStats, fetchServiceMetrics } from '$lib/api/monigo.js';
	import { RefreshCw, Activity } from 'lucide-svelte';

	let stats = $state<{ number_of_goroutines: number; stack_view: string[] } | null>(null);
	let historyData = $state<Array<{ time: string; value: Record<string, number> }>>([]);
	let loading = $state(true);
	let error = $state<string | null>(null);
	let chartEl: HTMLDivElement;

	function toLocalISOString(d: Date) {
		const tz = -d.getTimezoneOffset();
		const pad = (n: number) => Math.floor(Math.abs(n)).toString().padStart(2, '0');
		return (
			d.getFullYear() +
			'-' +
			pad(d.getMonth() + 1) +
			'-' +
			pad(d.getDate()) +
			'T' +
			pad(d.getHours()) +
			':' +
			pad(d.getMinutes()) +
			':' +
			pad(d.getSeconds()) +
			'.000' +
			(tz >= 0 ? '+' : '-') +
			pad(tz / 60) +
			':' +
			pad(tz % 60)
		);
	}

	function load() {
		loading = true;
		error = null;
		const now = new Date();
		const start = new Date(now.getTime() - 60 * 60000);
		Promise.all([
			fetchGoRoutinesStats(),
			fetchServiceMetrics({
				field_name: ['goroutines'],
				timerange: '1h',
				start_time: toLocalISOString(start),
				end_time: toLocalISOString(now)
			})
		])
			.then(([s, h]) => {
				stats = s;
				historyData = h;
			})
			.catch((e) => (error = e.message))
			.finally(() => (loading = false));
	}

	onMount(() => {
		load();
		return () => {
			if (chartEl) {
				const inst = echarts.getInstanceByDom(chartEl);
				if (inst) inst.dispose();
			}
		};
	});

	$effect(() => {
		if (!chartEl || historyData.length === 0 || loading) return;
		const chart = echarts.init(chartEl);
		const seriesData: Record<string, [string, number][]> = {};
		historyData.forEach((dp) => {
			const t = new Date(dp.time).toLocaleString();
			for (const k of Object.keys(dp.value || {})) {
				if (!seriesData[k]) seriesData[k] = [];
				seriesData[k].push([t, dp.value[k]]);
			}
		});
		chart.setOption({
			title: { text: 'Goroutines Over Time' },
			tooltip: { trigger: 'axis' },
			xAxis: { type: 'category', boundaryGap: false },
			yAxis: { type: 'value' },
			series: Object.entries(seriesData).map(([name, data]) => ({ name, type: 'line', data, smooth: true }))
		});
		return () => chart.dispose();
	});
</script>

<svelte:head><title>Go Routines Stats - MoniGo</title></svelte:head>

<div class="space-y-6 p-6 w-full">
	<div class="flex items-center justify-between">
		<h1 class="text-2xl font-bold">Go Routines Stats</h1>
		<Button variant="outline" size="sm" onclick={load} disabled={loading}>
			<RefreshCw class="mr-2 h-4 w-4" />
			Refresh
		</Button>
	</div>

	{#if error}
		<Card.Root class="border-destructive">
			<Card.Content class="pt-6"><p class="text-destructive">{error}</p></Card.Content>
		</Card.Root>
	{:else if loading}
		<Card.Root><Card.Content class="pt-6"><Skeleton class="h-32 w-full" /></Card.Content></Card.Root>
	{:else if stats}
		<Card.Root>
			<Card.Header class="flex flex-row items-center justify-between">
				<Card.Title class="flex items-center gap-2">
					<Activity class="h-5 w-5" />
					Goroutine Count
				</Card.Title>
				<div class="text-3xl font-bold">{stats.number_of_goroutines}</div>
			</Card.Header>
		</Card.Root>

		<Card.Root>
			<Card.Header><Card.Title>Goroutines Over Time (1h)</Card.Title></Card.Header>
			<Card.Content>
				{#if historyData.length > 0}
					<div bind:this={chartEl} class="h-64 w-full"></div>
				{:else}
					<p class="text-muted-foreground text-sm">No history data available.</p>
				{/if}
			</Card.Content>
		</Card.Root>

		{#if stats.stack_view?.length}
			<Card.Root>
				<Card.Header><Card.Title>Stack Traces</Card.Title></Card.Header>
				<Card.Content>
					<div class="space-y-4">
						{#each stats.stack_view as trace, i}
							<details class="rounded-md border p-4">
								<summary class="cursor-pointer font-medium">Goroutine {i + 1}</summary>
								<pre class="mt-2 text-xs overflow-auto whitespace-pre-wrap text-muted-foreground">{trace}</pre>
							</details>
						{/each}
					</div>
				</Card.Content>
			</Card.Root>
		{/if}
	{/if}
</div>
