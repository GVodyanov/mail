<!--
  - @copyright Copyright (c) 2022 Richard Steinmetz <richard@steinmetz.cloud>
  -
  - @author Richard Steinmetz <richard@steinmetz.cloud>
  -
  - @license AGPL-3.0-or-later
  -
  - This program is free software: you can redistribute it and/or modify
  - it under the terms of the GNU Affero General Public License as
  - published by the Free Software Foundation, either version 3 of the
  - License, or (at your option) any later version.
  -
  - This program is distributed in the hope that it will be useful,
  - but WITHOUT ANY WARRANTY; without even the implied warranty of
  - MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the
  - GNU Affero General Public License for more details.
  -
  - You should have received a copy of the GNU Affero General Public License
  - along with this program. If not, see <http://www.gnu.org/licenses/>.
  -
  -->

<template>
	<form class="form" @submit.prevent="submit">
		<div class="form__multi-row">
			<fieldset class="form__fieldset">
				<input
					id="ooo-disabled"
					class="radio"
					type="radio"
					name="enabled"
					:checked="enabled === OOO_DISABLED"
					@change="enabled = OOO_DISABLED">
				<label for="ooo-disabled">{{ t('mail', 'Autoresponder off') }}</label>
			</fieldset>

			<fieldset class="form__fieldset">
				<input
					id="ooo-enabled"
					class="radio"
					type="radio"
					name="enabled"
					:checked="enabled === OOO_ENABLED"
					@change="enabled = OOO_ENABLED">
				<label for="ooo-enabled">{{ t('mail', 'Autoresponder on') }}</label>
			</fieldset>

			<fieldset v-if="hasPersonalAbsenceSettings" class="form__fieldset">
				<input
					id="ooo-follow-system"
					class="radio"
					type="radio"
					name="enabled"
					:checked="enabled === OOO_FOLLOW_SYSTEM"
					@change="enabled = OOO_FOLLOW_SYSTEM">
				<label for="ooo-follow-system">{{ t('mail', 'Autoresponder follows system settings') }}</label>
			</fieldset>
		</div>

		<template v-if="followingSystem">
			<p>{{ t('mail', 'The autoresponder follows your personal absence period settings.') }}</p>
			<Button :href="personalAbsenceSettingsUrl" target="_blank" rel="noopener noreferrer">
				<template #icon>
					<OpenInNewIcon :size="20" />
				</template>
				{{ t('mail', 'Edit absence settings') }}
			</Button>
		</template>
		<template v-else>
			<div class="form__multi-row">
				<fieldset class="form__fieldset">
					<label for="ooo-first-day">{{ t('mail', 'First day') }}</label>
					<DatetimePicker
						id="ooo-first-day"
						v-model="firstDay"
						:disabled="!enabled" />
				</fieldset>

				<fieldset class="form__fieldset">
					<div class="form__fieldset__label">
						<input
							id="ooo-enable-last-day"
							v-model="enableLastDay"
							type="checkbox"
							:disabled="!enabled">
						<label for="ooo-enable-last-day">
							{{ t('mail', 'Last day (optional)') }}
						</label>
					</div>
					<DatetimePicker
						id="ooo-last-day"
						v-model="lastDay"
						:disabled="!enabled || !enableLastDay" />
				</fieldset>
			</div>

			<fieldset class="form__fieldset">
				<label for="ooo-subject">{{ t('mail', 'Subject') }}</label>
				<input
					id="ooo-subject"
					v-model="subject"
					type="text"
					:disabled="followingSystem">
				<p class="form__fieldset__description">
					{{ t('mail', '${subject} will be replaced with the subject of the message you are responding to') }}
				</p>
			</fieldset>

			<fieldset class="form__fieldset">
				<label for="ooo-message">{{ t('mail', 'Message') }}</label>
				<TextEditor
					id="ooo-message"
					v-model="message"
					:html="false"
					:disabled="followingSystem"
					:bus="{ '$on': () => { /* noop */ } }" />
			</fieldset>
		</template>

		<p v-if="errorMessage">
			{{ t('mail', 'Oh Snap!') }}
			{{ errorMessage }}
		</p>

		<Button
			type="primary"
			native-type="submit"
			:aria-label="t('mail', 'Save autoresponder')"
			:disabled="loading || !valid">
			<template #icon>
				<CheckIcon :size="20" />
			</template>
			{{ t('mail', 'Save autoresponder') }}
		</Button>
	</form>
</template>

<script>
import { NcDatetimePicker as DatetimePicker, NcButton as Button } from '@nextcloud/vue'
import TextEditor from './TextEditor.vue'
import CheckIcon from 'vue-material-design-icons/Check'
import { html, plain, toHtml, toPlain } from '../util/text.js'
import { loadState } from '@nextcloud/initial-state'
import { generateUrl } from '@nextcloud/router'
import OpenInNewIcon from 'vue-material-design-icons/OpenInNew.vue'
import * as OutOfOfficeService from '../service/OutOfOfficeService.js'

const OOO_DISABLED = 'disabled'
const OOO_ENABLED = 'enabled'
const OOO_FOLLOW_SYSTEM = 'system'

