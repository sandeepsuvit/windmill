<script lang="ts">
	import { getContext, onMount } from 'svelte'
	import type { AppInput } from '../../inputType'
	import type { AppEditorContext } from '../../types'
	import { isScriptByNameDefined, isScriptByPathDefined } from '../../utils'
	import NonRunnableComponent from './NonRunnableComponent.svelte'
	import RunnableComponent from './RunnableComponent.svelte'

	export let componentInput: AppInput | undefined
	export let id: string
	export let result: any = undefined
	export let noMinH = false

	export let extraQueryParams: Record<string, any> = {}
	export let autoRefresh: boolean = true
	export let runnableComponent: RunnableComponent | undefined = undefined
	export let forceSchemaDisplay: boolean = false

	const { staticExporter, noBackend } = getContext<AppEditorContext>('AppEditorContext')

	if (noBackend && componentInput?.type == 'runnable') {
		result = componentInput?.['value']
	}
	onMount(() => {
		$staticExporter[id] = () => result
	})

	function isRunnableDefined() {
		return isScriptByNameDefined(componentInput) || isScriptByPathDefined(componentInput)
	}
</script>

{#if componentInput === undefined}
	<slot />
{:else if componentInput.type === 'runnable' && isRunnableDefined()}
	<RunnableComponent
		bind:this={runnableComponent}
		bind:fields={componentInput.fields}
		bind:result
		runnable={componentInput.runnable}
		{autoRefresh}
		{id}
		{extraQueryParams}
		{forceSchemaDisplay}
		{noMinH}
	>
		<slot />
	</RunnableComponent>
{:else}
	<NonRunnableComponent bind:result {id} {componentInput}>
		<slot />
	</NonRunnableComponent>
{/if}
