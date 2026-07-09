<template>
	<div
		ref="wrapperContainer"
		class="group relative rounded-xl overflow-hidden bg-bg-raised border border-border/10 transition-all duration-300 hover:scale-[1.02] hover:shadow-xl hover:border-border/30 w-full cursor-pointer"
		@mouseenter="isHovered = true"
		@mouseleave="isHovered = false"
	>
		<!-- Floating Action Buttons -->
		<div
			v-if="loaded"
			class="absolute top-3 right-3 flex gap-2 transition-all duration-300 z-10"
			:class="{
				'opacity-0 translate-y-[-4px] pointer-events-none': !isHovered,
				'opacity-100 translate-y-0': isHovered,
			}"
		>
			<ButtonStyled
				v-tooltip="'Copy'"
				circular
				icon-only
				class="!bg-black/60 hover:!bg-black/85 border-none shadow-md backdrop-blur-sm transition-colors"
				@click.stop="copyImageToClipboard"
			>
				<Button><ClipboardCopyIcon class="text-white size-4" /></Button>
			</ButtonStyled>
			<ButtonStyled
				v-tooltip="'View in folder'"
				circular
				icon-only
				class="!bg-black/60 hover:!bg-black/85 border-none shadow-md backdrop-blur-sm transition-colors"
				@click.stop="viewInFolder"
			>
				<Button><ExternalIcon class="text-white size-4" /></Button>
			</ButtonStyled>
			<ButtonStyled
				v-tooltip="'Delete'"
				circular
				icon-only
				class="!bg-black/60 hover:!bg-red-600/90 border-none shadow-md backdrop-blur-sm transition-colors"
				@click.stop="deleteScreenshot"
			>
				<Button><TrashIcon class="text-white size-4" /></Button>
			</ButtonStyled>
		</div>

		<!-- Aspect ratio image container -->
		<div class="aspect-video bg-bg-raised overflow-hidden">
			<div v-if="!loaded" class="absolute inset-0 skeleton"></div>
			<img
				v-else
				:alt="getScreenshotFileName(screenshot.path)"
				:src="blobUrl"
				class="w-full h-full object-cover transition-all duration-500 group-hover:scale-105"
				:class="{ 'opacity-0': !loaded, 'opacity-100': loaded }"
				@load="onLoad"
				@click="
					imagePreviewModal.show(blobUrl, getScreenshotFileName(screenshot.path), {
						...screenshot,
						title: getScreenshotFileName(screenshot.path),
						description: `Taken on ${dayjs(screenshot.creation_date).format('MMMM Do, YYYY')}`,
					})
				"
			/>
		</div>

		<!-- Info Overlay -->
		<div
			class="absolute inset-x-0 bottom-0 p-3 bg-gradient-to-t from-black/90 via-black/40 to-transparent pt-8 opacity-0 group-hover:opacity-100 transition-opacity duration-300 pointer-events-none"
		>
			<p class="text-sm font-semibold text-white truncate">
				{{ getScreenshotFileName(screenshot.path) }}
			</p>
			<p class="text-xs text-white/70">{{ dayjs(screenshot.creation_date).format('h:mm A') }}</p>
		</div>
	</div>
</template>

<script lang="ts" setup>
import { ClipboardCopyIcon, ExternalIcon, TrashIcon } from '@modrinth/assets'
import type { ImagePreviewModal } from '@modrinth/ui'
import { Button, ButtonStyled, injectNotificationManager } from '@modrinth/ui'
import dayjs from 'dayjs'
import { onBeforeUnmount, onMounted, ref } from 'vue'

import {
	deleteProfileScreenshot,
	getScreenshotData,
	getScreenshotFileName,
	openProfileScreenshot,
	type Screenshot,
} from '@/helpers/screenshots'

const props = defineProps<{
	screenshot: Screenshot
	profilePath: string
	imagePreviewModal: typeof ImagePreviewModal
}>()
const emit = defineEmits(['deleted'])
const { addNotification } = injectNotificationManager()

const loaded = ref(false)
const blobUrl = ref<string>('')
const isHovered = ref(false)
const wrapperContainer = ref<HTMLElement | null>(null)
let observer: IntersectionObserver | null = null

function onLoad() {
	loaded.value = true
}

async function loadData(): Promise<void> {
	try {
		const rawData = await getScreenshotData(props.profilePath, props.screenshot)
		if (!rawData) {
			addNotification({
				title: 'Failed to load screenshot:',
				text: props.screenshot.path,
				type: 'error',
			})
			return
		}

		const bytes = new Uint8Array(rawData)
		const blob = new Blob([bytes], { type: 'image/png' })

		blobUrl.value = URL.createObjectURL(blob)
		loaded.value = true
	} catch (err: unknown) {
		addNotification({
			title: 'Error fetching screenshot',
			text: err instanceof Error ? err.message : String(err),
			type: 'error',
		})
	}
}

onMounted(() => {
	observer = new IntersectionObserver(
		(entries) => {
			for (const entry of entries) {
				if (entry.isIntersecting) {
					loadData()
					if (observer && wrapperContainer.value) {
						observer.unobserve(wrapperContainer.value)
					}
				}
			}
		},
		{ rootMargin: '100px', threshold: 0.1 },
	)
	if (wrapperContainer.value) observer.observe(wrapperContainer.value)
})

onBeforeUnmount(() => {
	if (observer && wrapperContainer.value) {
		observer.unobserve(wrapperContainer.value)
	}

	if (blobUrl.value) {
		URL.revokeObjectURL(blobUrl.value)
	}
})

async function copyImageToClipboard() {
	try {
		const resp = await fetch(blobUrl.value)
		const blob = await resp.blob()
		await navigator.clipboard.write([new ClipboardItem({ [blob.type]: blob })])
		addNotification({
			title: 'Copied to clipboard',
			text: 'The screenshot has been copied successfully.',
			type: 'success',
		})
	} catch (error: unknown) {
		addNotification({
			title: 'Copy failed',
			text: error instanceof Error ? error.message : String(error),
			type: 'error',
		})
	}
}

async function deleteScreenshot() {
	try {
		const ok = await deleteProfileScreenshot(props.profilePath, props.screenshot)
		if (!ok) throw new Error('Delete returned false')
		addNotification({ title: 'Successfully deleted screenshot', type: 'success' })
		emit('deleted')
	} catch (err: unknown) {
		addNotification({
			title: 'Error deleting screenshot',
			text: err instanceof Error ? err.message : String(err),
			type: 'error',
		})
	}
}

async function viewInFolder() {
	const ok = await openProfileScreenshot(props.profilePath, props.screenshot)
	if (!ok) {
		addNotification({ title: 'Unable to open screenshot in folder.', type: 'error' })
	}
}
</script>

<style scoped>
.skeleton {
	background: linear-gradient(
		90deg,
		var(--color-bg) 25%,
		var(--color-raised-bg) 50%,
		var(--color-bg) 75%
	);
	background-size: 200% 100%;
	animation: wave 1500ms infinite linear;
}

@keyframes wave {
	0% {
		background-position: -200% 0;
	}
	100% {
		background-position: 200% 0;
	}
}
</style>
