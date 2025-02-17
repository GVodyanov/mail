<!--
  - @copyright 2019 Christoph Wurst <christoph@winzerhof-wurst.at>
  -
  - @author 2019 Christoph Wurst <christoph@winzerhof-wurst.at>
  - @author 2022 Richard Steinmetz <richard@steinmetz.cloud>
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
  - MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
  - GNU Affero General Public License for more details.
  -
  - You should have received a copy of the GNU Affero General Public License
  - along with this program.  If not, see <http://www.gnu.org/licenses/>.
  -->

<template>
	<ckeditor
		v-if="ready"
		:value="value"
		:config="config"
		:editor="editor"
		:disabled="disabled"
		@input="onEditorInput"
		@ready="onEditorReady" />
</template>

<script>
import CKEditor from '@ckeditor/ckeditor5-vue2'
import AlignmentPlugin from '@ckeditor/ckeditor5-alignment/src/alignment'
import { Mention } from '@ckeditor/ckeditor5-mention'
import Editor from '@ckeditor/ckeditor5-editor-balloon/src/ballooneditor'
import EssentialsPlugin from '@ckeditor/ckeditor5-essentials/src/essentials'
import BlockQuotePlugin from '@ckeditor/ckeditor5-block-quote/src/blockquote'
import BoldPlugin from '@ckeditor/ckeditor5-basic-styles/src/bold'
import FontPlugin from '@ckeditor/ckeditor5-font/src/font'
import ParagraphPlugin from '@ckeditor/ckeditor5-paragraph/src/paragraph'
import HeadingPlugin from '@ckeditor/ckeditor5-heading/src/heading'
import ItalicPlugin from '@ckeditor/ckeditor5-basic-styles/src/italic'
import LinkPlugin from '@ckeditor/ckeditor5-link/src/link'
import ListStyle from '@ckeditor/ckeditor5-list/src/liststyle'
import RemoveFormat from '@ckeditor/ckeditor5-remove-format/src/removeformat'
import SignaturePlugin from '../ckeditor/signature/SignaturePlugin.js'
import StrikethroughPlugin from '@ckeditor/ckeditor5-basic-styles/src/strikethrough'
import QuotePlugin from '../ckeditor/quote/QuotePlugin.js'
import Base64UploadAdapter from '@ckeditor/ckeditor5-upload/src/adapters/base64uploadadapter'
import ImagePlugin from '@ckeditor/ckeditor5-image/src/image'
import ImageResizePlugin from '@ckeditor/ckeditor5-image/src/imageresize'
import ImageUploadPlugin from '@ckeditor/ckeditor5-image/src/imageupload'
import MailPlugin from '../ckeditor/mail/MailPlugin.js'
import { searchProvider, getLinkWithPicker } from '@nextcloud/vue/dist/Components/NcRichText'
import { emojiSearch, emojiAddRecent } from '@nextcloud/vue/dist/Functions/emoji'
import { getLanguage } from '@nextcloud/l10n'
import logger from '../logger.js'
import PickerPlugin from '../ckeditor/smartpicker/PickerPlugin.js'

