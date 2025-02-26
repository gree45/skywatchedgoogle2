<script lang="ts">
	import { crosspostModal, rateMovieModal } from '$lib/state/modals.svelte';
	import { settings, watchedItems } from '$lib/state/user.svelte';
	import { toast } from 'svelte-sonner';
	import Rating from '$lib/Components/Items/Rating.svelte';
	import SearchCombobox from '$lib/Components/UI/SearchCombobox.svelte';

	let rating = $state(rateMovieModal.selectedItem?.currentRating ?? 0);
	let review = $state(rateMovieModal.selectedItem?.currentReview ?? '');

	$effect(() => {
		rating = rateMovieModal.selectedItem?.currentRating ?? 0;
		review = rateMovieModal.selectedItem?.currentReview ?? '';
	});

	let sending = $state(false);

	async function saveNew() {
		const response = await fetch(`/api/rate`, {
			method: 'POST',
			body: JSON.stringify({
				rating,
				review,
				ref: rateMovieModal.selectedItem?.ref
			})
		});

		if (!response.ok) {
			toast.error('Failed to save rating');

			return;
		}
		rateMovieModal.showModal = false;

		const data = await response.json();

		if (rateMovieModal.selectedItem) {
			watchedItems.addRated({
				...rateMovieModal.selectedItem,
				rating
			});
		}

		toast.success('Rating saved');

		if (settings.crosspostEnabled && review.length > 0) {
			await new Promise((resolve) => setTimeout(resolve, 1000));

			crosspostModal.show(
				data.uri,
				review,
				rating,
				rateMovieModal?.selectedItem?.title ?? '',
				rateMovieModal?.selectedItem?.backdrop_path ?? '',
				rateMovieModal?.selectedItem?.poster_path ?? ''
			);
		}
	}

	async function saveEdit() {
		const response = await fetch(`/api/review`, {
			method: 'PUT',
			body: JSON.stringify({
				rating,
				review,
				...rateMovieModal?.selectedItem,
				uri: rateMovieModal?.selectedItem?.editUri
			})
		});

		if (!response.ok) {
			toast.error('Failed to save rating');

			return;
		}

		toast.success('Rating saved');

		rateMovieModal.showModal = false;

		await new Promise((resolve) => setTimeout(resolve, 1000));

		window.location.reload();
	}
</script>

{#if rateMovieModal.showModal}
	<div class="relative z-50" aria-labelledby="modal-title" role="dialog" aria-modal="true">
		<div
			class="fixed inset-0 bg-base-950/90 backdrop-blur-sm transition-opacity"
			onclick={() => (rateMovieModal.showModal = false)}
			aria-hidden="true"
		></div>
		<div class="pointer-events-none fixed inset-0 z-50 h-[100dvh] w-screen overflow-y-auto">
			<div class="flex h-[100dvh] items-end justify-center p-4 text-center sm:items-center sm:p-0">
				<div
					class="pointer-events-auto relative w-full transform overflow-hidden rounded-lg border border-base-800 bg-base-900 px-4 pb-4 pt-5 text-left shadow-xl transition-all sm:my-8 sm:max-w-sm sm:p-6"
				>
					<button
						class="absolute right-2 top-2 rounded-full bg-base-800/50 p-1 hover:bg-base-800/80"
						onclick={() => (rateMovieModal.showModal = false)}
					>
						<svg
							xmlns="http://www.w3.org/2000/svg"
							fill="none"
							viewBox="0 0 24 24"
							stroke-width="1.5"
							stroke="currentColor"
							class="size-4"
						>
							<path stroke-linecap="round" stroke-linejoin="round" d="M6 18 18 6M6 6l12 12" />
						</svg>

						<span class="sr-only">close</span>
					</button>

					<div>
						<h3 class="text-md mb-4 font-semibold text-base-50" id="modal-title">
							{#if rateMovieModal.selectedItem?.editUri}
								Edit review
							{:else}
								Rate and review
							{/if}
						</h3>

						{#if rateMovieModal.selectedItem?.title}
							<div class="relative flex items-center gap-4">
								<div
									class="relative z-20 aspect-[2/3] h-32 w-auto shrink-0 overflow-hidden rounded-md border border-base-800 bg-base-900/50"
								>
									{#if rateMovieModal.selectedItem?.poster_path}
										<img
											src="https://image.tmdb.org/t/p/w154{rateMovieModal.selectedItem.poster_path}"
											alt="movie poster for {rateMovieModal.selectedItem.title}"
											class="size-full object-cover object-center lg:size-full"
										/>
									{/if}
								</div>
								<h3
									class="mb-4 flex flex-col gap-2 text-xl font-semibold text-base-50"
									id="modal-title"
								>
									{rateMovieModal.selectedItem.title}

									<Rating bind:rating canChange size="size-7" />
								</h3>
								<!-- <div class="absolute right-2 top-0">
								<button
									onclick={() => (rateMovieModal.showModal = false)}
									class="rounded-full bg-base-800/50 p-1 px-2 text-xs text-base-50 hover:bg-base-800/70 hover:text-base-100"
								>
									change
								</button>
							</div> -->
							</div>
						{:else}
							<SearchCombobox />
						{/if}

						<div class="mt-4">
							<label for="comment" class="block text-xs font-medium text-base-50">review</label>
							<div class="mt-2">
								<textarea
									rows="4"
									name="comment"
									id="comment"
									bind:value={review}
									placeholder="write a review"
									class="outline-nonse block w-full rounded-lg border border-base-800 bg-base-950 px-3 py-1.5 text-base text-base-50 -outline-offset-1 placeholder:text-base-400 focus:outline focus:outline-2 focus:-outline-offset-2 focus:outline-accent-400 sm:text-sm/6"
								></textarea>
							</div>
						</div>
					</div>
					<div class="mt-5 sm:mt-6">
						<button
							onclick={async () => {
								if (rateMovieModal.selectedItem?.editUri) {
									saveEdit();
								} else {
									saveNew();
								}
							}}
							type="button"
							disabled={sending || !rateMovieModal.selectedItem?.title}
							class="inline-flex w-full justify-center rounded-md border border-accent-900 bg-accent-950/80 px-3 py-2 text-sm font-semibold text-accent-300 shadow-sm hover:bg-accent-950 focus-visible:outline focus-visible:outline-2 focus-visible:outline-offset-2 focus-visible:outline-accent-600 disabled:cursor-not-allowed disabled:opacity-50"
							>{sending
								? 'Sending...'
								: rateMovieModal.selectedItem?.editUri
									? 'Update'
									: 'Review'}</button
						>
					</div>
				</div>
			</div>
		</div>
	</div>
{/if}
