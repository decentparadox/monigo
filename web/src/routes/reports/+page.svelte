<script lang="ts">
	import { onMount } from 'svelte';
	import * as echarts from 'echarts';
	import * as Card from '$lib/components/ui/card/index.js';
	import { Button } from '$lib/components/ui/button/index.js';
	import { Skeleton } from '$lib/components/ui/skeleton/index.js';
	import { fetchReports } from '$lib/api/monigo.js';
	import { RefreshCw } from 'lucide-svelte';

	let reports = $state<Array<{ time: string; value: Record<string, number> }>>([]);
	let loading = $state(true);
	let error = $state<string | null>(null);
	let timeframe = $state('1h');
	let chartEl: HTMLDivElement;

	const timeRanges: Record<string, number> = {
		'5m': 5,
		'15m': 15,
		'30m': 30,
		'1h': 60,
		'6h': 360,
		'1d': 1440,
		'3d': 4320,
		'7d': 10080
	};

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

	async function load() {
		loading = true;
		error = null;
		const now = new Date();
		const mins = timeRanges[timeframe] ?? 60;
		const start = new Date(now.getTime() - mins * 60000);
		fetchReports({
			topic: 'LoadStatistics',
			start_time: toLocalISOString(start),
			end_time: toLocalISOString(now),
			time_frame: timeframe
		})
			.then((data) => (reports = data))
			.catch((e) => (error = e.message))
			.finally(() => (loading = false));
	}

	onMount(() => {
		return () => {
			if (chartEl) {
				const inst = echarts.getInstanceByDom(chartEl);
				if (inst) inst.dispose();
			}
		};
	});

	$effect(() => {
		timeframe;
		load();
	});

	$effect(() => {
		if (!chartEl || reports.length === 0 || loading) return;
		const chart = echarts.init(chartEl);
		const seriesData: Record<string, [string, number][]> = {};
		reports.forEach((dp) => {
			const t = new Date(dp.time).toLocaleString();
			for (const k of Object.keys(dp.value || {})) {
				if (!seriesData[k]) seriesData[k] = [];
				seriesData[k].push([t, dp.value[k]]);
			}
		});
		chart.setOption({
			title: { text: 'Load Report' },
			tooltip: { trigger: 'axis' },
			legend: { top: 30, data: Object.keys(seriesData) },
			xAxis: { type: 'category', boundaryGap: false },
			yAxis: { type: 'value' },
			series: Object.entries(seriesData).map(([name, data]) => ({ name, type: 'line', data, smooth: true }))
		});
		return () => chart.dispose();
	});
</script>

<svelte:head><title>Reports - MoniGo</title></svelte:head>

<div class="space-y-6 p-6">
	<div class="flex items-center justify-between">
		<h1 class="text-2xl font-bold">Reports</h1>
		<div class="flex gap-2">
			<select
				bind:value={timeframe}
				class="rounded-md border border-input bg-background px-3 py-1 text-sm"
			>
				<option value="5m">5 Minutes</option>
				<option value="15m">15 Minutes</option>
				<option value="30m">30 Minutes</option>
				<option value="1h">1 Hour</option>
				<option value="6h">6 Hours</option>
				<option value="1d">1 Day</option>
				<option value="3d">3 Days</option>
				<option value="7d">7 Days</option>
			</select>
			<Button variant="outline" size="sm" onclick={load} disabled={loading}>
				<RefreshCw class="mr-2 h-4 w-4" />
				Refresh
			</Button>
		</div>
	</div>

	{#if error}
		<Card.Root class="border-destructive">
			<Card.Content class="pt-6"><p class="text-destructive">{error}</p></Card.Content>
		</Card.Root>
	{:else}
		<Card.Root>
			<Card.Header><Card.Title>Load Metrics Over Time</Card.Title></Card.Header>
			<Card.Content>
				{#if loading}
					<Skeleton class="h-64 w-full" />
				{:else if reports.length > 0}
					<div bind:this={chartEl} class="h-64 w-full"></div>
				{:else}
					<p class="text-muted-foreground text-sm">No report data available.</p>
				{/if}
			</Card.Content>
		</Card.Root>
	{/if}
</div>
