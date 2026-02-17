<script lang="ts">
	import { onMount } from 'svelte';
	import * as Card from '$lib/components/ui/card/index.js';
	import { Button } from '$lib/components/ui/button/index.js';
	import { Skeleton } from '$lib/components/ui/skeleton/index.js';
	import { fetchFunctionTrace, fetchFunctionDetails } from '$lib/api/monigo.js';
	import { Search, Eye } from 'lucide-svelte';

	let functions = $state<Record<string, { function_last_ran_at: string }>>({});
	let selectedFunc = $state<string | null>(null);
	let funcDetails = $state<string | null>(null);
	let loading = $state(true);
	let detailsLoading = $state(false);
	let error = $state<string | null>(null);

	function load() {
		loading = true;
		error = null;
		fetchFunctionTrace()
			.then((data) => {
				functions = data;
			})
			.catch((e) => (error = e.message))
			.finally(() => (loading = false));
	}

	function viewDetails(name: string) {
		selectedFunc = name;
		funcDetails = null;
		detailsLoading = true;
		fetchFunctionDetails(name)
			.then((data) => (funcDetails = typeof data === 'string' ? data : JSON.stringify(data, null, 2)))
			.catch((e) => (funcDetails = `Error: ${e.message}`))
			.finally(() => (detailsLoading = false));
	}

	onMount(load);
</script>

<svelte:head><title>Function Metrics - MoniGo</title></svelte:head>

<div class="space-y-6 p-6 w-full">
	<div class="flex items-center justify-between">
		<h1 class="text-2xl font-bold">Function Metrics</h1>
		<Button variant="outline" size="sm" onclick={load} disabled={loading}>
			<Search class="mr-2 h-4 w-4" />
			Refresh
		</Button>
	</div>

	{#if error}
		<Card.Root class="border-destructive">
			<Card.Content class="pt-6"><p class="text-destructive">{error}</p></Card.Content>
		</Card.Root>
	{:else if loading}
		<Card.Root><Card.Content class="pt-6"><Skeleton class="h-32 w-full" /></Card.Content></Card.Root>
	{:else if Object.keys(functions).length === 0}
		<Card.Root>
			<Card.Content class="pt-6">
				<p class="text-muted-foreground">
					No function metrics available. Instrument functions with monigo.TraceFunction() to see metrics.
				</p>
			</Card.Content>
		</Card.Root>
	{:else}
		<div class="flex items-center gap-2 mb-4">
			<span class="text-sm font-medium">Total functions:</span>
			<span class="text-lg font-bold">{Object.keys(functions).length}</span>
		</div>
		<div class="grid gap-4 md:grid-cols-2 lg:grid-cols-3">
			{#each Object.entries(functions) as [name, data]}
				<Card.Root>
					<Card.Header class="pb-2">
						<Card.Title class="text-sm font-medium truncate" title={name}>{name}</Card.Title>
					</Card.Header>
					<Card.Content>
						<p class="text-xs text-muted-foreground mb-3">Last ran: {data.function_last_ran_at}</p>
						<Button variant="outline" size="sm" onclick={() => viewDetails(name)}>
							<Eye class="mr-2 h-4 w-4" />
							Details
						</Button>
					</Card.Content>
				</Card.Root>
			{/each}
		</div>

		{#if selectedFunc}
			<Card.Root class="mt-6">
				<Card.Header class="flex flex-row items-center justify-between">
					<Card.Title>Details: {selectedFunc}</Card.Title>
					<Button variant="ghost" size="sm" onclick={() => (selectedFunc = null)}>Close</Button>
				</Card.Header>
				<Card.Content>
					{#if detailsLoading}
						<Skeleton class="h-48 w-full" />
					{:else if funcDetails}
						<pre class="text-xs overflow-auto max-h-96 rounded-md bg-muted p-4">{funcDetails}</pre>
					{/if}
				</Card.Content>
			</Card.Root>
		{/if}
	{/if}
</div>
