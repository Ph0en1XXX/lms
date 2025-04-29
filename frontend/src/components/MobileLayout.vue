<template>
	<div class="flex h-full flex-col">
		<div class="h-full pb-10" id="scrollContainer">
			<slot />
		</div>
		<div
			class="fixed flex items-center justify-around border-t border-outline-gray-2 bottom-0 z-10 w-full bg-surface-white standalone:pb-4"
			:style="{
				gridTemplateColumns: `repeat(${sidebarLinks.length + 1}, minmax(0, 1fr))`,
			}"
		>
			<button
				v-for="tab in sidebarLinks"
				:key="tab.label"
				:class="isVisible(tab) ? 'block' : 'hidden'"
				class="flex flex-col items-center justify-center py-3 transition active:scale-95"
				@click="handleClick(tab)"
			>
				<component
					:is="icons[tab.icon]"
					class="h-6 w-6 stroke-1.5"
					:class="[isActive(tab) ? 'text-ink-gray-9' : 'text-ink-gray-5']"
				/>
			</button>
			<Popover
				trigger="hover"
				popoverClass="bottom-28 mx-2"
				placement="top-start"
			>
				<template #target>
					<component
						:is="icons['List']"
						class="h-6 w-6 stroke-1.5 text-ink-gray-5"
					/>
				</template>
				<template #body-main>
					<div class="text-base p-5 space-y-4">
						<div
							v-for="link in otherLinks"
							:key="link.label"
							class="flex items-center space-x-2"
							@click="handleClick(link)"
						>
							<component
								:is="icons[link.icon]"
								class="h-4 w-4 stroke-1.5 text-ink-gray-5"
							/>
							<div>
								{{ link.label }}
							</div>
						</div>
					</div>
				</template>
			</Popover>
		</div>
	</div>
</template>

<script setup>
import { getSidebarLinks } from '../utils'
import { useRouter } from 'vue-router'
import { watch, ref, onMounted } from 'vue'
import { sessionStore } from '@/stores/session'
import { usersStore } from '@/stores/user'
import { Popover } from 'frappe-ui'
import * as icons from 'lucide-vue-next'

const { logout, user } = sessionStore()
let { isLoggedIn } = sessionStore()
const router = useRouter()
let { userResource } = usersStore()
const sidebarLinks = ref([])
const otherLinks = ref([])

onMounted(() => {
	addSideBar()
	addOtherLinks()
})

const addSideBar = () => {

	sidebarLinks.value.push({
		label: __('Courses'),
		icon: 'BookOpen',
		to: 'Courses',
		activeFor: [
			'Courses',
			'CourseDetail',
			'Lesson',
			'CourseForm',
			'LessonForm',
		],
	})

	sidebarLinks.value.push({
		label: __('Leader Board'),
		icon: 'Trophy',
		to: 'leaderboardsample',
		external: true,
		activeFor: [],
	})

	sidebarLinks.value.push({
		label: __('My points'),
		icon: 'Award',
		to: 'my_points',
		external: true,
		activeFor: [],
	})

	let chatGPTURL = ''
		let chatGPTLabel = ''

		if (roles.includes('LMS Schoolchild')) {
			chatGPTURL = 'chatgpt-schoolchild'
			chatGPTLabel = __('ChatGPT for Schoolers')
		} else if (roles.includes('LMS Student')) {
			chatGPTURL = 'chatgpt-schoolchild'
			chatGPTLabel = __('ChatGPT for Students')
		} else if (roles.includes('Course Creator')) {
			chatGPTURL = 'AI-teachers'
			chatGPTLabel = __('ChatGPT for Teachers')
		}

	if (chatGPTURL) {
		sidebarLinks.value.push({
			label: chatGPTLabel,
			icon: 'Cpu',
			to: chatGPTURL,
			external: true,
			activeFor: [],
		})
	}

}

const addOtherLinks = () => {
	otherLinks.value = []

	if (user) {
		const roles = userResource.data?.roles || []

		if (!userResource.data?.is_instructor && !userResource.data?.is_moderator) {
			otherLinks.value.push({
				label: __('Programs'),
				icon: 'Route',
				to: 'Programs',
				activeFor: ['Programs', 'ProgramForm', 'CourseDetail', 'Lesson'],
			})
		} else if (userResource.data?.is_instructor || userResource.data?.is_moderator) {
			otherLinks.value.push({
				label: __('Programs'),
				icon: 'Route',
				to: 'Programs',
				activeFor: ['Programs', 'ProgramForm'],
			})
		}

		if (userResource.data?.is_moderator || userResource.data?.is_instructor) {
			otherLinks.value.push({
				label: __('Quizzes'),
				icon: 'CircleHelp',
				to: 'Quizzes',
				activeFor: [
					'Quizzes',
					'QuizForm',
					'QuizSubmissionList',
					'QuizSubmission',
				],
			})

			otherLinks.value.push({
				label: __('Assignments'),
				icon: 'Pencil',
				to: 'Assignments',
				activeFor: [
					'Assignments',
					'AssignmentForm',
					'AssignmentSubmissionList',
					'AssignmentSubmission',
				],
			})
		}

		if (roles.includes('LMS Student') || roles.includes('LMS Schoolchild')) {
			otherLinks.value.push({
				label: __('My points'),
				icon: 'Award',
				to: 'my_points',
				external: true,
				activeFor: [],
			})
		}

		let chatGPTURL = ''
		let chatGPTLabel = ''

		if (roles.includes('LMS Schoolchild')) {
			chatGPTURL = 'chatgpt-schoolchild'
			chatGPTLabel = __('ChatGPT for Schoolers')
		} else if (roles.includes('LMS Student')) {
			chatGPTURL = 'chatgpt-schoolchild'
			chatGPTLabel = __('ChatGPT for Students')
		} else if (roles.includes('Course Creator')) {
			chatGPTURL = 'AI-teachers'
			chatGPTLabel = __('ChatGPT for Teachers')
		}

		if (chatGPTURL) {
			otherLinks.value.push({
				label: chatGPTLabel,
				icon: 'Cpu',
				to: chatGPTURL,
				external: true,
				activeFor: [],
			})
		}

		otherLinks.value.push({
			label: __('Leader Board'),
			icon: 'Trophy',
			to: 'leaderboardsample',
			external: true,
			activeFor: [],
		})

		otherLinks.value.push({
			label: __('Profile'),
			icon: 'UserRound',
			to: 'Profile',
			params: { username: userResource.data?.username },
		})

		otherLinks.value.push({
			label: __('Log out'),
			icon: 'LogOut',
		})
	} else {
		otherLinks.value.push({
			label: __('Log in'),
			icon: 'LogIn',
		})
	}
}

watch(userResource, () => {
	if (userResource.data) {
		addOtherLinks()
	}
})

let isActive = (tab) => {
	return tab.activeFor?.includes(router.currentRoute.value.name)
}

const handleClick = (tab) => {
	if (tab.label === 'Log in') {
		window.location.href = '/login'
	} else if (tab.label === 'Log out') {
		logout.submit().then(() => {
			isLoggedIn = false
		})
	} else if (tab.label === 'Profile') {
		router.push({
			name: 'Profile',
			params: {
				username: userResource.data?.username,
			},
		})
	} else if (tab.external) {
		window.location.href = `/${tab.to}`
	} else {
		router.push({ name: tab.to, params: tab.params })
	}
}

const isVisible = (tab) => {
	if (tab.label === 'Log in') return !isLoggedIn
	else if (tab.label === 'Log out') return isLoggedIn
	else return true
}
</script>
