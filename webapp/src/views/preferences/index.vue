<template lang="pug">
.v-preferences
	.ui-page-header
		h1 {{ $t('preferences/index:heading') }}
	scrollbars(y)
		.inputs
			.avatar-wrapper
				avatar(:user="{profile}", :size="128")
				bunt-button#btn-change-avatar(@click="showChangeAvatar = true") {{ $t('preferences/index:btn-change-avatar:label') }}
			bunt-input.display-name(name="displayName", :label="$t('profile/GreetingPrompt:displayname:label')", v-model.trim="profile.display_name", :validation="$v.profile.display_name")
			change-additional-fields(v-model="profile.fields")
			template(v-if="languages")
				h2 {{ $t('preferences/index:interface-language:header') }}
				bunt-select#select-interface-language(name="interface-language", v-model="interfaceLanguage", :options="languages", option-value="code", option-label="nativeLabel")
			h2 {{ $t('preferences/index:notifications:header') }}
			p {{ $t('preferences/index:notifications:description') }}
			bunt-button#btn-enable-desktop-notifications(v-if="notificationPermission === 'default'", icon="bell", @click="$store.dispatch('notifications/askForPermission')") {{ $t('preferences/index:btn-enable-desktop-notifications:label') }}
			.notification-permission-denied(v-else-if="notificationPermission === 'denied'") {{ $t('preferences/index:notification-permission-denied-warning') }}
			template(v-else)
				bunt-switch(name="notificationSettings.notify", :label="$t('preferences/index:switch-enable-desktop-notifications:label')", v-model="notificationSettings.notify")
				bunt-switch(name="notificationSettings.playSounds", :label="$t('preferences/index:switch-enable-desktop-notification-sound:label')", v-model="notificationSettings.playSounds")
			h2 {{ $t('preferences/index:autoplay:header') }}
			p {{ $t('preferences/index:autoplay:description') }}
			bunt-switch(name="autoplay", v-model="autoplay", :label="$t('preferences/index:switch-autoplay:label')")
	.ui-form-actions
		bunt-button#btn-save(:disabled="$v.$invalid && $v.$dirty", :loading="saving", @click="save") {{ $t('preferences/index:btn-save:label') }}
	transition(name="prompt")
		prompt.change-avatar-prompt(v-if="showChangeAvatar", @close="showChangeAvatar = false")
			.content
				change-avatar(ref="avatar", v-model="profile.avatar", @blockSave="blockSave = $event")
				.actions
					bunt-button#btn-cancel(@click="showChangeAvatar = false") {{ $t('Prompt:cancel:label') }}
					bunt-button#btn-upload(:loading="savingAvatar", :disabled="blockSave", @click="uploadAvatar") {{ $t('preferences/index:btn-upload-save:label') }}
</template>
<script>
// TODO communicate language change to other tabs?
import { mapState } from 'vuex'
import cloneDeep from 'lodash/cloneDeep'
import config from 'config'
import { locales } from 'locales'
import Avatar from 'components/Avatar'
import Prompt from 'components/Prompt'
import ChangeAvatar from 'components/profile/ChangeAvatar'
import ChangeAdditionalFields from 'components/profile/ChangeAdditionalFields'
import ConnectGravatar from 'components/profile/ConnectGravatar'
import { required } from 'buntpapier/src/vuelidate/validators'

export default {
	components: { Avatar, Prompt, ChangeAvatar, ChangeAdditionalFields, ConnectGravatar},
	data () {
		return {
			profile: null,
			interfaceLanguage: this.$i18n.resolvedLanguage,
			notificationSettings: cloneDeep(this.$store.state.notifications.settings),
			autoplay: true,
			showChangeAvatar: false,
			savingAvatar: false,
			blockSave: false,
			saving: false
		}
	},
	validations: {
		profile: {
			display_name: {
				required: required('Display name cannot be empty')
			}
		}
	},
	computed: {
		...mapState(['user', 'world']),
		...mapState('notifications', {
			notificationPermission: 'permission'
		}),
		languages () {
			if (!config.locales?.length) return null
			return locales.filter(locale => config.locales.includes(locale.code))
		}
	},
	created () {
		this.profile = Object.assign({}, this.user.profile)
		this.autoplay = this.$store.state.autoplay
		if (!this.profile.avatar || (!this.profile.avatar.url && !this.profile.avatar.identicon)) {
			this.profile.avatar = {
				identicon: this.user.id
			}
		}
	},
	methods: {
		async uploadAvatar () {
			this.savingAvatar = true
			await this.$refs.avatar.update()
			await this.$store.dispatch('updateUser', {profile: Object.assign({}, this.user.profile, {avatar: this.profile.avatar})})
			this.showChangeAvatar = false
			this.savingAvatar = false
		},
		async save () {
			this.$v.$touch()
			if (this.$v.$invalid) return
			this.saving = true
			await this.$store.dispatch('updateUser', {profile: this.profile})
			this.$store.dispatch('notifications/updateSettings', this.notificationSettings)
			this.$store.dispatch('setAutoplay', this.autoplay)
			localStorage.userLanguage = this.interfaceLanguage
			try {
				await this.$store.dispatch('updateUserLocale', this.interfaceLanguage)
			} catch (error) {
				console.error(error)
			}
			this.saving = false
		}
	}
}
</script>
<style lang="stylus">
.v-preferences
	background-color: $clr-white
	display: flex
	flex-direction: column
	flex: auto
	min-height: 0
	.scroll-content
		padding: 16px 32px
	h1
		margin: 0
	h2
		font-size: 20px
		font-weight: 500
		border-bottom: border-separator()
	.avatar-wrapper
		display: flex
		align-items: center
		#btn-change-avatar
			themed-button-secondary()
			margin-left: 16px
	.inputs
		width: 420px
		display: flex
		flex-direction: column
		.permission-info
			font-size 13px
			line-height 18px
			padding 6px 0px 6px 16px
			color: $clr-red
	#btn-enable-desktop-notifications
		themed-button-secondary()
	.notification-permission-denied
		background-color: $clr-red-a700
		color: $clr-primary-text-dark
		border-radius: 4px
		padding: 16px
		font-weight: 500
	#btn-save
		themed-button-primary()

	.change-avatar-prompt
		.content
			display: flex
			flex-direction: column
			padding: 48px 32px 32px
		.actions
			margin-top: 32px
			align-self: stretch
			display: flex
			justify-content: flex-end
		#btn-cancel
			themed-button-secondary()
			margin-right: 8px
		#btn-upload
			themed-button-primary()
	+below('s')
		.inputs
			width: auto
</style>
