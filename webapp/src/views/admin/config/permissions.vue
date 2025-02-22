<template lang="pug">
.c-permissionconfig
	.ui-page-header
		h1 Permission Config
	bunt-progress-circular(size="huge", v-if="!error && !config")
	.error(v-if="error") We could not fetch the current configuration.
	bunt-tabs(v-if="config")
		bunt-tab(header="Global", v-scrollbar.y="")
			.permission-config
				.info
					h3 Roles assigned to traits globally
					p Users receive the following roles if they have the required traits in their token/ticket. Global roles are valid for #[b all rooms].
				trait-grants(:trait-grants="config.trait_grants", :config="config")
		bunt-tab(header="Per Room", v-scrollbar.y="")
			.permission-config
				.info
					h3 Roles assigned to traits #[i per room]
					p Users receive the following roles if they have the required traits in their token/ticket. #[b Per room roles] only apply for the corresponding room and they are applied #[b in addition to global roles].
				.searchbox
					bunt-input.search(name="search", placeholder="Search rooms", icon="search", v-model="search")
				.room-traits(v-for="room of filteredRooms")
					h4 {{ room.name }}
					trait-grants(:trait-grants="room.trait_grants", :config="config", @click.stop="", @changed="roomChanged(room)")
		bunt-tab(header="Roles", v-scrollbar.y="")
			.permission-config
				.info-wrapper
					.info
						h3 Roles
						p List of roles which can be assigned to users globally or per room via token/ticket traits. Each role can give a user certain permissions. Total permissions are computed from the sum of permissions from all granted roles. Permissions can #[b NOT] be subtracted by a role if granted by another role.
					.collapse-actions
				.role-definition(v-for="(val, key, index) of config.roles", @click="toggleRole(key)")
					.role-head
						bunt-icon-button.chevron {{ expandedRoles.includes(key) ? 'chevron-up' : 'chevron-down' }}
						h4 {{ key }}
						bunt-icon-button(@click.stop="deleteRole(key)") delete-outline
					.role-config-permissions(v-if="expandedRoles.includes(key)", @click.stop="")
						bunt-checkbox(v-for="p of config.available_permissions", :label="p", :value="val.includes(p)", name="p", @input="togglePermission(key, p, $event)")
				.role-add
					.role-head
						bunt-input(label="role name", v-model="newRoleName", name="newRoleName")
						bunt-button.btn-add-role(@click="addRole", :disabled="!newRoleName || newRoleName in config.roles") Add new role
	.ui-form-actions
		bunt-button.btn-save(@click="save", :loading="saving", :error-message="error") Save
</template>
<script>
import api from 'lib/api'
import fuzzysearch from 'lib/fuzzysearch'
import TraitGrants from './trait-grants'

export default {
	components: { TraitGrants },
	data () {
		return {
			config: null,
			expandedRoles: [],
			newRoleName: '',
			search: '',
			rooms: null,
			changedRoomIds: [],
			saving: false,
			error: null
		}
	},
	computed: {
		filteredRooms () {
			if (!this.rooms) return
			if (!this.search) return this.rooms
			return this.rooms.filter(room => room.id === this.search.trim() || fuzzysearch(this.search.toLowerCase(), room.name.toLowerCase()))
		}
	},
	async created () {
		// We don't use the global world object since it e.g. currently does not contain roles etc
		// TODO: Force reloading if world.updated is received from the server
		try {
			this.config = await api.call('world.config.get')
			this.rooms = await api.call('room.config.list')
		} catch (error) {
			this.error = error
			console.log(error)
		}
	},
	methods: {
		roomChanged (room) {
			if (!this.changedRoomIds.includes(room.id)) {
				this.changedRoomIds.push(room.id)
			}
		},
		toggleRole (role) {
			if (this.expandedRoles.includes(role)) {
				this.expandedRoles = this.expandedRoles.filter((r) => r !== role)
			} else {
				this.expandedRoles.push(role)
			}
		},
		deleteRole (role) {
			this.$delete(this.config.roles, role)
		},
		addRole () {
			if (this.newRoleName) {
				this.$set(this.config.roles, this.newRoleName, [])
				this.expandedRoles.push(this.newRoleName)
				this.newRoleName = ''
			}
		},
		togglePermission (role, perm, toggle) {
			if (toggle) {
				this.config.roles[role].push(perm)
			} else {
				this.$set(this.config.roles, role, this.config.roles[role].filter((i) => i !== perm))
			}
		},

		async save () {
			this.saving = true
			await api.call('world.config.patch', {
				roles: this.config.roles,
				trait_grants: this.config.trait_grants,
			})
			for (const room of this.rooms.filter((r) => this.changedRoomIds.includes(r.id))) {
				await api.call('room.config.patch', {
					room: room.id,
					trait_grants: room.trait_grants
				})
			}
			this.saving = false
			// TODO error handling
		},
	}
}
</script>
<style lang="stylus">
.c-permissionconfig
	flex: auto
	display: flex
	flex-direction: column
	.bunt-icon-button
		icon-button-style(style: clear)
	.bunt-input
		input-style(size: compact)
	.bunt-tabs
		tabs-style(active-color: var(--clr-primary), indicator-color: var(--clr-primary), background-color: transparent)
		margin-bottom: 0
	.bunt-tabs-header
		border-bottom: border-separator()
	.bunt-tabs, .bunt-tabs-body, .bunt-tab
		flex: auto
		display: flex
		flex-direction: column
		min-height: 0
	.searchbox
		padding: 0 8px 16px 8px
	.info
		padding: 16px 16px 0 16px
		p
			max-width: 960px
	.room-traits
		border: border-separator()
		margin: 0 8px 16px 8px
		border-radius: 4px
		h4
			margin: 8px 16px 0px 16px
	.role-definition
		border: border-separator()
		border-radius: 4px
		padding: 4px 16px
		margin: 0 8px 16px 8px
		.role-head
			display: flex
			flex-direction: row
			align-items: center
			cursor: pointer
			.bunt-icon-button.chevron
				position: relative
				left: -6px
			h4, .bunt-input
				flex: auto 1 1
	.role-add
		padding: 0 8px
	.role-config-permissions
		margin: 8px 16px
		display: grid
		// grid-auto-flow: column
		grid-template-columns: repeat(4, 1fr)
		// grid-template-rows: repeat(2, 150px)
		// grid-auto-rows: 36px
		gap: 4px
	.trait-grants
		width: 100%
	.btn-add-role
		themed-button-secondary()
</style>
