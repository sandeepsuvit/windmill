<script lang="ts">
	import { Badge } from '$lib/components/common'
	import { classNames } from '$lib/utils'
	import { getContext } from 'svelte'
	import type { AppComponent, AppEditorContext } from '../../types'
	import PanelSection from '../settingsPanel/common/PanelSection.svelte'

	export let selectedScriptComponentId: string | undefined = undefined

	const { app, selectedComponent, lazyGrid } = getContext<AppEditorContext>('AppEditorContext')

	function selectInlineScript(id: string) {
		selectedScriptComponentId = id
		if (!id.startsWith('unused-')) {
			$selectedComponent = selectedScriptComponentId
		}
	}

	$: runnablesByName = $lazyGrid.reduce((acc, gridComponent) => {
		const component: AppComponent = gridComponent.data

		if (component.type === 'tablecomponent') {
			component.actionButtons.forEach((actionButton) => {
				if (actionButton.componentInput?.type === 'runnable') {
					if (actionButton.componentInput.runnable?.type === 'runnableByName') {
						acc.push({
							name: actionButton.componentInput.runnable.name,
							id: actionButton.id
						})
					}
				}
			})
		}

		const componentInput = component.componentInput

		if (componentInput?.type === 'runnable') {
			if (componentInput.runnable?.type === 'runnableByName') {
				acc.push({
					name: componentInput.runnable.name,
					id: gridComponent.id
				})
			}
		}
		return acc
	}, [] as { name: string; id: string }[])

	$: runnablesByPath = $lazyGrid.reduce((acc, gridComponent) => {
		const component: AppComponent = gridComponent.data

		if (component.type === 'tablecomponent') {
			component.actionButtons.forEach((actionButton) => {
				if (actionButton.componentInput?.type === 'runnable') {
					if (actionButton.componentInput.runnable?.type === 'runnableByPath') {
						acc.push({
							name: actionButton.componentInput.runnable.path,
							id: actionButton.id
						})
					}
				}
			})
		}

		const componentInput = component.componentInput

		if (componentInput?.type === 'runnable') {
			if (componentInput.runnable?.type === 'runnableByPath') {
				acc.push({
					name: componentInput.runnable.path,
					id: gridComponent.id
				})
			}
		}
		return acc
	}, [] as { name: string; id: string }[])

	// When seleced component changes, update selectedScriptComponentId
	$: {
		if (selectedComponent) {
			selectedScriptComponentId = $selectedComponent
		}
	}
</script>

<div class="h-full flex flex-col gap-4">
	<PanelSection title="Inline scripts" smallPadding>
		<div class="flex flex-col gap-2 w-full">
			{#if runnablesByName.length > 0}
				<div class="flex gap-2 flex-col ">
					{#each runnablesByName as { name, id }, index (index)}
						<!-- svelte-ignore a11y-click-events-have-key-events -->
						<div
							class="{classNames(
								'border flex  gap-1 truncate justify-between flex-row w-full items-center p-2 rounded-sm cursor-pointer hover:bg-blue-50 hover:text-blue-400',
								selectedScriptComponentId === id ? 'border-blue-500 border' : ''
							)},"
							on:click={() => selectInlineScript(id)}
						>
							<span class="text-xs truncate">{name}</span>
							<div>
								<Badge color="dark-indigo">{id}</Badge>
							</div>
						</div>
					{/each}
				</div>
			{/if}

			{#if $app.unusedInlineScripts?.length > 0}
				<div class="flex gap-2 flex-col ">
					{#each $app.unusedInlineScripts as unusedInlineScript, index (index)}
						<!-- svelte-ignore a11y-click-events-have-key-events -->
						<div
							class="{classNames(
								'border flex gap-1 truncate justify-between flex-row w-full items-center p-2 rounded-md cursor-pointer hover:bg-blue-50 hover:text-blue-400',
								selectedScriptComponentId === `unused-${index}` ? 'bg-blue-100 text-blue-600' : ''
							)},"
							on:click={() => selectInlineScript(`unused-${index}`)}
						>
							<span class="text-xs truncate">{unusedInlineScript.name}</span>
							<Badge color="red">Detached</Badge>
						</div>
					{/each}
				</div>
			{/if}

			{#if runnablesByName.length == 0 && $app.unusedInlineScripts?.length == 0}
				<div class="text-sm text-gray-500">No inline scripts</div>
			{/if}
		</div>
	</PanelSection>

	<PanelSection title="Others" smallPadding>
		<div class="flex flex-col gap-2 w-full">
			{#if runnablesByPath.length > 0}
				<div class="flex gap-2 flex-col ">
					{#each runnablesByPath as { name, id }, index (index)}
						<!-- svelte-ignore a11y-click-events-have-key-events -->
						<div
							class="{classNames(
								'border flex gap-1 truncate justify-between flex-row w-full items-center p-2 rounded-md cursor-pointer hover:bg-blue-50 hover:text-blue-400',
								selectedScriptComponentId === id ? 'bg-blue-100 text-blue-600' : ''
							)},"
							on:click={() => selectInlineScript(id)}
						>
							<span class="text-xs truncate">{name}</span>
							<Badge color="dark-indigo">{id}</Badge>
						</div>
					{/each}
				</div>
			{:else}
				<div class="text-sm text-gray-500">No items</div>
			{/if}
		</div>
	</PanelSection>
</div>
