<template>
	<div class="mt-7 mb-10">
		<h2 class="mb-3 text-lg font-semibold text-ink-gray-9">
			{{ __('About') }}
		</h2>
		<div
			v-if="filteredBio"
			v-html="filteredBio"
			class="ProseMirror prose prose-table:table-fixed prose-td:p-2 prose-th:p-2 prose-td:border prose-th:border prose-td:border-outline-gray-2 prose-th:border-outline-gray-2 prose-td:relative prose-th:relative prose-th:bg-surface-gray-2 prose-sm max-w-none !whitespace-normal"
		></div>

		<!-- Email
		<div v-if="profile.data.email" class="mt-3 text-gray-700 text-sm">
			<strong>{{ __('Email') }}:</strong> {{ profile.data.email }}
		</div>

		Phone
		<div v-if="profile.data.phone" class="mt-1 text-gray-700 text-sm">
			<strong>{{ __('Phone') }}:</strong> {{ profile.data.phone }}
		</div>-->

		<div v-else-if="!profile.data.email && !profile.data.phone && !profile.data.bio" class="text-ink-gray-7 text-sm italic">
			{{ __('No introduction') }}
		</div>
	</div>

	<!-- Points Section -->
	<div class="mt-7 mb-10" v-if="energyPoints.data?.length">
		<h2 class="mb-3 text-lg font-semibold text-ink-gray-9">
			{{ __('Points') }}
		</h2>
		<h2 class="mb-3 text-lg font-semibold text-ink-gray-9">
			{{ __('The last 10 score records have been uploaded') }}
		</h2>
		<ul class="space-y-4">
			<li v-for="item in energyPoints.data.slice(0, 10)" :key="item.name" class="text-sm text-gray-700">
				<div class="flex justify-between">
					<span>{{ __('Points') }}: {{ item.points }}</span>
					<span>{{ dayjs(item.creation).format('DD-MM-YYYY') }}</span>
				</div>
				<div v-if="item.rule" class="text-ink-gray-7">
					{{ __('Reason') }}: {{ item.rule }}
				</div>
			</li>
		</ul>
		<div class="mt-4 text-sm">
			<div class="font-semibold">
				{{ __('Total Points') }}: {{ totalPoints }}
			</div>
			<div class="font-semibold">
				{{ __('Additional Points for MPGU Admission') }}: {{ additionalPoints }}
			</div>
		</div>
	</div>
	<div v-else class="mt-7 text-ink-gray-7 text-sm italic">
		{{ __('No points yet') }}
	</div>

	<!-- Courses Section -->
	<div class="mt-7 mb-10" v-if="coursesWithTitles.length">
		<h2 class="mb-3 text-lg font-semibold text-ink-gray-9">
			{{ __('Completed Courses') }}
		</h2>
		<div class="grid grid-cols-1 gap-4">
			<div v-for="course in coursesWithTitles" :key="course.name">
				<a
					:href="`/lms/courses/${course.course}`"
					class="text-base text-ink-gray-9 hover:text-blue-600"
				>
					{{ course.title || course.course }} <!-- Отображаем title, если есть, иначе course -->
				</a>
				<div class="text-sm text-ink-gray-7">
					{{ __('Completed on') }}: {{ dayjs(course.creation).format('DD MMM YYYY') }}
				</div>
			</div>
		</div>
	</div>
	<div v-else class="mt-7 text-ink-gray-7 text-sm italic">
		{{ __('No completed courses yet') }}
	</div>

	<!-- Achievements Section -->
	<div class="mt-7 mb-10" v-if="badges.data?.length">
		<h2 class="mb-3 text-lg font-semibold text-ink-gray-9">
			{{ __('Achievements') }}
		</h2>
		<div class="grid grid-cols-2 md:grid-cols-3 lg:grid-cols-5 gap-4">
			<div v-for="badge in badges.data">
				<Popover trigger="hover" :leaveDelay="Number(0.01)">
					<template #target>
						<div class="relative">
							<img
								:src="badge.badge_image"
								:alt="badge.badge"
								class="h-[80px]"
							/>
							<div
								v-if="badge.count > 1"
								class="flex items-end bg-surface-gray-2 p-2 text-xs font-semibold rounded-full absolute right-0 bottom-0"
							>
								<span>
									<X class="w-3 h-3" />
								</span>
								{{ badge.count }}
							</div>
						</div>
					</template>
					<template #body-main>
						<div class="w-[250px] text-base">
							<img
								:src="badge.badge_image"
								:alt="badge.badge"
								class="bg-surface-gray-2 rounded-t-md h-[200px] mx-auto"
							/>
							<div class="p-5">
								<div class="text-2xl font-semibold mb-2">
									{{ badge.badge }}
								</div>
								<div class="leading-5 mb-4">
									{{ badge.badge_description }}
								</div>
								<div class="flex flex-col mb-4">
									<span class="text-xs text-ink-gray-7 font-medium mb-1">
										{{ __('Issued on') }}:
									</span>
									{{ dayjs(badge.issued_on).format('DD MMM YYYY') }}
								</div>
								<div class="flex flex-col">
									<span class="text-xs text-ink-gray-7 font-medium mb-1">
										{{ __('Share on') }}:
									</span>
									<div class="flex items-center space-x-2">
										<Button
											variant="outline"
											size="sm"
											@click="shareOnSocial(badge, 'LinkedIn')"
										>
											<template #prefix>
												<LinkedinIcon class="h-3 w-3 text-ink-gray-7" />
											</template>
											<span class="text-xs">
												{{ __('LinkedIn') }}
											</span>
										</Button>
										<Button
											variant="outline"
											size="sm"
											@click="shareOnSocial(badge, 'Twitter')"
										>
											<template #prefix>
												<Twitter class="h-3 w-3 text-ink-gray-7" />
											</template>
											<span class="text-xs">
												{{ __('Twitter') }}
											</span>
										</Button>
									</div>
								</div>
							</div>
						</div>
					</template>
				</Popover>
			</div>
		</div>
	</div>
