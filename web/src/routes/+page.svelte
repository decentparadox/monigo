<script lang="ts">
	import { onMount } from 'svelte';
	import * as echarts from 'echarts';
	import * as Card from '$lib/components/ui/card/index.js';
	import { Button } from '$lib/components/ui/button/index.js';
	import { Skeleton } from '$lib/components/ui/skeleton/index.js';
	import { fetchServiceInfo, fetchMetrics, fetchServiceMetrics } from '$lib/api/monigo.js';
	import {
		Cpu,
		HardDrive,
		Activity,
		MemoryStick,
		Gauge,
		Clock,
		RefreshCw,
		Server
	} from 'lucide-svelte';

	let serviceInfo = $state<{
		service_name: string;
		service_start_time: string;
		go_version: string;
		process_id: number;
	} | null>(null);
	let metrics = $state<{
		core_statistics: Record<string, unknown>;
		load_statistics: Record<string, unknown>;
		cpu_statistics: Record<string, unknown>;
		memory_statistics: Record<string, unknown>;
		health: { service_health: Record<string, unknown>; system_health: Record<string, unknown> };
	} | null>(null);
	let error = $state<string | null>(null);
	let loading = $state(true);
	let historyMetric = $state('heap');
	let historyTimeRange = $state('5m');
	let historyData = $state<Array<{ time: string; value: Record<string, number> }>>([]);
	let historyLoading = $state(false);

	function loadData() {
		loading = true;
		error = null;
		Promise.all([fetchServiceInfo(), fetchMetrics()])
			.then(([info, m]) => {
				serviceInfo = info;
				metrics = m;
			})
			.catch((e) => (error = e.message))
			.finally(() => (loading = false));
	}

	function parsePercent(val: unknown): number {
		if (typeof val === 'number') return val;
		if (typeof val === 'string') return parseFloat(val.replace('%', '')) || 0;
		return 0;
	}

	function parseMemoryToMB(val: unknown): number {
		if (typeof val === 'number') return val / 1024;
		const s = String(val ?? '');
		const num = parseFloat(s) || 0;
		if (s.includes('GB')) return num * 1024;
		if (s.includes('MB')) return num;
		if (s.includes('KB')) return num / 1024;
		return num;
	}

	function toLocalISOString(date: Date) {
		const tzOffset = -date.getTimezoneOffset();
		const diff = tzOffset >= 0 ? '+' : '-';
		const pad = (n: number) => Math.floor(Math.abs(n)).toString().padStart(2, '0');
		return (
			date.getFullYear() +
			'-' +
			pad(date.getMonth() + 1) +
			'-' +
			pad(date.getDate()) +
			'T' +
			pad(date.getHours()) +
			':' +
			pad(date.getMinutes()) +
			':' +
			pad(date.getSeconds()) +
			'.000' +
			diff +
			pad(tzOffset / 60) +
			':' +
			pad(tzOffset % 60)
		);
	}

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

	const metricFields: Record<string, string[]> = {
		heap: ['heap_alloc', 'heap_sys', 'heap_inuse', 'heap_idle', 'heap_released'],
		stack: ['stack_inuse', 'stack_sys'],
		gc: ['pause_total_ns', 'num_gc', 'gc_cpu_fraction'],
		misc: ['m_span_inuse', 'm_span_sys', 'm_cache_inuse', 'm_cache_sys', 'buck_hash_sys', 'gc_sys', 'other_sys']
	};

	function loadHistoryChart() {
		historyLoading = true;
		const now = new Date();
		const mins = timeRanges[historyTimeRange] ?? 5;
		const start = new Date(now.getTime() - mins * 60000);
		fetchServiceMetrics({
			field_name: metricFields[historyMetric] ?? metricFields.heap,
			timerange: historyTimeRange,
			start_time: toLocalISOString(start),
			end_time: toLocalISOString(now)
		})
			.then((data) => {
				historyData = data;
			})
			.catch((e) => (error = e.message))
			.finally(() => (historyLoading = false));
	}

	let loadChartEl: HTMLDivElement;
	let cpuChartEl: HTMLDivElement;
	let memChartEl: HTMLDivElement;
	let heapChartEl: HTMLDivElement;
	let historyChartEl: HTMLDivElement;

	onMount(() => {
		loadData();
		loadHistoryChart();
		const onResize = () => {
			[loadChartEl, cpuChartEl, memChartEl, heapChartEl, historyChartEl].forEach((el) => {
				if (el) {
					const inst = echarts.getInstanceByDom(el);
					if (inst) inst.resize();
				}
			});
		};
		window.addEventListener('resize', onResize);
		return () => {
			window.removeEventListener('resize', onResize);
			[loadChartEl, cpuChartEl, memChartEl, heapChartEl, historyChartEl].forEach((el) => {
				if (el) {
					const inst = echarts.getInstanceByDom(el);
					if (inst) inst.dispose();
				}
			});
		};
	});

	$effect(() => {
		if (!metrics || loading || !loadChartEl || !cpuChartEl || !memChartEl || !heapChartEl) return;
		const load = metrics.load_statistics;
		const cpu = metrics.cpu_statistics;
		const mem = metrics.memory_statistics;
		if (!load || !cpu || !mem) return;

		const loadChart = echarts.init(loadChartEl);
		loadChart.setOption({
			title: { text: 'Load Statistics' },
			tooltip: { trigger: 'axis' },
			xAxis: {
				type: 'category',
				data: [
					'Service CPU',
					'System CPU',
					'Total CPU',
					'Service Memory',
					'System Memory'
				]
			},
			yAxis: { type: 'value', max: 100 },
			series: [
				{
					data: [
						parsePercent(load.service_cpu_load),
						parsePercent(load.system_cpu_load),
						parsePercent(load.total_cpu_load),
						parsePercent(load.service_memory_load),
						parsePercent(load.system_memory_load)
					],
					type: 'bar',
					itemStyle: {
						color: (p: { value: number }) =>
							p.value > 90 ? '#ef4444' : p.value > 80 ? '#f97316' : p.value > 50 ? '#eab308' : '#22c55e'
					}
				}
			]
		});

		const cpuChart = echarts.init(cpuChartEl);
		cpuChart.setOption({
			title: { text: 'CPU Statistics' },
			tooltip: { trigger: 'item' },
			legend: { bottom: 0 },
			series: [
				{
					type: 'pie',
					radius: '55%',
					data: [
						{ value: cpu.cores_used_by_service, name: 'Service', itemStyle: { color: '#00A1E4' } },
						{ value: cpu.cores_used_by_system, name: 'System', itemStyle: { color: '#FF6F61' } },
						{ value: cpu.total_cores, name: 'Total', itemStyle: { color: '#FFD166' } }
					]
				}
			]
		});

		const memChart = echarts.init(memChartEl);
		memChart.setOption({
			title: { text: 'Memory Distribution' },
			tooltip: { trigger: 'item', formatter: '{b}: {c} MB ({d}%)' },
			legend: { bottom: 0 },
			series: [
				{
					type: 'pie',
					radius: '55%',
					data: [
						{ value: parseMemoryToMB(mem.memory_used_by_service), name: 'Service' },
						{ value: parseMemoryToMB(mem.memory_used_by_system), name: 'System' },
						{ value: parseMemoryToMB(mem.available_memory), name: 'Available' }
					]
				}
			]
		});

		const heapValues: number[] = [];
		for (const r of mem.mem_stats_records || []) {
			if (['HeapAlloc', 'HeapSys', 'HeapIdle', 'HeapInuse', 'HeapReleased'].includes(r.record_name)) {
				heapValues.push(Number(r.record_value) || 0);
			}
		}

		const heapChart = echarts.init(heapChartEl);
		heapChart.setOption({
			title: { text: 'Heap Memory Usage' },
			tooltip: {},
			xAxis: {
				type: 'category',
				data: ['HeapAlloc', 'HeapSys', 'HeapIdle', 'HeapInuse', 'HeapReleased']
			},
			yAxis: { type: 'value', name: 'MB' },
			series: [{ type: 'bar', data: heapValues }]
		});

		return () => {
			loadChart.dispose();
			cpuChart.dispose();
			memChart.dispose();
			heapChart.dispose();
		};
	});

	$effect(() => {
		if (!historyChartEl || historyData.length === 0 || historyLoading) return;
		const chart = echarts.init(historyChartEl);
		const seriesData: Record<string, [string, number][]> = {};
		historyData.forEach((dp) => {
			const timeLabel = new Date(dp.time).toLocaleString();
			for (const key of Object.keys(dp.value || {})) {
				if (!seriesData[key]) seriesData[key] = [];
				seriesData[key].push([timeLabel, dp.value[key]]);
			}
		});
		const series = Object.entries(seriesData).map(([name, data]) => ({
			name,
			type: 'line',
			data,
			smooth: true
		}));
		chart.setOption({
			title: { text: 'History Statistics', left: 'center' },
			tooltip: { trigger: 'axis' },
			legend: { top: 30, data: Object.keys(seriesData) },
			xAxis: { type: 'category', boundaryGap: false },
			yAxis: { type: 'value' },
			series
		});
		return () => chart.dispose();
	});

	$effect(() => {
		historyMetric;
		historyTimeRange;
		loadHistoryChart();
	});
