<script lang="ts">
	import { Button, type ButtonType } from '$lib/components/common'
	import { Loader2 } from 'lucide-svelte'
	import { getContext } from 'svelte'
	import type { AppInput } from '../../inputType'
	import type { Output } from '../../rx'
	import type { AppEditorContext } from '../../types'
	import AlignWrapper from '../helpers/AlignWrapper.svelte'
	import InputValue from '../helpers/InputValue.svelte'
	import type RunnableComponent from '../helpers/RunnableComponent.svelte'
	import RunnableWrapper from '../helpers/RunnableWrapper.svelte'

	export let id: string
	export let componentInput: AppInput | undefined
	export let configuration: Record<string, AppInput>
	export let recomputeIds: string[] | undefined = undefined
	export let extraQueryParams: Record<string, any> = {}
	export let horizontalAlignment: 'left' | 'center' | 'right' | undefined = undefined
	export let verticalAlignment: 'top' | 'center' | 'bottom' | undefined = undefined
	export let noWFull = false

	export const staticOutputs: string[] = ['loading', 'result']

	const { runnableComponents, worldStore } = getContext<AppEditorContext>('AppEditorContext')

	let labelValue: string
	let color: ButtonType.Color
	let size: ButtonType.Size
	let runnableComponent: RunnableComponent

	let isLoading: boolean = false
	let ownClick: boolean = false

	$: outputs = $worldStore?.outputsById[id] as {
		result: Output<Array<any>>
		loading: Output<boolean>
	}

	$: if (outputs?.loading != undefined) {
		outputs.loading.set(false, true)
	}

	$: outputs?.loading.subscribe({
		next: (value) => {
			isLoading = value
			if (ownClick && !value) {
				ownClick = false
			}
		}
	})

	$: loading = isLoading && ownClick
</script>

<InputValue {id} input={configuration.label} bind:value={labelValue} />
<InputValue {id} input={configuration.color} bind:value={color} />
<InputValue {id} input={configuration.size} bind:value={size} />

<RunnableWrapper
	bind:runnableComponent
	bind:componentInput
	{id}
	{extraQueryParams}
	autoRefresh={false}
>
	<AlignWrapper {noWFull} {horizontalAlignment} {verticalAlignment}>
		<Button
			on:pointerdown={(e) => {
				e?.stopPropagation()
				window.dispatchEvent(new Event('pointerup'))
			}}
			on:click={(e) => {
				e?.stopPropagation()
				e?.preventDefault()
				ownClick = true
				runnableComponent?.runComponent()

				if (recomputeIds) {
					recomputeIds.forEach((id) => {
						$runnableComponents[id]?.()
					})
				}
			}}
			{size}
			{color}
		>
			{labelValue}
			{#if !loading}
				<span class="w-5" />
			{:else}
				<Loader2 class="animate-spin ml-1" size={14} />
			{/if}
		</Button>
	</AlignWrapper>
</RunnableWrapper>
