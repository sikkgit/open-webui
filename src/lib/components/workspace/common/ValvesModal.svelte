<script lang="ts">
	import { toast } from 'svelte-sonner';
	import { createEventDispatcher } from 'svelte';
	import { onMount, getContext } from 'svelte';
	import { addUser } from '$lib/apis/auths';

	import Modal from '../../common/Modal.svelte';
	import {
		getFunctionValvesById,
		getFunctionValvesSpecById,
		updateFunctionValvesById
	} from '$lib/apis/functions';
	import { getToolValvesById, getToolValvesSpecById, updateToolValvesById } from '$lib/apis/tools';
	import Spinner from '../../common/Spinner.svelte';
	import Switch from '$lib/components/common/Switch.svelte';

	const i18n = getContext('i18n');
	const dispatch = createEventDispatcher();

	export let show = false;

	export let type = 'tool';
	export let id = null;

	let saving = false;
	let loading = false;

	let valvesSpec = null;
	let valves = {};

	const submitHandler = async () => {
		saving = true;

		if (valvesSpec) {
			// Convert string to array
			for (const property in valvesSpec.properties) {
				if (valvesSpec.properties[property]?.type === 'array') {
					valves[property] = (valves[property] ?? '').split(',').map((v) => v.trim());
				}
			}

			let res = null;

			if (type === 'tool') {
				res = await updateToolValvesById(localStorage.token, id, valves).catch((error) => {
					toast.error(error);
				});
			} else if (type === 'function') {
				res = await updateFunctionValvesById(localStorage.token, id, valves).catch((error) => {
					toast.error(error);
				});
			}

			if (res) {
				toast.success('Valves updated successfully');
				dispatch('save');
			}
		}

		saving = false;
	};

	const initHandler = async () => {
		loading = true;
		valves = {};
		valvesSpec = null;

		if (type === 'tool') {
			valves = await getToolValvesById(localStorage.token, id);
			valvesSpec = await getToolValvesSpecById(localStorage.token, id);
		} else if (type === 'function') {
			valves = await getFunctionValvesById(localStorage.token, id);
			valvesSpec = await getFunctionValvesSpecById(localStorage.token, id);
		}

		if (!valves) {
			valves = {};
		}

		if (valvesSpec) {
			// Convert array to string
			for (const property in valvesSpec.properties) {
				if (valvesSpec.properties[property]?.type === 'array') {
					valves[property] = (valves[property] ?? []).join(',');
				}
			}
		}

		loading = false;
	};

	$: if (show) {
		initHandler();
	}
</script>