export default {
	name: 'TextEditor',
	components: {
		ckeditor: CKEditor.component,
	},
	props: {
		value: {
			type: String,
			required: true,
		},
		html: {
			type: Boolean,
			default: false,
		},
		placeholder: {
			type: String,
			default: '',
		},
		focus: {
			type: Boolean,
			default: false,
		},
		bus: {
			type: Object,
			required: true,
		},
		disabled: {
			type: Boolean,
			default: false,
		},
	},
	data() {
		const plugins = [
			EssentialsPlugin,
			ParagraphPlugin,
			SignaturePlugin,
			QuotePlugin,
			PickerPlugin,
			Mention,
		]
		const toolbar = ['undo', 'redo']

		if (this.html) {
			plugins.push(...[
				HeadingPlugin,
				AlignmentPlugin,
				BoldPlugin,
				ItalicPlugin,
				BlockQuotePlugin,
				LinkPlugin,
				ListStyle,
				FontPlugin,
				RemoveFormat,
				StrikethroughPlugin,
				ImagePlugin,
				ImageUploadPlugin,
				Base64UploadAdapter,
				ImageResizePlugin,
				MailPlugin,
			])
			toolbar.unshift(...[
				'heading',
				'fontFamily',
				'fontSize',
				'bold',
				'italic',
				'fontColor',
				'imageUpload',
				'alignment',
				'bulletedList',
				'numberedList',
				'blockquote',
				'fontBackgroundColor',
				'strikethrough',
				'link',
				'removeFormat',
			])
		}

		return {
			linkTribute: null,
			emojiTribute: null,
			textSmiles: [],
			ready: false,
			editor: Editor,
			config: {
				placeholder: this.placeholder,
				plugins,
				toolbar: {
					items: toolbar,
				},
				language: 'en',
				mention: {
					feeds: [
						{
							marker: ':',
							feed: this.getEmoji,
							itemRenderer: this.customEmojiRenderer,
						},
						{
							marker: '/',
							feed: this.getLink,
							itemRenderer: this.customLinkRenderer,
						},
					],
				},
			},
		}
	},
	beforeMount() {
		this.loadEditorTranslations(getLanguage())
	},
	methods: {
		getLink(text) {
			const results = searchProvider(text)
			if (results.length === 1 && !results[0].title.toLowerCase().includes(text.toLowerCase())) {
				return []
			}
			return results
		},
		getEmoji(text) {
			const emojiResults = emojiSearch(text)
			if (this.textSmiles.includes(':' + text)) {

				emojiResults.unshift(':' + text)
			}
			return emojiResults
		},
		 customEmojiRenderer(item) {
			const itemElement = document.createElement('span')

			itemElement.classList.add('custom-item')
			itemElement.id = `mention-list-item-id-${item.colons}`
			itemElement.textContent = `${item.native} `
			const usernameElement = document.createElement('span')

			usernameElement.classList.add('custom-item-username')
			usernameElement.textContent = item.colons

			itemElement.appendChild(usernameElement)

			return itemElement
		},
		customLinkRenderer(item) {
			const itemElement = document.createElement('span')
			itemElement.classList.add('link-container')

			const icon = document.createElement('img')
			icon.classList.add('link-icon')
			icon.src = `${item.icon_url} `

			const usernameElement = document.createElement('span')

			usernameElement.classList.add('link-title')
			usernameElement.textContent = `${item.title} `
			itemElement.appendChild(icon)
			itemElement.appendChild(usernameElement)

			return itemElement
		},
		async loadEditorTranslations(language) {
			if (language === 'en') {
				// The default, nothing to fetch
				return this.showEditor('en')
			}

			try {
				logger.debug(`loading ${language} translations for CKEditor`)
				await import(
					/* webpackMode: "lazy-once" */
					/* webpackPrefetch: true */
					/* webpackPreload: true */
					`@ckeditor/ckeditor5-build-balloon/build/translations/${language}`
				)
				this.showEditor(language)
			} catch (error) {
				logger.error(`could not find CKEditor translations for "${language}"`, { error })
				this.showEditor('en')
			}
		},
		showEditor(language) {
			logger.debug(`using "${language}" as CKEditor language`)
			this.config.language = language

			this.ready = true
		},
		/**
		 * @param {module:core/editor/editor~Editor} editor editor the editor instance
		 */
		onEditorReady(editor) {
			logger.debug('TextEditor is ready', { editor })

			editor.commands.get('mention')?.on('execute', (event, data) => {
				event.stop()
				const eventData = data[0]
				const item = eventData.mention
				if (eventData.marker === ':') {
					emojiAddRecent(item)
					this.editorInstance.execute('insertItem', item.native, ':')
				}
				if (eventData.marker === '/') {
					getLinkWithPicker(item.id)
						.then((link) => {
							this.editorInstance.execute('insertItem', link, '/')
							this.editorInstance.editing.view.focus()
						})
						.catch((error) => {
							console.debug('Smart picker promise rejected:', error)
						})
				}
			}, { priority: 'high' })
			this.editorInstance = editor

			if (this.focus) {
				logger.debug('focusing TextEditor')
				editor.editing.view.focus()
			}

			if (this.html) {
				this.$emit('show-toolbar', editor.ui._focusableToolbarDefinitions[0].toolbarView.element)
			}
			this.bus.$on('append-to-body-at-cursor', this.appendToBodyAtCursor)
			this.$emit('ready', editor)
		},
		onEditorInput(text) {
			if (text !== this.value) {
				logger.debug(`TextEditor input changed to <${text}>`)
				this.$emit('input', text)
			}
		},
		appendToBodyAtCursor(toAppend) {
			// https://ckeditor.com/docs/ckeditor5/latest/builds/guides/faq.html#where-are-the-editorinserthtml-and-editorinserttext-methods-how-to-insert-some-content
			const viewFragment = this.editorInstance.data.processor.toView(toAppend)
			const modelFragment = this.editorInstance.data.toModel(viewFragment)
			this.editorInstance.model.insertContent(modelFragment)
		},
		editorExecute(commandName, ...args) {
			if (this.editorInstance) {
				this.editorInstance.execute(commandName, ...args)
			} else {
				throw new Error('Impossible to execute a command before editor is ready.')
			}
		},
	},
}
</script>

<style lang="scss" scoped>
:deep(a) {
	color: #07d;
}
:deep(p) {
	cursor: text;
	margin: 0 !important;
}
</style>

<style>
/*
Overwrite the default z-index for CKEditor
https://github.com/ckeditor/ckeditor5/issues/1142
 */
 .ck .ck-reset {
	background: var(--color-main-background) !important;
 }
 .custom-item-username {
	color: var(--color-main-text) !important;
 }
 .link-title{
	color: var(--color-main-text) !important;
 }
 .link-icon{
	width : 20px !important;
	margin-right : 1em !important;
	filter : var(--background-invert-if-dark);
 }
 .custom-item {
	width : 100% !important;
	border-radius : 8px !important;
	padding : 4px 8px !important;
	display :block;
	background:var(--color-main-background)!important;
 }
 .custom-item:hover {
	background:var(--color-primary-element-light)!important;
 }
 .link-container{
	border-radius :8px !important;
	padding :4px 8px !important;
	display : block;
	width : 100% !important;
	background:var(--color-main-background)!important;
 }
 .link-container:hover {
	background:var(--color-primary-element-light)!important;
 }
:root {
	--ck-z-default: 10000;
	--ck-balloon-border-width:  0;
	}
</style>