export default {
	name: 'OutOfOfficeForm',
	components: {
		DatetimePicker,
		TextEditor,
		Button,
		CheckIcon,
		OpenInNewIcon,
	},
	props: {
		account: {
			type: Object,
			required: true,
		},
	},
	data() {
		const nextcloudVersion = parseInt(OC.config.version.split('.')[0])
		const enableSystemOutOfOffice = loadState('mail', 'enable-system-out-of-office', false)

		return {
			OOO_DISABLED,
			OOO_ENABLED,
			OOO_FOLLOW_SYSTEM,
			enabled: this.account.outOfOfficeFollowsSystem ? OOO_FOLLOW_SYSTEM : OOO_DISABLED,
			enableLastDay: false,
			firstDay: new Date(),
			lastDay: null,
			subject: '',
			message: '',
			loading: false,
			errorMessage: '',
			hasPersonalAbsenceSettings: nextcloudVersion >= 28 && enableSystemOutOfOffice,
			personalAbsenceSettingsUrl: generateUrl('/settings/user/availability'),
		}
	},
	computed: {
		/**
		 * @return {boolean}
		 */
		valid() {
			if (this.followingSystem) {
				return true
			}

			if (this.enabled === OOO_DISABLED) {
				return true
			}

			return !!(this.firstDay
				&& (!this.enableLastDay || (this.enableLastDay && this.lastDay))
				&& (!this.enableLastDay || (this.lastDay >= this.firstDay))
				&& this.subject
				&& this.message)
		},

		/**
		 * Main address and all aliases formatted for use with sieve.
		 *
		 * @return {string[]}
		 */
		aliases() {
			 return [
				 {
					 name: this.account.name,
					 alias: this.account.emailAddress,
				 },
				 ...this.account.aliases,
			 ].map(({ name, alias }) => `${name} <${alias}>`)
		},

		/**
		 * @return {boolean}
		 */
		followingSystem() {
			return this.hasPersonalAbsenceSettings && this.enabled === OOO_FOLLOW_SYSTEM
		},
	},
	watch: {
		enableLastDay(enableLastDay) {
			if (enableLastDay) {
				this.lastDay = new Date(this.firstDay)
				this.lastDay.setDate(this.lastDay.getDate() + 6)
			} else {
				this.lastDay = null
			}
		},
		firstDay(firstDay, previousFirstDay) {
			if (!this.enableLastDay) {
				return
			}

			const dayInMillis = 24 * 60 * 60 * 1000
			const diffDays = Math.floor((this.lastDay - previousFirstDay) / dayInMillis)
			if (diffDays < 0) {
				return
			}

			this.lastDay = new Date(firstDay)
			this.lastDay.setDate(firstDay.getDate() + diffDays)
		},
	},
	async mounted() {
		await this.fetchState()
	},
	methods: {
		async fetchState() {
			const { state } = await OutOfOfficeService.fetch(this.account.id)

			if (this.account.outOfOfficeFollowsSystem) {
				this.enabled = OOO_FOLLOW_SYSTEM
			} else {
				this.enabled = state.enabled ? OOO_ENABLED : OOO_DISABLED
			}
			if (state.enabled) {
				this.firstDay = state.start ? new Date(state.start) : new Date()
				this.lastDay = state.end ? new Date(state.end) : null
			}
			this.enableLastDay = !!this.lastDay
			this.subject = state.subject
			this.message = toHtml(plain(state.message)).value
		},
		async submit() {
			this.loading = true
			this.errorMessage = ''

			try {
				if (this.followingSystem) {
					await OutOfOfficeService.followSystem(this.account.id)
					this.$store.commit('patchAccount', {
						account: this.account,
						data: {
							outOfOfficeFollowsSystem: true,
						},
					})
				} else {
					await OutOfOfficeService.update(this.account.id, {
						enabled: this.enabled === OOO_ENABLED,
						start: this.firstDay,
						end: this.lastDay,
						subject: this.subject,
						message: toPlain(html(this.message)).value, // CKEditor always returns html data
						allowedRecipients: this.aliases,
					})
					this.$store.commit('patchAccount', {
						account: this.account,
						data: {
							outOfOfficeFollowsSystem: false,
						},
					})
				}
				await this.$store.dispatch('fetchActiveSieveScript', { accountId: this.account.id })
			} catch (error) {
				this.errorMessage = error.message
			} finally {
				this.loading = false
			}
		},
	},
}
</script>

<style lang="scss" scoped>
.form {
	display: flex;
	flex-direction: column;
	gap: 15px;

	&__fieldset {
		display: flex;
		flex-direction: column;

		&__label {
			display: flex;
			align-items: center;
			gap: 5px;
		}

		&__input {
			flex: 1 auto;
		}

		&__description {
			color: var(--color-text-maxcontrast);
		}
	}

	&__multi-row {
		display: flex;
		align-items: end;
		gap: 15px;
	}

	#ooo-enable-last-day {
		cursor: pointer;
		min-height: unset;
	}

	#ooo-subject {
		width: 100%;
	}

	#ooo-message {
		width: 100%;
		min-height: 100px;
		border: 1px solid var(--color-border);

		&:active,
		&:focus,
		&:hover {
			border-color: var(--color-primary-element) !important;
		}
	}
}
</style>
