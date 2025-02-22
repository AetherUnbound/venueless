<template lang="pug">
.c-channel(:class="{'has-call': hasCall}")
	.ui-page-header
		h2
			span.user(v-for="(u, key) in otherUsers")
				span(v-if="key !== 0") {{ ', ' }}
				.online-status(:class="onlineStatus[u.id] ? 'online' : (onlineStatus[u.id] === false ? 'offline' : 'unknown')", v-tooltip="onlineStatus[u.id] ? $t('UserAction:state.online:tooltip') : (onlineStatus[u.id] === false ? $t('UserAction:state.offline:tooltip') : '')")
				span {{ u.profile.display_name }}
		bunt-icon-button(@click="startCall", tooltip="start video call", tooltipPlacement="left") phone_outline
	.main
		media-source-placeholder.channel-call(v-if="hasCall")
		chat(:mode="hasCall ? 'compact' : 'standalone'", :module="{channel_id: channelId}", :showUserlist="false")
</template>
<script>
import { mapState } from 'vuex'
import api from 'lib/api'
import Chat from 'components/Chat'
import MediaSourcePlaceholder from 'components/MediaSourcePlaceholder'

export default {
	components: { Chat, MediaSourcePlaceholder },
	props: {
		channelId: String
	},
	data () {
		return {
			onlineStatus: {},
			pollOnlineStatusStatusTimeout: null,
		}
	},
	computed: {
		...mapState(['user']),
		...mapState('chat', ['joinedChannels', 'call']),
		hasCall () {
			return this.call && this.call.channel === this.channelId
		},
		channel () {
			return this.joinedChannels?.find(channel => channel.id === this.channelId)
		},
		otherUsers () {
			return this.channel?.members.filter(member => member.id !== this.user.id)
		}
	},
	async created () {
		await this.pollOnlineStatus()
	},
	destroyed () {
		if (this.pollOnlineStatusStatusTimeout) {
			window.clearTimeout(this.pollOnlineStatusStatusTimeout)
		}
	},
	methods: {
		startCall () {
			const channel = this.channel
			this.$store.dispatch('chat/startCall', {channel})
		},
		async pollOnlineStatus () {
			this.onlineStatus = (await api.call('user.online_status', {ids: this.otherUsers.map(u => u.id)}))
			this.pollOnlineStatusStatusTimeout = window.setTimeout(this.pollOnlineStatus, 20000)
		}
	}
}
</script>
<style lang="stylus">
.c-channel
	flex: auto
	display: flex
	flex-direction: column
	background-color: $clr-white
	min-height: 0
	.ui-page-header
		padding: 8px 16px
		justify-content: space-between
		h2
			margin: 0
			.online-status
				display: inline-block
				margin-right: 8px
				&::before
					content: ''
					display: inline-block
					background-color: $clr-grey-400
					width: 8px
					height: 8px
					border-radius: 50px
					vertical-align: middle
				&.online::before
					background-color: $clr-success
		.bunt-icon-button
			icon-button-style(style: clear)
	.main
		flex: auto
		display: flex
		min-height: 0
		.channel-call
			min-height: 0
			flex: auto 1 1
	&.has-call .c-chat
		flex: 380px 0 0

	+below('m')
		&.has-call
			.main
				flex-direction: column
			.c-chat
				flex: auto
				width: 100vw
				min-height: 0
			.channel-call
				height: var(--mobile-media-height)
				flex: var(--mobile-media-height) 0 0
</style>
