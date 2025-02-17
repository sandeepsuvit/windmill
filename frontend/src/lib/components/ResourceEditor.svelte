<script lang="ts">
	import { type Resource, ResourceService, type ResourceType, VariableService } from '$lib/gen'
	import { canWrite, emptyString, sendUserToast } from '$lib/utils'
	import { createEventDispatcher } from 'svelte'
	import type { Schema } from '$lib/common'
	import Path from './Path.svelte'
	import Required from './Required.svelte'
	import { Alert, Button, Drawer, Skeleton } from './common'

	import { userStore, workspaceStore } from '$lib/stores'
	import DrawerContent from './common/drawer/DrawerContent.svelte'
	import autosize from 'svelte-autosize'
	import SimpleEditor from './SimpleEditor.svelte'
	import { faSave } from '@fortawesome/free-solid-svg-icons'
	import SchemaForm from './SchemaForm.svelte'

	let path = ''
	let initialPath = ''

	let resourceToEdit: Resource | undefined

	let description: string = ''
	let DESCRIPTION_PLACEHOLDER = `Describe what this resource is for`
	let selectedResourceType: string | undefined
	let resourceSchema: Schema | undefined
	let args: Record<string, any> = {}
	let can_write = true
	let loadingSchema = false

	let drawer: Drawer

	let rawCode: string | undefined = undefined

	const dispatch = createEventDispatcher()

	export async function initEdit(p: string): Promise<void> {
		initialPath = p
		path = p
		resourceToEdit = undefined
		resourceSchema = undefined
		loadingSchema = true
		drawer.openDrawer?.()
		resourceToEdit = await ResourceService.getResource({ workspace: $workspaceStore!, path: p })
		description = resourceToEdit!.description ?? ''
		selectedResourceType = resourceToEdit!.resource_type
		loadResourceType()
		args = resourceToEdit!.value
		can_write =
			resourceToEdit.workspace_id == $workspaceStore &&
			canWrite(p, resourceToEdit.extra_perms ?? {}, $userStore)
	}

	async function editResource(): Promise<void> {
		if (resourceToEdit) {
			await ResourceService.updateResource({
				workspace: $workspaceStore!,
				path: resourceToEdit.path,
				requestBody: { path, value: args, description }
			})
			sendUserToast(`Updated resource at ${path}`)
			dispatch('refresh')
			drawer.closeDrawer?.()
		} else {
			throw Error('Cannot edit undefined resourceToEdit')
		}
	}

	async function loadResourceType(): Promise<void> {
		if (selectedResourceType) {
			try {
				const resourceType = await ResourceService.getResourceType({
					workspace: $workspaceStore!,
					path: selectedResourceType
				})

				if (resourceType.schema) {
					resourceSchema = resourceType.schema as Schema
				}
			} catch (err) {
				resourceSchema = undefined
				loadingSchema = false
				rawCode = JSON.stringify(args, null, 2)
			}
		} else {
			sendUserToast(`ResourceType cannot be undefined.`, true)
		}
		loadingSchema = false
	}

	let isValid = true
	let jsonError = ''

	$: rawCode && parseJson()

	function parseJson() {
		try {
			args = JSON.parse(rawCode ?? '')
			jsonError = ''
		} catch (e) {
			jsonError = e.message
		}
	}
</script>

<Drawer bind:this={drawer} size="800px">
	<DrawerContent
		title={resourceToEdit ? 'Edit ' + resourceToEdit.path : 'Add a resource'}
		on:close={drawer.closeDrawer}
	>
		<div>
			<div class="flex flex-col gap-3 py-3  text-gray-700">
				<div>
					{#if !can_write}
						<div class="m-2">
							<Alert type="warning" title="Only read access"
								>You only have read access to this resource and cannot edit it</Alert
							>
						</div>
					{/if}
					<Path
						disabled={!can_write}
						bind:path
						{initialPath}
						namePlaceholder="my_resource"
						kind="resource"
					/>
				</div>
				<h3>Description <Required required={false} /> </h3>
				<textarea
					type="text"
					disabled={!can_write}
					use:autosize
					bind:value={description}
					placeholder={DESCRIPTION_PLACEHOLDER}
				/>

				<h3 class="mt-4">Value</h3>
				<div class="text-sm">
					{#if loadingSchema}
						<Skeleton layout={[[4]]} />
					{:else if resourceSchema && resourceSchema?.properties}
						<SchemaForm
							disabled={!can_write}
							compact
							schema={resourceSchema}
							bind:args
							bind:isValid
						/>
					{:else if !can_write}
						<input type="text" disabled value={rawCode} />
					{:else}
						<p class="italic text-gray-500 text-xs mb-4"
							>No corresponding resource type found in your workspace for {selectedResourceType}.
							Define the value in JSON directly</p
						>
						{#if !emptyString(jsonError)}<span
								class="text-red-400 text-xs mb-1 flex flex-row-reverse">{jsonError}</span
							>{:else}<div class="py-2" />{/if}
						<div class="h-full w-full border p-1 rounded">
							<SimpleEditor
								autoHeight
								class="editor"
								lang="json"
								bind:code={rawCode}
								fixedOverflowWidgets={false}
							/>
						</div>
					{/if}
				</div>
			</div>
		</div>
		<svelte:fragment slot="actions">
			<Button
				startIcon={{ icon: faSave }}
				on:click={editResource}
				disabled={!can_write || !isValid || jsonError != ''}
			>
				Save
			</Button>
		</svelte:fragment>
	</DrawerContent>
</Drawer>
