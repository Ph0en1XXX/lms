<template>
	<div v-if="youtube">
		<iframe
			class="youtube-video"
			:src="getYouTubeVideoSource(youtube.split('/').pop())"
			width="100%"
			:height="screenSize.width < 640 ? 200 : 400"
			frameborder="0"
			allowfullscreen
		></iframe>
	</div>
	<div v-if="rutube">
		<iframe
			class="rutube-video"
			:src="getRutubeVideoSource(rutube.split('/').pop())"
			width="100%"
			:height="screenSize.width < 640 ? 200 : 400"
			frameborder="0"
			allowfullscreen
		></iframe>
	</div>
	<div v-for="block in content?.split('\n\n')">
		<div v-if="block.includes('{{ YouTubeVideo')">
			<iframe
				class="youtube-video"
				:src="getYouTubeVideoSource(block)"
				width="100%"
				:height="screenSize.width < 640 ? 200 : 400"
				frameborder="0"
				allowfullscreen
			></iframe>
		</div>
		<div v-else-if="block.includes('{{ RutubeVideo')">
			<iframe
				class="rutube-video"
				:src="getRutubeVideoSource(block)"
				width="100%"
				:height="screenSize.width < 640 ? 200 : 400"
				frameborder="0"
				allowfullscreen
			></iframe>
		</div>
		<div v-else-if="block.includes('{{ Quiz')">
			<Quiz :quiz="getId(block)" />
		</div>
		<div v-else-if="block.includes('{{ Video')">
			<video
				controls
				width="100%"
				controlsList="nodownload"
				oncontextmenu="return false;"
			>
				<source :src="getId(block)" type="video/mp4" />
			</video>
		</div>
		<div v-else-if="block.includes('{{ PDF')">
			<iframe
				:src="getPDFSource(block)"
				width="100%"
				height="700px"
				frameborder="0"
				allowfullscreen
			></iframe>
		</div>
		<div v-else-if="block.includes('{{ Audio')">
			<audio width="100%" controls controlsList="nodownload">
				<source :src="getId(block)" type="audio/mp3" />
			</audio>
		</div>
		<div v-else-if="block.includes('{{ Embed')">
			<iframe
				width="100%"
				height="400"
				:src="getId(block)"
				frameborder="0"
				allowfullscreen
			></iframe>
		</div>
		<div v-else v-html="markdown.render(block)"></div>
	</div>
	<div v-if="quizId">
		<Quiz :quiz="quizId" />
	</div>
</template>

<script setup>
import Quiz from '@/components/QuizBlock.vue'
import MarkdownIt from 'markdown-it'
import { useScreenSize } from '@/utils/composables'

const screenSize = useScreenSize()

const markdown = new MarkdownIt({
	html: true,
	linkify: true,
})

const props = defineProps({
	content: {
		type: String,
		required: true,
	},
	youtube: {
		type: String,
		required: false,
	},
	rutube: {
		type: String,
		required: false,
	},
	quizId: {
		type: String,
		required: false,
	},
})

const getYouTubeVideoSource = (block) => {
	if (block.includes('{{')) {
		block = getId(block)
	}
	return `https://www.youtube.com/embed/${block}`
}

const getRutubeVideoSource = (block) => {
	if (block.includes('{{')) {
		block = getId(block)
	}
	return `https://rutube.ru/play/embed/${block}`
}

const getPDFSource = (block) => {
	return `${getId(block)}#toolbar=0`
}

const getId = (block) => {
	const match = block.match(/\(["']([^"']+?)["']\)/)
	return match ? match[1] : ''
}
</script>

<style scoped>
.youtube-video,
.rutube-video {
	display: block;
	margin: 0 auto;
}
</style>