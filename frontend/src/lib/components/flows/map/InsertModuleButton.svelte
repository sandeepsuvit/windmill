<script lang="ts">
	import { Menu } from '$lib/components/common'
	import {
		faCode,
		faCodeBranch,
		faPlus,
		faBarsStaggered,
		faBolt,
		faCheck
	} from '@fortawesome/free-solid-svg-icons'
	import { createEventDispatcher } from 'svelte'
	import Icon from 'svelte-awesome'
	import { Repeat } from 'lucide-svelte'

	const dispatch = createEventDispatcher()
	export let trigger = false
</script>

<Menu noMinW placement="bottom-start" let:close>
	<button
		slot="trigger"
		type="button"
		class="text-gray-900 bg-white border mx-0.5 border-gray-300 focus:outline-none hover:bg-gray-100 focus:ring-4 focus:ring-gray-200 font-medium rounded-full text-sm w-6 h-6 flex items-center justify-center"
	>
		<Icon data={faPlus} scale={0.8} />
	</button>
	<div class="divide-y divide-gray-100 text-xs w-40">
		<button
			class="w-full text-left p-2 hover:bg-gray-100"
			on:click={() => {
				close()
				dispatch('new', 'script')
			}}
			role="menuitem"
			tabindex="-1"
		>
			<Icon data={faCode} scale={0.8} class="mr-1" />
			Action (Script)
		</button>
		{#if trigger}
			<button
				class="w-full text-left p-2 hover:bg-gray-100"
				on:click={() => {
					close()
					dispatch('new', 'trigger')
				}}
				role="menuitem"
				tabindex="-1"
			>
				<Icon data={faBolt} scale={0.8} class="mr-1" />
				Trigger (Script)
			</button>
		{/if}
		<button
			class="w-full text-left p-2 hover:bg-gray-100"
			on:click={() => {
				close()
				dispatch('new', 'approval')
			}}
			role="menuitem"
			tabindex="-1"
		>
			<Icon data={faCheck} scale={0.8} class="mr-1" />
			Approval (Script)
		</button>
		<button
			class="w-full inline-flex gap-1 text-left p-2 hover:bg-gray-100"
			on:click={() => {
				close()
				dispatch('new', 'forloop')
			}}
			role="menuitem"
		>
			<span>
				<Repeat size={14} />
			</span>

			For Loop
		</button>

		<button
			class="w-full text-left p-2 hover:bg-gray-100"
			on:click={() => {
				close()
				dispatch('new', 'branchone')
			}}
			role="menuitem"
		>
			<Icon data={faCodeBranch} scale={0.8} class="mr-1" />
			Branch to one
		</button>

		<button
			class="w-full text-left p-2 hover:bg-gray-100"
			on:click={() => {
				close()
				dispatch('new', 'branchall')
			}}
			role="menuitem"
		>
			<Icon data={faCodeBranch} scale={0.8} class="mr-1" />
			Branch to all
		</button>

		<button
			class="w-full text-left p-2 hover:bg-gray-100"
			on:click={() => {
				close()
				dispatch('new', 'flow')
			}}
			role="menuitem"
		>
			<Icon data={faBarsStaggered} scale={0.8} class="mr-1" />
			Flow
		</button>
	</div>
</Menu>