</template>

<script setup>
import { inject, computed, ref, watch } from 'vue'
import { createResource, Popover, Button } from 'frappe-ui'
import { X, LinkedinIcon, Twitter } from 'lucide-vue-next'
import { sessionStore } from '@/stores/session'

const dayjs = inject('$dayjs')
const { branding } = sessionStore()

const props = defineProps({
	profile: {
		type: Object,
		required: true,
	},
})

const badges = createResource({
	url: 'frappe.client.get_list',
	params: {
		doctype: 'LMS Badge Assignment',
		fields: ['name', 'badge', 'badge_image', 'badge_description', 'issued_on'],
		filters: {
			member: props.profile.data.name,
		},
	},
	auto: true,
	transform(data) {
		let finalBadges = []
		let groupedBadges = Object.groupBy(data, ({ badge }) => badge)
		for (let badge in groupedBadges) {
			let badgeData = groupedBadges[badge][0]
			badgeData.count = groupedBadges[badge].length
			finalBadges.push(badgeData)
		}
		return finalBadges
	},
})

const courses = createResource({
	url: 'frappe.client.get_list',
	params: {
		doctype: 'LMS Course Progress',
		fields: ['name', 'member', 'course', 'status', 'creation'],
		filters: {
			member: props.profile.data.email,
			status: 'Complete',
		},
	},
	auto: true,
	onSuccess(data) {
		console.log('LMS Course Progress data:', data) // Отладка
	},
})

const courseTitles = ref({})
const coursesWithTitles = computed(() => {
	if (!courses.data) return []
	const result = courses.data.map(course => ({
		...course,
		title: courseTitles.value[course.course] || null,
	}))
	console.log('Courses with titles:', result) // Отладка
	return result
})

// Запрашиваем title для каждого курса
watch(
	() => courses.data,
	(newCourses) => {
		if (newCourses && newCourses.length) {
			const courseIds = newCourses.map(course => course.course)
			console.log('Course IDs:', courseIds) // Отладка
			createResource({
				url: 'frappe.client.get_list',
				params: {
					doctype: 'LMS Course',
					fields: ['name', 'title'],
					filters: {
						name: ['in', courseIds],
					},
				},
				auto: true,
				onSuccess(data) {
					console.log('Raw course titles data:', data) // Отладка сырых данных
					const titles = {}
					data.forEach(course => {
						titles[course.name] = course.title
					})
					courseTitles.value = titles
					console.log('Processed course titles:', titles) // Отладка обработанных titles
				},
				onError(error) {
					console.error('Error fetching course titles:', error) // Отладка ошибок
				},
			})
		}
	},
	{ immediate: true }
)

const energyPoints = createResource({
    url: 'frappe.client.get_list',
    params: {
        doctype: 'Energy Point Log',
        fields: ['name', 'user', 'points', 'rule', 'creation'],
        filters: {
            user: props.profile.data.email,
        },
		limit_page_length: 1000,
    },
    auto: true,
    onSuccess(data) {
        console.log('Energy Points data:', data) // Отладка
        data.forEach(item => {
            console.log('Points for', item.name, ':', item.points, typeof item.points)
        })
    },
})

const totalPoints = computed(() => {
	return energyPoints.data?.reduce((sum, item) => sum + (item.points || 0), 0) || 0
})

const additionalPoints = computed(() => {
	const points = Math.floor(totalPoints.value / 100)
	return points < 10 ? points : 10
})

const shareOnSocial = (badge, medium) => {
	let shareUrl
	const url = encodeURIComponent(
		`${window.location.origin}/lms/badges/${badge.badge}/${props.profile.data?.email}`
	)
	const summary = `I am happy to announce that I earned the ${
		badge.badge
	} badge on ${dayjs(badge.issued_on).format('DD MMM YYYY')} at ${
		branding.data?.app_name
	}.`

	if (medium == 'LinkedIn')
		shareUrl = `https://www.linkedin.com/shareArticle?mini=true&url=${url}&text=${summary}`
	else if (medium == 'Twitter')
		shareUrl = `https://twitter.com/intent/tweet?text=${summary}&url=${url}`

	window.open(shareUrl, '_blank')
}

const filteredBio = computed(() => {
    const bio = props.profile.data.bio;
    if (!bio) return null;

    const cleanBio = bio.trim();
    if (cleanBio === '<p></p>' || cleanBio === '<p><br></p>' || cleanBio === '<p>&nbsp;</p>') {
        return null;
    }

    return bio;
})

</script>
