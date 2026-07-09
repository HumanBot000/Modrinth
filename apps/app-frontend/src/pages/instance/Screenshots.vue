<script setup lang="ts">
import { FolderOpenIcon } from '@modrinth/assets'
import { ButtonStyled, EmptyState, injectNotificationManager } from '@modrinth/ui'
import type {
	GalleryEntry,
	NavigationFunction,
	OpenExternallyFunction,
	OpenFileFunction,
} from '@modrinth/ui/src/components/modal/ImagePreviewModal.vue'
import ImagePreviewModal from '@modrinth/ui/src/components/modal/ImagePreviewModal.vue'
import type { Version } from '@modrinth/utils'
import { mkdir } from '@tauri-apps/plugin-fs'
import dayjs from 'dayjs'
import advancedFormat from 'dayjs/plugin/advancedFormat.js'
import { computed, onUnmounted, ref } from 'vue'

import type ContextMenu from '@/components/ui/ContextMenu.vue'
import ScreenshotCard from '@/components/ui/ScreenshotCard.vue'
import { instance_listener } from '@/helpers/events'
import { get_full_path } from '@/helpers/instance'
import type { Screenshot } from '@/helpers/screenshots.ts'
import {
	getAllProfileScreenshots,
	getScreenshotData,
	getScreenshotFileName,
	openProfileScreenshot,
	openScreenshotFile,
} from '@/helpers/screenshots.ts'
import type { GameInstance, InstanceEvent } from '@/helpers/types'
import { openPath } from '@/helpers/utils'

dayjs.extend(advancedFormat)

const props = defineProps<{
	instance: GameInstance
	options: InstanceType<typeof ContextMenu> | null
	offline: boolean
	playing: boolean
	versions: Version[]
	installed: boolean
}>()

const { addNotification } = injectNotificationManager()
const screenshots = ref<Screenshot[]>((await getAllProfileScreenshots(props.instance.id)) ?? [])
const imagePreviewModal = ref<typeof ImagePreviewModal>()

async function openScreenshotsFolder() {
	const fullPath = await get_full_path(props.instance.id)
	const screenshotsPath = `${fullPath}/screenshots`
	try {
		await mkdir(screenshotsPath)
	} catch {
		// Ignore if folder already exists
	}
	await openPath(screenshotsPath)
}

function groupAndSortByDate(items: Screenshot[]): readonly [string, Screenshot[]][] {
	const todayTS = dayjs().startOf('day').valueOf()
	const yesterdayTS = dayjs().subtract(1, 'day').startOf('day').valueOf()

	const groups = new Map<number, Screenshot[]>()
	for (const shot of items) {
		const ts = dayjs(shot.creation_date).startOf('day').valueOf()
		const bucket = groups.get(ts)
		if (bucket) bucket.push(shot)
		else groups.set(ts, [shot])
	}

	const sortedTS = Array.from(groups.keys()).sort((a, b) => b - a)
	return sortedTS.map((ts) => {
		let label: string
		if (ts === todayTS) label = 'Today'
		else if (ts === yesterdayTS) label = 'Yesterday'
		else label = dayjs(ts).format('MMMM Do, YYYY')

		return [label, groups.get(ts)!] as const
	})
}

function markDeleted(s: Screenshot): void {
	screenshots.value = screenshots.value.filter((shot) => shot.path !== s.path)
}

async function navigateScreenshot(screenshot: Screenshot, offset: number): Promise<GalleryEntry> {
	const list = screenshots.value
	const idx = list.findIndex((s) => s.path === screenshot.path)
	const newIdx = (idx + offset + list.length) % list.length
	const next = list[newIdx]

	const rawData = await getScreenshotData(props.instance.id, next)
	let src = ''
	if (rawData) {
		const bytes = new Uint8Array(rawData)
		const blob = new Blob([bytes], { type: 'image/png' })
		src = await new Promise<string>((resolve, reject) => {
			const reader = new FileReader()
			reader.onload = () => resolve(reader.result as string)
			reader.onerror = () => reject(new Error('Failed to read screenshot blob'))
			reader.readAsDataURL(blob)
		})
	}

	return {
		src,
		alt: getScreenshotFileName(next.path),
		key: {
			...next,
			title: getScreenshotFileName(next.path),
			description: `Taken on ${dayjs(next.creation_date).format('MMMM Do, YYYY')}`,
		},
	}
}

