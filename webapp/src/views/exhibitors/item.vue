<template lang="pug">
scrollbars.c-exhibitor(y)
	.content-wrapper(v-if="exhibitor")
		.content
			img.banner(:src="exhibitor.banner_detail", v-if="exhibitor.banner_detail && !bannerIsVideo && !bannerIsFrame")
			.iframe-banner(v-else-if="bannerIsFrame")
				iframe(:src="bannerVideoSource", allowfullscreen, allow="fullscreen")
			.video-banner(v-else-if="bannerIsVideo")
				video(:src="exhibitor.banner_detail", autoplay, controls, loop)
			markdown-content.text(v-if="exhibitor.text_legacy", :markdown="exhibitor.text_legacy")
			rich-text-content.text(v-else, :content="exhibitor.text_content")
			.downloads(v-if="downloadLinks.length > 0")
				h2 {{ $t("Exhibitor:downloads-headline:text") }}
				a.download(v-for="link in downloadLinks", :href="link.url", target="_blank")
					.mdi(:class="`mdi-${getIconByFileEnding(link.url)}`")
					.filename {{ link.display_text }}
		.sidebar
			.header
				img.logo(:src="exhibitor.logo", v-if="exhibitor.logo")
				.heading
					h2.name {{ exhibitor.name }}
					h3.tagline(v-if="exhibitor.tagline") {{ exhibitor.tagline }}
			.social-media-links(v-if="exhibitor.social_media_links.length > 0")
				a.mdi(v-for="link in exhibitor.social_media_links", :class="`mdi-${link.display_text.toLowerCase()}`", :href="link.url", :title="link.display_text", target="_blank")
			table.external-links(v-if="profileLinks.length > 0")
				tr(v-for="link in profileLinks")
					th.name {{ link.display_text }}
					td: a(:href="link.url", target="_blank") {{ prettifyUrl(link.url) }}
			.join-room(v-if="exhibitor.highlighted_room_id")
				bunt-button(@click="joinRoom") {{ $t('Exhibition:room-button:label') }}
			template(v-if="exhibitor.staff.length > 0")
				.contact(v-if="hasPermission('world:exhibition.contact') && exhibitor.contact_enabled")
					bunt-button(@click="showContactPrompt = true", :tooltip="$t('Exhibition:contact-button:tooltip')") {{ $t('Exhibition:contact-button:label') }}
				.staff
					h3 {{ $t("Exhibitor:staff-headline:text") }}
					.user(v-for="user in exhibitor.staff", @click="showUserCard($event, user)")
						avatar(:user="user", :size="36")
						span.display-name {{ user ? user.profile.display_name : '' }}
	bunt-progress-circular(v-else, size="huge", :page="true")
	chat-user-card(v-if="selectedUser", ref="avatarCard", :sender="selectedUser", @close="selectedUser = null")
	transition(name="prompt")
		contact-exhibitor-prompt(v-if="showContactPrompt", @close="showContactPrompt = false", :exhibitor="exhibitor")
</template>
<script>
// TODO
// - user action for staff list?
import { mapState, mapGetters } from 'vuex'
import api from 'lib/api'
import Avatar from 'components/Avatar'
import ContactExhibitorPrompt from 'components/ContactExhibitorPrompt'
import ChatUserCard from 'components/ChatUserCard'
import MarkdownContent from 'components/MarkdownContent'
import RichTextContent from 'components/RichTextContent'
import { createPopper } from '@popperjs/core'
import { getIconByFileEnding } from 'lib/filetypes'

