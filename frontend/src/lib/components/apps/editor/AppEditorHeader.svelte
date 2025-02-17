<script lang="ts">
	import { goto } from '$app/navigation'
	import { page } from '$app/stores'
	import { Alert, Badge, Drawer, DrawerContent } from '$lib/components/common'
	import Button from '$lib/components/common/button/Button.svelte'
	import { dirtyStore } from '$lib/components/common/confirmationModal/dirtyStore'
	import Skeleton from '$lib/components/common/skeleton/Skeleton.svelte'
	import ToggleButton from '$lib/components/common/toggleButton/ToggleButton.svelte'
	import ToggleButtonGroup from '$lib/components/common/toggleButton/ToggleButtonGroup.svelte'
	import DisplayResult from '$lib/components/DisplayResult.svelte'
	import Dropdown from '$lib/components/Dropdown.svelte'
	import FlowProgressBar from '$lib/components/flows/FlowProgressBar.svelte'
	import FlowStatusViewer from '$lib/components/FlowStatusViewer.svelte'
	import JobArgs from '$lib/components/JobArgs.svelte'
	import LogViewer from '$lib/components/LogViewer.svelte'
	import Path from '$lib/components/Path.svelte'
	import TestJobLoader from '$lib/components/TestJobLoader.svelte'
	import Toggle from '$lib/components/Toggle.svelte'
	import Tooltip from '$lib/components/Tooltip.svelte'
	import { AppService, Job, Policy } from '$lib/gen'
	import { userStore, workspaceStore } from '$lib/stores'
	import {
		faBug,
		faClipboard,
		faExternalLink,
		faFileExport,
		faGlobe,
		faSave
	} from '@fortawesome/free-solid-svg-icons'
	import {
		AlignHorizontalSpaceAround,
		Expand,
		Eye,
		Laptop2,
		Loader2,
		MoreVertical,
		Pencil,
		Smartphone
	} from 'lucide-svelte'
	import { getContext } from 'svelte'
	import { Icon } from 'svelte-awesome'
	import { Pane, Splitpanes } from 'svelte-splitpanes'
	import { appToHubUrl, classNames, copyToClipboard, sendUserToast } from '../../../utils'
	import type { App, AppComponent, AppEditorContext } from '../types'
	import AppExportButton from './AppExportButton.svelte'
	import PanelSection from './settingsPanel/common/PanelSection.svelte'

	async function hash(message) {
		const msgUint8 = new TextEncoder().encode(message) // encode as (utf-8) Uint8Array
		const hashBuffer = await crypto.subtle.digest('SHA-256', msgUint8) // hash the message
		const hashArray = Array.from(new Uint8Array(hashBuffer)) // convert buffer to byte array
		const hashHex = hashArray.map((b) => b.toString(16).padStart(2, '0')).join('') // convert bytes to hex string
		return hashHex
	}

	export let policy: Policy

	const { app, summary, mode, breakpoint, appPath, jobs, staticExporter } =
		getContext<AppEditorContext>('AppEditorContext')
	const loading = {
		publish: false,
		save: false
	}

	let newPath: string = ''
	let pathError: string | undefined = undefined

	let appExport: AppExportButton

	let saveDrawerOpen = false
	let jobsDrawerOpen = false
	let publishDrawerOpen = false

	function closeSaveDrawer() {
		saveDrawerOpen = false
	}

	function toStatic(): { app: App; summary: string } {
		const newApp: App = JSON.parse(JSON.stringify($app))
		newApp.grid.forEach((x) => {
			let c: AppComponent = x.data
			if (c.componentInput?.type == 'runnable') {
				c.componentInput.value = $staticExporter[x.id]()
			}
		})
		return { app: newApp, summary: $summary }
	}

	async function computeTriggerables() {
		const allTriggers = await Promise.all(
			$app.grid.map(async (x) => {
				let c = x.data as AppComponent
				if (c.componentInput?.type == 'runnable') {
					const staticInputs = Object.fromEntries(
						Object.entries(c.componentInput.fields ?? {})
							.filter(([k, v]) => v.type == 'static')
							.map(([k, v]) => {
								return [k, v['value']]
							})
					)
					if (c.componentInput.runnable?.type == 'runnableByName') {
						let hex = await hash(c.componentInput.runnable.inlineScript?.content)
						return [`rawscript/${hex}`, staticInputs]
					} else if (c.componentInput.runnable?.type == 'runnableByPath') {
						return [
							`${c.componentInput.runnable.runType}/${c.componentInput.runnable.path}`,
							staticInputs
						]
					}
				}
				return []
			})
		)
		policy.triggerables = Object.fromEntries(allTriggers)
		policy.on_behalf_of = `u/${$userStore?.username}`
		policy.on_behalf_of_email = $userStore?.email
	}
	async function createApp(path: string) {
		await computeTriggerables()
		try {
			const appId = await AppService.createApp({
				workspace: $workspaceStore!,
				requestBody: {
					value: $app,
					path,
					summary: $summary,
					policy
				}
			})
			goto(`/apps/edit/${appId}`)
		} catch (e) {
			sendUserToast('Error creating app', e)
		}
	}

	let secretUrl: string | undefined = undefined

	$: secretUrl == undefined && policy.execution_mode == 'anonymous' && getSecretUrl()

	async function getSecretUrl() {
		secretUrl = await AppService.getPublicSecretOfApp({
			workspace: $workspaceStore!,
			path: appPath
		})
	}

	async function setPublishState() {
		await AppService.updateApp({
			workspace: $workspaceStore!,
			path: appPath,
			requestBody: { policy }
		})
		if (policy.execution_mode == 'anonymous') {
			sendUserToast('App made visible publicly at the secret URL.')
		} else {
			sendUserToast('App made unaccessible publicly')
		}
	}

	async function save() {
		$dirtyStore = false
		if ($page.params.path == undefined) {
			saveDrawerOpen = true
			return
		}
		loading.save = true
		await computeTriggerables()
		await AppService.updateApp({
			workspace: $workspaceStore!,
			path: $page.params.path,
			requestBody: {
				value: $app!,
				summary: $summary,
				policy
			}
		})
		loading.save = false
		sendUserToast('App saved')
	}

	let selectedJobId: string | undefined = undefined
	let testJobLoader: TestJobLoader
	let job: Job | undefined = undefined
	let testIsLoading = false

	$: selectedJobId && testJobLoader?.watchJob(selectedJobId)