const viewNextScreenshot: NavigationFunction = ((s) =>
	navigateScreenshot(s, 1)) as NavigationFunction

const viewPreviousScreenshot: NavigationFunction = ((s) =>
	navigateScreenshot(s, -1)) as NavigationFunction

const openExternally: OpenExternallyFunction = (async (src: string, screenshot: Screenshot) => {
	const result = await openProfileScreenshot(props.instance.id, screenshot)
	if (!result) {
		addNotification({
			title: 'Unable to open screenshot in folder.',
			type: 'error',
		})
	}
}) as OpenExternallyFunction

const openFile: OpenFileFunction = (async (src: string, screenshot: Screenshot) => {
	const result = await openScreenshotFile(props.instance.id, screenshot)
	if (!result) {
		addNotification({
			title: 'Unable to open screenshot.',
			type: 'error',
		})
	}
}) as OpenFileFunction

const screenshotsByDate = computed(() => groupAndSortByDate(screenshots.value))
const hasToday = computed(() => screenshotsByDate.value.some(([label]) => label === 'Today'))

const unlistenProfile = await instance_listener(async (e: InstanceEvent) => {
	if (e.instance_id !== props.instance.id) return

	console.info(`Handling profile event '${e.event}' for profile: ${e.instance_id}`)

	if (e.event === 'screenshot_changed') {
		const exists = screenshots.value.some((shot) => shot.path === e.screenshot.path)
		if (e.file_exists && !exists) {
			screenshots.value = [e.screenshot, ...screenshots.value]
		}

		if (!e.file_exists && exists) {
			screenshots.value = screenshots.value.filter((shot) => shot.path !== e.screenshot.path)
		}
	}
})

onUnmounted(() => {
	unlistenProfile()
})
</script>

<template>
	<div class="h-full flex flex-col">
		<ImagePreviewModal
			ref="imagePreviewModal"
			:next="viewNextScreenshot"
			:prev="viewPreviousScreenshot"
			:open-externally="openExternally"
			:open-externally-tooltip="'Open in containing folder'"
			:open-file="openFile"
			:open-file-tooltip="'Open in default system viewer'"
		/>

		<EmptyState
			v-if="!screenshots.length"
			type="empty-inbox"
			heading="No screenshots yet"
			description="Screenshots taken in-game will appear here"
			class="my-auto"
		>
			<template #actions>
				<ButtonStyled type="outlined">
					<button class="!h-10 flex items-center gap-2" @click="openScreenshotsFolder">
						<FolderOpenIcon class="size-5" />
						Open screenshots folder
					</button>
				</ButtonStyled>
			</template>
		</EmptyState>

		<div v-else class="space-y-10 py-2">
			<!-- Only show Today helper message if user has screenshots but none are from today -->
			<div v-if="!hasToday" class="space-y-4">
				<div class="flex items-center gap-3">
					<h3 class="text-xl font-bold tracking-tight text-primary">Today</h3>
					<span
						class="px-2.5 py-0.5 text-xs font-semibold bg-bg-raised border border-border/10 text-secondary rounded-full shadow-sm"
					>
						0
					</span>
					<div class="flex-1 h-px bg-border/10"></div>
				</div>
				<p class="text-sm text-gray-500 dark:text-gray-400 italic pl-1">
					You haven't taken any screenshots today.
				</p>
			</div>

			<div v-for="[date, shots] in screenshotsByDate" :key="date" class="space-y-4">
				<div class="flex items-center gap-3">
					<h3 class="text-xl font-bold tracking-tight text-primary">
						{{ date }}
					</h3>
					<span
						class="px-2.5 py-0.5 text-xs font-semibold bg-bg-raised border border-border/10 text-secondary rounded-full shadow-sm"
					>
						{{ shots.length }}
					</span>
					<div class="flex-1 h-px bg-border/10"></div>
				</div>
				<div class="grid grid-cols-1 sm:grid-cols-2 md:grid-cols-3 xl:grid-cols-4 gap-6 pt-1">
					<ScreenshotCard
						v-for="s in shots"
						:key="s.path"
						:screenshot="s"
						:profile-path="instance.id"
						:image-preview-modal="imagePreviewModal!"
						@deleted="markDeleted(s)"
					/>
				</div>
			</div>
		</div>
	</div>
</template>