export default {
	components: { Avatar, ChatUserCard, ContactExhibitorPrompt, MarkdownContent, RichTextContent },
	props: {
		exhibitorId: String,
		exhibitorProp: Object,
	},
	data () {
		return {
			exhibitorApi: null,
			selectedUser: null,
			showContactPrompt: false,
			getIconByFileEnding
		}
	},
	computed: {
		...mapState(['user']),
		...mapGetters(['hasPermission']),

		exhibitor () {
			return this.exhibitorProp ? this.exhibitorProp : this.exhibitorApi
		},
		bannerIsFrame () {
			return this.exhibitor.banner_detail && (
				this.exhibitor.banner_detail.match('^https?://(www.)?(youtube.com/watch\\?v=|youtu.be/)([^&?]*)([&?].*)?$') ||
				this.exhibitor.banner_detail.match('^https?://(www.)?(vimeo.com)/(.*)$')
			)
		},
		bannerIsVideo () {
			return this.exhibitor.banner_detail && (
				this.exhibitor.banner_detail.match('^(.*)\\.webm$') ||
				this.exhibitor.banner_detail.match('^(.*)\\.mp4$')
			)
		},
		bannerVideoSource () {
			const ytMatch = this.exhibitor.banner_detail.match('^https?://(www.)?(youtube.com/watch\\?v=|youtu.be/)([^&?]*)([&?].*)?$')
			const vimeoMatch = this.exhibitor.banner_detail.match('^https?://(www.)?(vimeo.com)/(.*)$')
			if (ytMatch) {
				return 'https://www.youtube-nocookie.com/embed/' + ytMatch[3]
			}
			if (vimeoMatch) {
				return 'https://player.vimeo.com/video/' + vimeoMatch[3]
			}
			return this.exhibitor.banner_detail
		},
		profileLinks () {
			return this.exhibitor.links.filter((l) => (l.category === 'profile')).sort((a, b) => a.sorting_priority - b.sorting_priority)
		},
		downloadLinks () {
			return this.exhibitor.links.filter((l) => (l.category === 'download')).sort((a, b) => a.sorting_priority - b.sorting_priority)
		}
	},
	async created () {
		if (this.exhibitor) return
		this.exhibitorApi = (await api.call('exhibition.get', {exhibitor: this.exhibitorId})).exhibitor
	},
	methods: {
		prettifyUrl (link) {
			try {
				const url = new URL(link)
				return url.hostname + (url.pathname !== '/' ? url.pathname : '')
			} catch (e) {
				return link
			}
		},
		joinRoom () {
			this.$router.push({name: 'room', params: {roomId: this.exhibitor.highlighted_room_id}})
		},
		async showUserCard (event, user) {
			this.selectedUser = user
			await this.$nextTick()
			const target = event.target.closest('.user')
			createPopper(target, this.$refs.avatarCard.$refs.card, {
				placement: 'left-start',
				strategy: 'fixed',
				modifiers: [{
					name: 'flip',
					options: {
						flipVariations: false
					}
				}, {
					name: 'preventOverflow',
					options: {
						padding: 8
					}
				}]
			})
		}
	}
}
</script>
<style lang="stylus">
.c-exhibitor
	flex: auto
	display: flex
	background-color: $clr-white
	.content-wrapper
		display: flex
		justify-content: center
		padding: 8px
		> .content
			width: 100%
			max-width: 720px
			padding: 0 16px 0 0
			.iframe-banner, .video-banner
				padding-top: 56.25%
				position: relative
				height: 0
				overflow: hidden
				margin-top: 16px
				iframe, video
					position: absolute
					top: 0
					left: 0
					width: 100%
					height: 100%
					border: 0
			img.banner
				object-fit: contain
				width: 100%
				margin-top: 16px
	.sidebar
		flex: none
		min-height: min-content
		display: flex
		flex-direction: column
		width: 360px
		border: border-separator()
		border-radius: 4px
		margin-top: 16px
		align-self: flex-start
		.header
			flex: none
			display: flex
			flex-direction: column
			border-bottom: border-separator()
			padding: 8px
			img.logo
				object-fit: contain
				width: 100%
				height: 344px
				max-height: 344px
			.heading
				display: flex
				width: 100%
				flex-direction: column
				.name
					margin: 16px 0 8px 0
				.tagline
					margin: 0
		.social-media-links
			flex: none
			display: flex
			border-bottom: border-separator()
			padding: 4px 16px
			justify-content: center
			a
				font-size: 36px
				line-height: @font-size
		.external-links
			flex: none
			border-bottom: border-separator()
			tr
				height: 24px
			th
				font-weight: 400
				text-align: right
			td
				overflow: hidden
				white-space: nowrap
				text-overflow: ellipsis
				max-width: 0
				width: 100%
			.name
				white-space: nowrap
			a:hover
					text-decoration: underline
		.contact, .join-room
			flex: none
			padding: 8px
			display: flex
			flex-direction: column
			border-bottom: border-separator()
			.bunt-button
				themed-button-primary()
		.staff
			padding: 8px
			h3
				margin: 0
			.user
				display: flex
				align-items: center
				min-height: 48px
				cursor: pointer
				.display-name
					margin-left: 8px
					flex: auto
					ellipsis()
				.bunt-icon-button
					icon-button-style(style: clear)
				&:not(:hover) .bunt-icon-button
					display: none
	.downloads
		border: border-separator()
		border-radius: 4px
		display: flex
		flex-direction: column
		margin-top: 16px
		h2
			margin: 4px 8px
		.download
			display: flex
			align-items: center
			height: 56px
			font-weight: 600
			font-size: 16px
			border-top: border-separator()
			&:hover
				background-color: $clr-grey-100
				text-decoration: underline
			.mdi
				font-size: 36px
				margin: 0 4px
	+below('m')
		.content-wrapper
			flex-direction: column-reverse
			> .content
				max-width: none
				padding: 16px 0
		.sidebar
			width: auto
</style>