</script>

<TestJobLoader bind:this={testJobLoader} bind:isLoading={testIsLoading} bind:job />

<Drawer bind:open={saveDrawerOpen} size="800px">
	<DrawerContent title="Create an App" on:close={() => closeSaveDrawer()}>
		<Path
			bind:error={pathError}
			bind:path={newPath}
			initialPath=""
			namePlaceholder="my_app"
			kind="app"
		/>

		<div slot="actions">
			<Button
				startIcon={{ icon: faSave }}
				disabled={pathError != ''}
				on:click={() => createApp(newPath)}>Create app</Button
			>
		</div>
	</DrawerContent>
</Drawer>

<Drawer bind:open={jobsDrawerOpen} size="900px">
	<DrawerContent noPadding title="Debug Runs" on:close={() => (jobsDrawerOpen = false)}>
		<Splitpanes class="!overflow-visible">
			<Pane size={25}>
				<PanelSection title="Past Runs" smallPadding>
					<div class="flex flex-col gap-2 w-full">
						{#if $jobs.length > 0}
							<div class="flex gap-2 flex-col ">
								{#each $jobs ?? [] as { job, component } (job)}
									<!-- svelte-ignore a11y-click-events-have-key-events -->
									<div
										class="{classNames(
											'border flex gap-1 truncate justify-between flex-row w-full items-center p-2 rounded-md cursor-pointer hover:bg-blue-50 hover:text-blue-400',
											selectedJobId == job ? 'bg-blue-100 text-blue-600' : ''
										)},"
										on:click={() => (selectedJobId = job)}
									>
										<span class="text-xs truncate">{job}</span>
										<Badge color="dark-indigo">{component}</Badge>
									</div>
								{/each}
							</div>
						{:else}
							<div class="text-sm text-gray-500">No items</div>
						{/if}
					</div>
				</PanelSection>
			</Pane>
			<Pane size={75}>
				<div class="h-full w-full overflow-hidden">
					{#if selectedJobId}
						{#if !job}
							<Skeleton layout={[[40]]} />
						{:else}
							<div class="flex flex-col h-full w-full gap-4 mb-4">
								{#if job?.['running']}
									<div class="flex flex-row-reverse w-full"
										><Button
											color="red"
											variant="border"
											on:click={() => testJobLoader?.cancelJob()}>Cancel</Button
										>
									</div>
								{/if}
								<div class="p-2">
									<JobArgs args={job?.args} />
								</div>
								{#if job?.job_kind !== 'flow' && job?.job_kind !== 'flowpreview'}
									<Splitpanes horizontal class="grow border w-full">
										<Pane size={50} minSize={10}>
											<LogViewer content={job?.logs} isLoading={testIsLoading} />
										</Pane>
										<Pane size={50} minSize={10} class="text-sm text-gray-600">
											{#if job != undefined && 'result' in job && job.result != undefined}
												<pre class="overflow-x-auto break-words relative h-full px-2"
													><DisplayResult result={job.result} /></pre
												>
											{:else if testIsLoading}
												<div class="p-2"><Loader2 class="animate-spin" /> </div>
											{/if}
										</Pane>
									</Splitpanes>
								{:else}
									<div class="mt-10" />
									<FlowProgressBar {job} class="py-4" />
									<div class="w-full mt-10 mb-20">
										<FlowStatusViewer
											jobId={job.id}
											on:jobsLoaded={({ detail }) => {
												job = detail
											}}
										/>
									</div>
								{/if}
							</div>
						{/if}
					{:else}
						<div class="text-sm p-2 text-gray-500">Select a job to see its details</div>
					{/if}
				</div>
			</Pane>
		</Splitpanes>
	</DrawerContent>
</Drawer>

<Drawer bind:open={publishDrawerOpen} size="800px">
	<DrawerContent title="Publish an App" on:close={() => (publishDrawerOpen = false)}>
		{#if appPath == ''}
			<Alert title="Require saving" type="error">
				Save this app once before you can publish it
			</Alert>
		{:else}
			<Alert title="App executed on behalf of publisher">
				A viewer of the app will execute the runnables of the app on behalf of the publisher
				avoiding the risk that a resource or script would not be available to the viewer. To
				guarantee tight security, a policy is computed at time of saving of the app which only allow
				the scripts/flows referred to in the app to be called on behalf of. Furthermore, static
				parameters are not overridable. Hence, users will only be able to use the app as intended by
				the publisher without risk for leaking resources not used in the app.
			</Alert>
			<div class="mt-4" />
			<Toggle
				options={{
					left: `Require read-access`,
					right: `Publish publicly for anyone knowing the secret url`
				}}
				checked={policy.execution_mode == 'anonymous'}
				on:change={(e) => {
					policy.execution_mode = e.detail
						? Policy.execution_mode.ANONYMOUS
						: Policy.execution_mode.PUBLISHER
					setPublishState()
				}}
			/>

			{#if policy.execution_mode == 'anonymous' && secretUrl}
				{@const url = `${$page.url.hostname}/public/${$workspaceStore}/${secretUrl}`}
				{@const href = $page.url.protocol + '//' + url}
				<div class="my-6 box">
					Public url:
					<a
						on:click={(e) => {
							e.preventDefault()
							copyToClipboard(href)
						}}
						{href}
						class="whitespace-nowrap text-ellipsis overflow-hidden mr-1"
					>
						{url}
						<span class="text-gray-700 ml-2">
							<Icon data={faClipboard} />
						</span>
					</a>
				</div>

				<Alert type="info" title="Only show latest saved app">
					Once made public, you will still need to save the app to make visible the latest changes
				</Alert>
			{/if}
		{/if}

		<div />
	</DrawerContent>
</Drawer>

<div
	class="border-b flex flex-row justify-between py-1 gap-4  gap-y-2 px-4 items-center overflow-y-visible"
>
	<div class="min-w-64 w-64">
		<input type="text" placeholder="App summary" class="text-sm w-full" bind:value={$summary} />
	</div>
	<div class="flex gap-4 items-center grow justify-center">
		<div>
			<ToggleButtonGroup bind:selected={$mode}>
				<ToggleButton position="left" value="dnd" size="xs">
					<div class="inline-flex gap-1 items-center">
						<Pencil size={14} />
						<span class="hidden md:inline">Editor</span>
					</div>
				</ToggleButton>
				<ToggleButton position="right" value="preview" size="xs">
					<div class="inline-flex gap-1 items-center">
						<Eye size={14} /> <span class="hidden md:inline">Preview</span>
					</div>
				</ToggleButton>
			</ToggleButtonGroup>
		</div>
		<div>
			<ToggleButtonGroup bind:selected={$breakpoint}>
				<ToggleButton position="left" value="sm" size="xs">
					<Smartphone size={14} />
				</ToggleButton>
				<ToggleButton position="right" value="lg" size="xs">
					<Laptop2 size={14} />
				</ToggleButton>
			</ToggleButtonGroup>
		</div>

		<div class="hidden lg:block">
			<ToggleButtonGroup bind:selected={$app.fullscreen}>
				<ToggleButton position="left" value={false} size="xs">
					<div class="flex gap-1 justify-start">
						<AlignHorizontalSpaceAround size={14} />
						<Tooltip>
							The max width is 1168px and the content stay centered instead of taking the full page
							width
						</Tooltip>
					</div>
				</ToggleButton>
				<ToggleButton position="right" value={true} size="xs">
					<Expand size={14} />
				</ToggleButton>
			</ToggleButtonGroup>
		</div>
	</div>
	<div class="flex flex-row grow gap-2 justify-end items-center overflow-visible">
		<Dropdown
			placement="bottom-end"
			btnClasses="!text-gray-700 !bg-transparent hover:!bg-gray-400/20 !p-[6px] hidden lg:block"
			dropdownItems={[
				{
					displayName: 'JSON',
					icon: faFileExport,
					action: () => {
						appExport.open()
					}
				},
				{
					displayName: 'Publish to Hub',
					icon: faGlobe,
					action: () => {
						const url = appToHubUrl(toStatic())
						window.open(url.toString(), '_blank')
					}
				}
			]}
		>
			<MoreVertical size={20} />
		</Dropdown>
		<span class="hidden md:inline">
			<Button
				on:click={() => (jobsDrawerOpen = true)}
				color="light"
				size="xs"
				variant="border"
				startIcon={{ icon: faBug }}
			>
				<span class="hidden md:inline">Debug Runs</span>
			</Button>
		</span>
		<AppExportButton bind:this={appExport} app={$app} />
		<Button
			on:click={() => (publishDrawerOpen = true)}
			color="light"
			size="xs"
			variant="border"
			startIcon={{ icon: faExternalLink }}
		>
			<span class="hidden md:inline">Publish</span>
		</Button>
		<Button
			loading={loading.save}
			startIcon={{ icon: faSave }}
			on:click={save}
			color="dark"
			size="xs"
		>
			<span class="hidden md:inline">Save</span>
		</Button>
	</div>
</div>