</script>

<svelte:head><title>MoniGo Dashboard</title></svelte:head>

<div class="space-y-6 p-6">
	<div class="flex items-center justify-between">
		<h1 class="text-2xl font-bold">Dashboard</h1>
		<Button variant="outline" size="sm" onclick={loadData} disabled={loading}>
			<RefreshCw class="mr-2 h-4 w-4" />
			Refresh
		</Button>
	</div>

	{#if error}
		<Card.Root class="border-destructive">
			<Card.Content class="pt-6">
				<p class="text-destructive">{error}</p>
				<p class="text-sm text-muted-foreground mt-2">
					Ensure the MoniGo backend is running and the dashboard is served from it.
				</p>
			</Card.Content>
		</Card.Root>
	{/if}

	{#if loading}
		<div class="grid gap-4 md:grid-cols-2 lg:grid-cols-4">
			{#each Array(8) as _}
				<Card.Root><Card.Content class="pt-6"><Skeleton class="h-20 w-full" /></Card.Content></Card.Root>
			{/each}
		</div>
	{:else if metrics}
		<!-- Service info -->
		<div class="grid gap-4 md:grid-cols-2 lg:grid-cols-4">
			{#if serviceInfo}
				<Card.Root>
					<Card.Header class="flex flex-row items-center justify-between pb-2">
						<Card.Title class="text-sm font-medium">Service</Card.Title>
						<Server class="h-4 w-4 text-muted-foreground" />
					</Card.Header>
					<Card.Content>
						<div class="text-2xl font-bold">{serviceInfo.service_name}</div>
					</Card.Content>
				</Card.Root>
				<Card.Root>
					<Card.Header class="flex flex-row items-center justify-between pb-2">
						<Card.Title class="text-sm font-medium">Go Version</Card.Title>
					</Card.Header>
					<Card.Content>
						<div class="text-2xl font-bold">{serviceInfo.go_version}</div>
					</Card.Content>
				</Card.Root>
				<Card.Root>
					<Card.Header class="flex flex-row items-center justify-between pb-2">
						<Card.Title class="text-sm font-medium">Process ID</Card.Title>
					</Card.Header>
					<Card.Content>
						<div class="text-2xl font-bold">{serviceInfo.process_id}</div>
					</Card.Content>
				</Card.Root>
			{/if}
			<Card.Root>
				<Card.Header class="flex flex-row items-center justify-between pb-2">
					<Card.Title class="text-sm font-medium">Health</Card.Title>
					<Gauge class="h-4 w-4 text-muted-foreground" />
				</Card.Header>
				<Card.Content>
					<div class="text-2xl font-bold">
						{metrics.health?.service_health?.percent ?? 0}%
					</div>
					<p class="text-xs text-muted-foreground mt-1">
						{metrics.health?.service_health?.message ?? '-'}
					</p>
				</Card.Content>
			</Card.Root>
		</div>

		<!-- Stats cards -->
		<div class="grid gap-4 md:grid-cols-2 lg:grid-cols-3">
			<Card.Root>
				<Card.Header class="flex flex-row items-center justify-between pb-2">
					<Card.Title class="text-sm font-medium">Go Routines</Card.Title>
					<Activity class="h-4 w-4 text-muted-foreground" />
				</Card.Header>
				<Card.Content>
					<div class="text-2xl font-bold">{metrics.core_statistics?.goroutines ?? '-'}</div>
				</Card.Content>
			</Card.Root>
			<Card.Root>
				<Card.Header class="flex flex-row items-center justify-between pb-2">
					<Card.Title class="text-sm font-medium">Load</Card.Title>
					<Cpu class="h-4 w-4 text-muted-foreground" />
				</Card.Header>
				<Card.Content>
					<div class="text-2xl font-bold">{metrics.load_statistics?.overall_load_of_service ?? '-'}</div>
				</Card.Content>
			</Card.Root>
			<Card.Root>
				<Card.Header class="flex flex-row items-center justify-between pb-2">
					<Card.Title class="text-sm font-medium">Cores</Card.Title>
					<HardDrive class="h-4 w-4 text-muted-foreground" />
				</Card.Header>
				<Card.Content>
					<div class="text-2xl font-bold">
						{metrics.cpu_statistics?.cores_used_by_service ?? '-'} / {metrics.cpu_statistics?.total_cores ?? '-'}
					</div>
				</Card.Content>
			</Card.Root>
			<Card.Root>
				<Card.Header class="flex flex-row items-center justify-between pb-2">
					<Card.Title class="text-sm font-medium">Memory</Card.Title>
					<MemoryStick class="h-4 w-4 text-muted-foreground" />
				</Card.Header>
				<Card.Content>
					<div class="text-2xl font-bold">{metrics.memory_statistics?.memory_used_by_service ?? '-'}</div>
				</Card.Content>
			</Card.Root>
			<Card.Root>
				<Card.Header class="flex flex-row items-center justify-between pb-2">
					<Card.Title class="text-sm font-medium">CPU Usage</Card.Title>
					<Cpu class="h-4 w-4 text-muted-foreground" />
				</Card.Header>
				<Card.Content>
					<div class="text-2xl font-bold">{metrics.cpu_statistics?.cores_used_by_service_in_percent ?? '-'}</div>
				</Card.Content>
			</Card.Root>
			<Card.Root>
				<Card.Header class="flex flex-row items-center justify-between pb-2">
					<Card.Title class="text-sm font-medium">Uptime</Card.Title>
					<Clock class="h-4 w-4 text-muted-foreground" />
				</Card.Header>
				<Card.Content>
					<div class="text-2xl font-bold">{metrics.core_statistics?.uptime ?? '-'}</div>
				</Card.Content>
			</Card.Root>
		</div>

		<!-- Charts -->
		<div class="grid gap-4 md:grid-cols-2">
			<Card.Root>
				<Card.Header><Card.Title>Load Statistics</Card.Title></Card.Header>
				<Card.Content>
					<div bind:this={loadChartEl} class="h-64 w-full"></div>
				</Card.Content>
			</Card.Root>
			<Card.Root>
				<Card.Header><Card.Title>CPU Statistics</Card.Title></Card.Header>
				<Card.Content>
					<div bind:this={cpuChartEl} class="h-64 w-full"></div>
				</Card.Content>
			</Card.Root>
			<Card.Root>
				<Card.Header><Card.Title>Memory Distribution</Card.Title></Card.Header>
				<Card.Content>
					<div bind:this={memChartEl} class="h-64 w-full"></div>
				</Card.Content>
			</Card.Root>
			<Card.Root>
				<Card.Header><Card.Title>Heap Memory Usage</Card.Title></Card.Header>
				<Card.Content>
					<div bind:this={heapChartEl} class="h-64 w-full"></div>
				</Card.Content>
			</Card.Root>
		</div>

		<!-- History chart -->
		<Card.Root>
			<Card.Header class="flex flex-row items-center justify-between">
				<Card.Title>History Statistics</Card.Title>
				<div class="flex gap-2">
					<select
						bind:value={historyMetric}
						class="rounded-md border border-input bg-background px-3 py-1 text-sm"
					>
						<option value="heap">Heap Memory</option>
						<option value="stack">Stack Memory</option>
						<option value="gc">Garbage Collection</option>
						<option value="misc">Misc System Memory</option>
					</select>
					<select
						bind:value={historyTimeRange}
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
				</div>
			</Card.Header>
			<Card.Content>
				{#if historyLoading}
					<div class="h-64 flex items-center justify-center"><Skeleton class="h-full w-full" /></div>
				{:else}
					<div bind:this={historyChartEl} class="h-64 w-full"></div>
				{/if}
			</Card.Content>
		</Card.Root>
	{/if}
</div>
