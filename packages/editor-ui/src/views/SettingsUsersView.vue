<template>
	<div :class="$style.container">
		<div>
			<n8n-heading size="2xlarge">{{ $locale.baseText('settings.users') }}</n8n-heading>
			<div :class="$style.buttonContainer" v-if="!usersStore.showUMSetupWarning">
				<n8n-tooltip :disabled="settingsStore.isSmtpSetup" placement="bottom">
					<template #content>
						<i18n path="settings.users.setupSMTPToInviteUsers" tag="span">
							<template #action>
								<a
									href="https://docs.n8n.io/reference/user-management.html#step-one-smtp"
									target="_blank"
									v-text="$locale.baseText('settings.users.setupSMTPToInviteUsers.instructions')"
								/>
							</template>
						</i18n>
					</template>
					<div>
						<n8n-button
							:label="$locale.baseText('settings.users.invite')"
							@click="onInvite"
							size="large"
							:disabled="!settingsStore.isSmtpSetup"
						/>
					</div>
				</n8n-tooltip>
			</div>
		</div>
		<div v-if="usersStore.showUMSetupWarning" :class="$style.setupInfoContainer">
			<n8n-action-box
				:heading="$locale.baseText('settings.users.setupToInviteUsers')"
				:buttonText="$locale.baseText('settings.users.setupMyAccount')"
				:description="`${
					isSharingEnabled ? '' : $locale.baseText('settings.users.setupToInviteUsersInfo')
				}${$locale.baseText('settings.users.setupSMTPInfo')}`"
				@click="redirectToSetup"
			/>
		</div>
		<div :class="$style.usersContainer" v-else>
			<PageAlert
				v-if="!settingsStore.isSmtpSetup"
				:message="$locale.baseText('settings.users.smtpToAddUsersWarning')"
				:popupClass="$style.alert"
			/>
			<n8n-users-list
				:users="usersStore.allUsers"
				:currentUserId="usersStore.currentUserId"
				@delete="onDelete"
				@reinvite="onReinvite"
			/>
		</div>
		<feature-coming-soon
			v-for="fakeDoorFeature in fakeDoorFeatures"
			:key="fakeDoorFeature.id"
			:featureId="fakeDoorFeature.id"
			showTitle
		/>
	</div>
</template>

<script lang="ts">
import { EnterpriseEditionFeature, INVITE_USER_MODAL_KEY, VIEWS } from '@/constants';

import PageAlert from '../components/PageAlert.vue';
import FeatureComingSoon from '@/components/FeatureComingSoon.vue';
import { IFakeDoor, IUser } from '@/Interface';
import mixins from 'vue-typed-mixins';
import { showMessage } from '@/mixins/showMessage';
import { mapStores } from 'pinia';
import { useUIStore } from '@/stores/ui';
import { useSettingsStore } from '@/stores/settings';
import { useUsersStore } from '@/stores/users';

export default mixins(showMessage).extend({
	name: 'SettingsUsersView',
	components: {
		PageAlert,
		FeatureComingSoon,
	},
	async mounted() {
		if (!this.usersStore.showUMSetupWarning) {
			await this.usersStore.fetchUsers();
		}
	},
	computed: {
		...mapStores(useSettingsStore, useUIStore, useUsersStore),
		isSharingEnabled() {
			return this.settingsStore.isEnterpriseFeatureEnabled(EnterpriseEditionFeature.Sharing);
		},
		fakeDoorFeatures(): IFakeDoor[] {
			return this.uiStore.getFakeDoorByLocation('settings/users');
		},
	},
	methods: {
		redirectToSetup() {
			this.$router.push({ name: VIEWS.SETUP });
		},
		onInvite() {
			this.uiStore.openModal(INVITE_USER_MODAL_KEY);
		},
		async onDelete(userId: string) {
			const user = this.usersStore.getUserById(userId) as IUser | null;
			if (user) {
				this.uiStore.openDeleteUserModal(userId);
			}
		},
		async onReinvite(userId: string) {
			const user = this.usersStore.getUserById(userId) as IUser | null;
			if (user) {
				try {
					await this.usersStore.reinviteUser({ id: user.id });

					this.$showToast({
						type: 'success',
						title: this.$locale.baseText('settings.users.inviteResent'),
						message: this.$locale.baseText('settings.users.emailSentTo', {
							interpolate: { email: user.email || '' },
						}),
					});
				} catch (e) {
					this.$showError(e, this.$locale.baseText('settings.users.userReinviteError'));
				}
			}
		},
	},
});
</script>

<style lang="scss" module>
.container {
	height: 100%;
	padding-right: var(--spacing-2xs);

	> * {
		margin-bottom: var(--spacing-2xl);
	}
}

.usersContainer {
	> * {
		margin-bottom: var(--spacing-2xs);
	}
}

.buttonContainer {
	display: inline-block;
	float: right;
	margin-bottom: var(--spacing-l);
}

.setupInfoContainer {
	max-width: 728px;
}

.alert {
	left: calc(50% + 100px);
}
</style>
