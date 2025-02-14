<template>
	<Modal @close="onClose">
		<div class="modal-content">
			<h2>{{ t('mail', 'Create task') }}</h2>
			<div class="taskTitle">
				<input v-model="taskTitle" type="text">
			</div>
			<div class="all-day">
				<DatetimePicker v-model="startDate"
					:format="dateFormat"
					:clearable="false"
					:minute-step="5"
					:show-second="false"
					:type="datePickerType"
					:show-timezone-select="true"
					:timezone-id="startTimezoneId" />
				<DatetimePicker v-model="endDate"
					:format="dateFormat"
					:clearable="false"
					:minute-step="5"
					:show-second="false"
					:type="datePickerType"
					:show-timezone-select="true"
					:timezone-id="endTimezoneId" />
			</div>
			<label for="note">Description:</label>
			<textarea id="note" v-model="note" rows="7" />
			<div class="all-day">
				<input
					id="allDay"
					v-model="isAllDay"
					type="checkbox"
					class="checkbox">
				<label
					for="allDay">
					{{ t('mail', 'All day') }}
				</label>
			</div>
			<Multiselect
				v-model="selectedCalendar"
				label="displayname"
				track-by="url"
				:placeholder="t('mail', 'Select calendar')"
				:allow-empty="false"
				:options="calendars">
				<template #option="{option}">
					<CalendarPickerOption
						v-bind="option" />
				</template>
				<template #singleLabel="{option}">
					<CalendarPickerOption
						:display-icon="option.displayIcon"
						v-bind="option" />
				</template>
				<span slot="noOptions">{{ t('mail', 'No calendars with task list support') }}</span>
			</Multiselect>
			<br>
			<button class="primary" :disabled="disabled" @click="onSave">
				{{ t('mail', 'Create') }}
			</button>
		</div>
	</Modal>
</template>

<script>
import { NcDatetimePicker as DatetimePicker, NcModal as Modal, NcMultiselect as Multiselect } from '@nextcloud/vue'
import jstz from 'jstz'

import logger from '../logger.js'
import ICAL from 'ical.js'
import Task from '../task.js'
import CalendarPickerOption from './CalendarPickerOption.vue'
import { showError, showSuccess } from '@nextcloud/dialogs'
import moment from '@nextcloud/moment'

export default {
	name: 'TaskModal',
	components: {
		CalendarPickerOption,
		DatetimePicker,
		Modal,
		Multiselect,
	},
	props: {
		envelope: {
			type: Object,
			required: true,
		},
	},
	data() {
		// Try to determine the current timezone, and fall back to UTC otherwise
		const defaultTimezone = jstz.determine()
		const defaultTimezoneId = defaultTimezone ? defaultTimezone.name() : 'UTC'

		return {

			taskTitle: this.envelope.subject,
			startDate: null,
			endDate: null,
			isAllDay: true,
			startTimezoneId: defaultTimezoneId,
			endTimezoneId: defaultTimezoneId,
			saving: false,
			selectedCalendar: undefined,
			note: this.envelope.previewText,
		}
	},
	computed: {
		disabled() {
			return this.saving || this.calendars.length === 0
		},
		dateFormat() {
			return this.isAllDay ? 'YYYY-MM-DD' : 'YYYY-MM-DD HH:mm'
		},
		datePickerType() {
			return this.isAllDay ? 'date' : 'datetime'
		},
		tags() {
			return this.$store.getters.getAllTags
		},
		calendars() {
			return this.$store.getters.getTaskCalendarsForCurrentUser
		},
	},
	created() {
		logger.debug('creating task from envelope', {
			envelope: this.envelope,
		})
	},
	async mounted() {

		if (this.calendars.length) {
			this.selectedCalendar = this.calendars[0]
		}
	},
	methods: {

		onClose() {
			this.$emit('close')
		},
		async createTask(taskData) {
			const task = new Task('BEGIN:VCALENDAR\nVERSION:2.0\nPRODID:-//Nextcloud Mail v' + this.$store.getters.getAppVersion + '\nEND:VCALENDAR', taskData.calendar)
			task.created = ICAL.Time.now()
			task.summary = taskData.summary
			task.hidesubtasks = 0
			if (taskData.priority) {
				task.priority = taskData.priority
			}
			if (taskData.complete) {
				task.complete = taskData.complete
			}
			if (taskData.note) {
				task.note = taskData.note
			}
			if (taskData.due) {
				task.due = taskData.due
			}
			if (taskData.start) {
				task.start = taskData.start
			}
			if (taskData.allDay) {
				task.allDay = taskData.allDay
			}
			const vData = ICAL.stringify(task.jCal)

			await task.calendar.dav.createVObject(vData)

			return task

		},
		async onSave() {
			this.saving = true

			const taskData = {
				summary: this.taskTitle,
				calendar: this.selectedCalendar,
				start: this.startDate ? moment(this.startDate).format().toString() : null,
				due: this.endDate ? moment(this.endDate).set().format().toString() : null,
				allDay: this.isAllDay,
				note: this.note,
			}
			try {
				logger.debug('create task', taskData)

				await this.createTask(taskData)

				showSuccess(t('mail', 'Task created'))

				this.onClose()
			} catch (error) {
				showError(t('mail', 'Could not create task'))

				logger.error('Creating event from message failed', { error })
			} finally {
				this.saving = false
			}
		},
	},
}
</script>

<style lang="scss" scoped>
:deep(.modal-wrapper .modal-container) {
	width: calc(100vw - 120px) !important;
	height: calc(100vh - 120px) !important;
	max-width: 490px !important;
	max-height: 500px !important;
}
:deep(.calendar-picker-option__color-indicator){
    margin-left: 10px !important;
}
.modal-content {
	padding: 30px 30px 20px !important;
}
input , textarea {
	width: 100%;
}
:deep(input[type='text'].multiselect__input) {
	padding: 0 !important;
}
:deep(.multiselect__single) {
	margin-left: -18px;
	width: 100px;
}
:deep(.multiselect__tags) {
	border: none !important;
}
.all-day {
	margin-left: -1px;
	margin-top: 5px;
	margin-bottom: 5px;
}
.taskTitle {
	margin-bottom: 5px;
}
.primary {
	height: 44px !important;
	float: right;
}
:deep(.mx-datepicker) {
	width: 213px;
}
</style>
