<template>
	<div ref="videoContainer" class="video-block group relative">
		<div v-if="isYoutube || isRutube">
			<iframe
				:src="fileURL"
				class="rounded-lg border border-gray-100 w-full"
				:style="{ minHeight: '500px' }"
				frameborder="0"
				allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
				allowfullscreen
			></iframe>
		</div>
		<video
			v-else
			@timeupdate="updateTime"
			@ended="videoEnded"
			@click="togglePlay"
			oncontextmenu="return false"
			class="rounded-lg border border-gray-100 group cursor-pointer"
			ref="videoRef"
		>
			<source :src="fileURL" :type="type" />
		</video>
		<div
			v-if="!isYoutube && !isRutube"
			class="flex items-center space-x-2 bg-surface-gray-3 rounded-md p-0.5 absolute bottom-3 w-[98%] left-0 right-0 mx-auto invisible group-hover:visible"
		>
			<Button variant="ghost">
				<template #icon>
					<Play
						v-if="!playing"
						@click="playVideo"
						class="w-4 h-4 text-ink-gray-9"
					/>
					<Pause v-else @click="pauseVideo" class="w-4 h-4 text-ink-gray-9" />
				</template>
			</Button>
			<Button variant="ghost" @click="toggleMute">
				<template #icon>
					<Volume2 v-if="!muted" class="w-4 h-4 text-ink-gray-9" />
					<VolumeX v-else class="w-4 h-4 text-ink-gray-9" />
				</template>
			</Button>
			<input
				type="range"
				min="0"
				:max="duration"
				step="0.1"
				v-model="currentTime"
				@input="changeCurrentTime"
				class="duration-slider w-full h-1"
			/>
			<span class="text-xs font-medium">
				{{ formatTime(currentTime) }} / {{ formatTime(duration) }}
			</span>
			<Button variant="ghost" @click="toggleFullscreen">
				<template #icon>
					<Maximize class="w-4 h-4 text-ink-gray-9" />
				</template>
			</Button>
		</div>
	</div>
</template>

<script setup>
import { ref, onMounted, computed } from 'vue';
import { Play, Pause, Maximize, Volume2, VolumeX } from 'lucide-vue-next';
import { Button } from 'frappe-ui';

const videoRef = ref(null);
const videoContainer = ref(null);
let playing = ref(false);
let currentTime = ref(0);
let duration = ref(0);
let muted = ref(false);

const props = defineProps({
	file: {
		type: String,
		required: true,
	},
	type: {
		type: String,
		default: 'video/mp4',
	},
});

const isYoutube = computed(() => {
	return props.type === 'video/youtube';
});

const isRutube = computed(() => {
	return props.type === 'video/rutube' || props.file.includes('rutube.ru');
});

const fileURL = computed(() => {
	if (isYoutube.value) {
		let url = props.file;
		if (url.includes('watch?v=')) {
			url = url.replace('watch?v=', 'embed/');
		}
		return `${url}?autoplay=0&controls=0&disablekb=1&playsinline=1&cc_load_policy=1&cc_lang_pref=auto`;
	}
	if (isRutube.value) {
		let url = props.file;
		if (url.includes('rutube.ru')) {
			url = url.split('/').pop();
		}
		if (!url) {
			console.warn('Invalid Rutube ID:', props.file);
			return '';
		}
		return `https://rutube.ru/play/embed/${url}`;
	}
	return props.file;
});

onMounted(() => {
	setTimeout(() => {
		if (!isYoutube.value && !isRutube.value) {
			videoRef.value.onloadedmetadata = () => {
				duration.value = videoRef.value.duration;
			};
			videoRef.value.ontimeupdate = () => {
				currentTime.value = videoRef.value.currentTime;
			};
		}
	}, 0);
});

const playVideo = () => {
	if (!isYoutube.value && !isRutube.value) {
		videoRef.value.play();
		playing.value = true;
	}
};

const pauseVideo = () => {
	if (!isYoutube.value && !isRutube.value) {
		videoRef.value.pause();
		playing.value = false;
	}
};

const togglePlay = () => {
	if (playing.value) {
		pauseVideo();
	} else {
		playVideo();
	}
};

const videoEnded = () => {
	if (!isYoutube.value && !isRutube.value) {
		playing.value = false;
	}
};

const toggleMute = () => {
	if (!isYoutube.value && !isRutube.value) {
		videoRef.value.muted = !videoRef.value.muted;
		muted.value = videoRef.value.muted;
	}
};

const changeCurrentTime = () => {
	if (!isYoutube.value && !isRutube.value) {
		videoRef.value.currentTime = currentTime.value;
	}
};

const formatTime = (time) => {
	const minutes = Math.floor(time / 60);
	const seconds = Math.floor(time % 60);
	return `${minutes}:${seconds < 10 ? '0' : ''}${seconds}`;
};

const toggleFullscreen = () => {
	if (document.fullscreenElement) {
		document.exitFullscreen();
	} else {
		videoContainer.value.requestFullscreen();
	}
};
</script>

<style scoped>
.video-block {
	width: 100%;
	max-width: 900px;
	margin: 0 auto;
}

.video-block video {
	width: 100%;
	height: auto;
}

iframe {
	width: 100%;
	min-height: 500px;
	border-radius: 8px;
	border: 1px solid theme('colors.gray.100');
}

.duration-slider {
	flex: 1;
	-webkit-appearance: none;
	appearance: none;
	background-color: theme('colors.gray.400');
	cursor: pointer;
}

.duration-slider::-webkit-slider-thumb {
	height: 10px;
	width: 10px;
	-webkit-appearance: none;
	background-color: theme('colors.gray.900');
}

@media screen and (-webkit-min-device-pixel-ratio: 0) {
	input[type='range'] {
		overflow: hidden;
		width: 150px;
		-webkit-appearance: none;
	}

	input[type='range']::-webkit-slider-thumb {
		-webkit-appearance: none;
		cursor: pointer;
		box-shadow: -500px 0 0 500px theme('colors.gray.900');
	}
}
</style>