<Modal size="sm" bind:show>
	<div>
		<div class=" flex justify-between dark:text-gray-300 px-5 pt-4 pb-2">
			<div class=" text-lg font-medium self-center">{$i18n.t('Valves')}</div>
			<button
				class="self-center"
				on:click={() => {
					show = false;
				}}
			>
				<svg
					xmlns="http://www.w3.org/2000/svg"
					viewBox="0 0 20 20"
					fill="currentColor"
					class="w-5 h-5"
				>
					<path
						d="M6.28 5.22a.75.75 0 00-1.06 1.06L8.94 10l-3.72 3.72a.75.75 0 101.06 1.06L10 11.06l3.72 3.72a.75.75 0 101.06-1.06L11.06 10l3.72-3.72a.75.75 0 00-1.06-1.06L10 8.94 6.28 5.22z"
					/>
				</svg>
			</button>
		</div>

		<div class="flex flex-col md:flex-row w-full px-5 pb-4 md:space-x-4 dark:text-gray-200">
			<div class=" flex flex-col w-full sm:flex-row sm:justify-center sm:space-x-6">
				<form
					class="flex flex-col w-full"
					on:submit|preventDefault={() => {
						submitHandler();
					}}
				>
					<div class="px-1">
						{#if !loading}
							{#if valvesSpec}
								{#each Object.keys(valvesSpec.properties) as property, idx}
									<div class=" py-0.5 w-full justify-between">
										<div class="flex w-full justify-between">
											<div class=" self-center text-xs font-medium">
												{valvesSpec.properties[property].title}

												{#if (valvesSpec?.required ?? []).includes(property)}
													<span class=" text-gray-500">*required</span>
												{/if}
											</div>

											<button
												class="p-1 px-3 text-xs flex rounded transition"
												type="button"
												on:click={() => {
													valves[property] =
														(valves[property] ?? null) === null
															? valvesSpec.properties[property]?.default ?? ''
															: null;
												}}
											>
												{#if (valves[property] ?? null) === null}
													<span class="ml-2 self-center">
														{#if (valvesSpec?.required ?? []).includes(property)}
															{$i18n.t('None')}
														{:else}
															{$i18n.t('Default')}
														{/if}
													</span>
												{:else}
													<span class="ml-2 self-center"> {$i18n.t('Custom')} </span>
												{/if}
											</button>
										</div>

										{#if (valves[property] ?? null) !== null}
											<!-- {valves[property]} -->
											<div class="flex mt-0.5 mb-1.5 space-x-2">
												<div class=" flex-1">
													{#if valvesSpec.properties[property]?.enum ?? null}
														<select
															class="w-full rounded-lg py-2 px-4 text-sm dark:text-gray-300 dark:bg-gray-850 outline-none"
															bind:value={valves[property]}
														>
															{#each valvesSpec.properties[property].enum as option}
																<option value={option} selected={option === valves[property]}>
																	{option}
																</option>
															{/each}
														</select>
													{:else if (valvesSpec.properties[property]?.type ?? null) === 'boolean'}
														<div class="flex justify-between items-center">
															<div class="text-xs text-gray-500">
																{valves[property] ? 'Enabled' : 'Disabled'}
															</div>

															<div class=" pr-2">
																<Switch bind:state={valves[property]} />
															</div>
														</div>
													{:else}
														<input
															class="w-full rounded-lg py-2 px-4 text-sm dark:text-gray-300 dark:bg-gray-850 outline-none"
															type="text"
															placeholder={valvesSpec.properties[property].title}
															bind:value={valves[property]}
															autocomplete="off"
															required
														/>
													{/if}
												</div>
											</div>
										{/if}

										{#if (valvesSpec.properties[property]?.description ?? null) !== null}
											<div class="text-xs text-gray-500">
												{valvesSpec.properties[property].description}
											</div>
										{/if}
									</div>
								{/each}
							{:else}
								<div class="text-sm">No valves</div>
							{/if}
						{:else}
							<Spinner className="size-5" />
						{/if}
					</div>

					<div class="flex justify-end pt-3 text-sm font-medium">
						<button
							class=" px-4 py-2 bg-emerald-700 hover:bg-emerald-800 text-gray-100 transition rounded-lg flex flex-row space-x-1 items-center {saving
								? ' cursor-not-allowed'
								: ''}"
							type="submit"
							disabled={saving}
						>
							{$i18n.t('Save')}

							{#if saving}
								<div class="ml-2 self-center">
									<svg
										class=" w-4 h-4"
										viewBox="0 0 24 24"
										fill="currentColor"
										xmlns="http://www.w3.org/2000/svg"
										><style>
											.spinner_ajPY {
												transform-origin: center;
												animation: spinner_AtaB 0.75s infinite linear;
											}
											@keyframes spinner_AtaB {
												100% {
													transform: rotate(360deg);
												}
											}
										</style><path
											d="M12,1A11,11,0,1,0,23,12,11,11,0,0,0,12,1Zm0,19a8,8,0,1,1,8-8A8,8,0,0,1,12,20Z"
											opacity=".25"
										/><path
											d="M10.14,1.16a11,11,0,0,0-9,8.92A1.59,1.59,0,0,0,2.46,12,1.52,1.52,0,0,0,4.11,10.7a8,8,0,0,1,6.66-6.61A1.42,1.42,0,0,0,12,2.69h0A1.57,1.57,0,0,0,10.14,1.16Z"
											class="spinner_ajPY"
										/></svg
									>
								</div>
							{/if}
						</button>
					</div>
				</form>
			</div>
		</div>
	</div>
</Modal>

<style>
	input::-webkit-outer-spin-button,
	input::-webkit-inner-spin-button {
		/* display: none; <- Crashes Chrome on hover */
		-webkit-appearance: none;
		margin: 0; /* <-- Apparently some margin are still there even though it's hidden */
	}

	.tabs::-webkit-scrollbar {
		display: none; /* for Chrome, Safari and Opera */
	}

	.tabs {
		-ms-overflow-style: none; /* IE and Edge */
		scrollbar-width: none; /* Firefox */
	}

	input[type='number'] {
		-moz-appearance: textfield; /* Firefox */
	}
</style>
