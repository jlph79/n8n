<script lang="ts" setup>
import Modal from './Modal.vue';
import { CREDENTIAL_EDIT_MODAL_KEY, SOURCE_CONTROL_PUSH_MODAL_KEY } from '@/constants';
import { computed, onMounted, ref } from 'vue';
import type { PropType } from 'vue';
import type { EventBus } from 'n8n-design-system/utils';
import type { SourceControlAggregatedFile } from '@/Interface';
import { useI18n, useLoadingService, useToast } from '@/composables';
import { useSourceControlStore } from '@/stores/sourceControl.store';
import { useUIStore } from '@/stores';
import { useRoute } from 'vue-router/composables';
import dateformat from 'dateformat';

const props = defineProps({
	data: {
		type: Object as PropType<{ eventBus: EventBus; status: SourceControlAggregatedFile[] }>,
		default: () => ({}),
	},
});

const defaultStagedFileTypes = ['tags', 'variables', 'credential'];

const loadingService = useLoadingService();
const uiStore = useUIStore();
const toast = useToast();
const { i18n } = useI18n();
const sourceControlStore = useSourceControlStore();
const route = useRoute();

const staged = ref<Record<string, boolean>>({});
const files = ref<SourceControlAggregatedFile[]>(props.data.status || []);

const commitMessage = ref('');
const loading = ref(true);
const context = ref<'workflow' | 'workflows' | 'credentials' | string>('');

const statusToBadgeThemeMap = {
	created: 'success',
	deleted: 'danger',
	modified: 'warning',
	renamed: 'warning',
};

const isSubmitDisabled = computed(() => {
	return !commitMessage.value || Object.values(staged.value).every((value) => !value);
});

const workflowId = computed(() => {
	if (context.value === 'workflow') {
		return route.params.name as string;
	}

	return '';
});

const sortedFiles = computed(() => {
	const statusPriority = {
		deleted: 1,
		modified: 2,
		renamed: 3,
		created: 4,
	};

	return [...files.value].sort((a, b) => {
		if (context.value === 'workflow') {
			if (a.id === workflowId.value) {
				return -1;
			} else if (b.id === workflowId.value) {
				return 1;
			}
		}

		if (statusPriority[a.status] < statusPriority[b.status]) {
			return -1;
		} else if (statusPriority[a.status] > statusPriority[b.status]) {
			return 1;
		}

		return a.updatedAt < b.updatedAt ? 1 : a.updatedAt > b.updatedAt ? -1 : 0;
	});
});

const selectAll = computed(() => {
	return files.value.every((file) => staged.value[file.file]);
});

const workflowFiles = computed(() => {
	return files.value.filter((file) => file.type === 'workflow');
});

const stagedWorkflowFiles = computed(() => {
	return workflowFiles.value.filter((workflow) => staged.value[workflow.file]);
});

const selectAllIndeterminate = computed(() => {
	return (
		stagedWorkflowFiles.value.length > 0 &&
		stagedWorkflowFiles.value.length < workflowFiles.value.length
	);
});

onMounted(async () => {
	context.value = getContext();
	try {
		staged.value = getStagedFilesByContext(files.value);
	} catch (error) {
		toast.showError(error, i18n.baseText('error'));
	} finally {
		loading.value = false;
	}
});

function onToggleSelectAll() {
	if (selectAll.value) {
		files.value.forEach((file) => {
			if (!defaultStagedFileTypes.includes(file.type)) {
				staged.value[file.file] = false;
			}
		});
	} else {
		files.value.forEach((file) => {
			if (!defaultStagedFileTypes.includes(file.type)) {
				staged.value[file.file] = true;
			}
		});
	}
}

function getContext() {
	if (route.fullPath.startsWith('/workflows')) {
		return 'workflows';
	} else if (
		route.fullPath.startsWith('/credentials') ||
		uiStore.modals[CREDENTIAL_EDIT_MODAL_KEY].open
	) {
		return 'credentials';
	} else if (route.fullPath.startsWith('/workflow/')) {
		return 'workflow';
	}

	return '';
}

function getStagedFilesByContext(files: SourceControlAggregatedFile[]): Record<string, boolean> {
	const stagedFiles = files.reduce((acc, file) => {
		acc[file.file] = false;
		return acc;
	}, {});

	files.forEach((file) => {
		if (defaultStagedFileTypes.includes(file.type)) {
			stagedFiles[file.file] = true;
		}

		if (context.value === 'workflow' && file.type === 'workflow' && file.id === workflowId.value) {
			stagedFiles[file.file] = true;
		} else if (context.value === 'workflows' && file.type === 'workflow') {
			stagedFiles[file.file] = true;
		}
	});

	return stagedFiles;
}

function setStagedStatus(file: SourceControlAggregatedFile, status: boolean) {
	staged.value = {
		...staged.value,
		[file.file]: status,
	};
}

function close() {
	uiStore.closeModal(SOURCE_CONTROL_PUSH_MODAL_KEY);
}

function renderUpdatedAt(file: SourceControlAggregatedFile) {
	const currentYear = new Date().getFullYear();

	return i18n.baseText('settings.sourceControl.lastUpdated', {
		interpolate: {
			date: dateformat(
				file.updatedAt,
				`d mmm${file.updatedAt.startsWith(currentYear) ? '' : ', yyyy'}`,
			),
			time: dateformat(file.updatedAt, 'HH:MM'),
		},
	});
}

async function commitAndPush() {
	const fileNames = files.value.filter((file) => staged.value[file.file]).map((file) => file.file);

	loadingService.startLoading(i18n.baseText('settings.sourceControl.loading.push'));
	close();

	try {
		await sourceControlStore.pushWorkfolder({
			commitMessage: commitMessage.value,
			fileNames,
		});

		toast.showToast({
			title: i18n.baseText('settings.sourceControl.modals.push.success.title'),
			message: i18n.baseText('settings.sourceControl.modals.push.success.description'),
			type: 'success',
		});
	} catch (error) {
		toast.showError(error, i18n.baseText('error'));
	} finally {
		loadingService.stopLoading();
	}
}
</script>

<template>
	<Modal
		width="812px"
		:title="i18n.baseText('settings.sourceControl.modals.push.title')"
		:eventBus="data.eventBus"
		:name="SOURCE_CONTROL_PUSH_MODAL_KEY"
	>
		<template #content>
			<div :class="$style.container">
				<n8n-text>
					{{ i18n.baseText('settings.sourceControl.modals.push.description') }}
					<span v-if="context">
						{{ i18n.baseText(`settings.sourceControl.modals.push.description.${context}`) }}
					</span>
					<n8n-link
						:href="i18n.baseText('settings.sourceControl.modals.push.description.learnMore.url')"
					>
						{{ i18n.baseText('settings.sourceControl.modals.push.description.learnMore') }}
					</n8n-link>
				</n8n-text>

				<div v-if="workflowFiles.length > 0">
					<div class="mt-l mb-2xs">
						<n8n-checkbox
							:indeterminate="selectAllIndeterminate"
							:value="selectAll"
							@input="onToggleSelectAll"
						>
							<n8n-text bold tag="strong">
								{{ i18n.baseText('settings.sourceControl.modals.push.workflowsToCommit') }}
							</n8n-text>
							<n8n-text tag="strong" v-show="workflowFiles.length > 0">
								({{ stagedWorkflowFiles.length }}/{{ workflowFiles.length }})
							</n8n-text>
						</n8n-checkbox>
					</div>
					<n8n-card
						v-for="file in sortedFiles"
						v-show="!defaultStagedFileTypes.includes(file.type)"
						:key="file.file"
						:class="$style.listItem"
						@click="setStagedStatus(file, !staged[file.file])"
					>
						<div :class="$style.listItemBody">
							<n8n-checkbox
								:value="staged[file.file]"
								:class="$style.listItemCheckbox"
								@input="setStagedStatus(file, !staged[file.file])"
							/>
							<div>
								<n8n-text v-if="file.status === 'deleted'" color="text-light">
									<span v-if="file.type === 'workflow'"> Deleted Workflow: </span>
									<span v-if="file.type === 'credential'"> Deleted Credential: </span>
									<strong>{{ file.id }}</strong>
								</n8n-text>
								<n8n-text bold v-else>
									{{ file.name }}
								</n8n-text>
								<div v-if="file.updatedAt">
									<n8n-text color="text-light" size="small">
										{{ renderUpdatedAt(file) }}
									</n8n-text>
								</div>
								<div v-if="file.conflict">
									<n8n-text color="danger" size="small">
										{{ i18n.baseText('settings.sourceControl.modals.push.overrideVersionInGit') }}
									</n8n-text>
								</div>
							</div>
							<div :class="$style.listItemStatus">
								<n8n-badge class="mr-2xs" v-if="workflowId === file.id && file.type === 'workflow'">
									Current workflow
								</n8n-badge>
								<n8n-badge :theme="statusToBadgeThemeMap[file.status] || 'default'">
									{{ file.status }}
								</n8n-badge>
							</div>
						</div>
					</n8n-card>

					<n8n-text bold tag="p" class="mt-l mb-2xs">
						{{ i18n.baseText('settings.sourceControl.modals.push.commitMessage') }}
					</n8n-text>
					<n8n-input
						type="text"
						v-model="commitMessage"
						:placeholder="
							i18n.baseText('settings.sourceControl.modals.push.commitMessage.placeholder')
						"
					/>
				</div>
				<div v-else-if="!loading">
					<n8n-callout class="mt-l">
						{{ i18n.baseText('settings.sourceControl.modals.push.everythingIsUpToDate') }}
					</n8n-callout>
				</div>
			</div>
		</template>

		<template #footer>
			<div :class="$style.footer">
				<n8n-button type="tertiary" class="mr-2xs" @click="close">
					{{ i18n.baseText('settings.sourceControl.modals.push.buttons.cancel') }}
				</n8n-button>
				<n8n-button type="primary" :disabled="isSubmitDisabled" @click="commitAndPush">
					{{ i18n.baseText('settings.sourceControl.modals.push.buttons.save') }}
				</n8n-button>
			</div>
		</template>
	</Modal>
</template>

<style module lang="scss">
.container > * {
	overflow-wrap: break-word;
}

.actionButtons {
	display: flex;
	justify-content: flex-end;
	align-items: center;
}

.listItem {
	margin-top: var(--spacing-2xs);
	margin-bottom: var(--spacing-2xs);
	cursor: pointer;
	transition: border 0.3s ease;
	padding: var(--spacing-xs);

	&:hover {
		border-color: var(--color-foreground-dark);
	}

	&:first-child {
		margin-top: 0;
	}

	&:last-child {
		margin-bottom: 0;
	}
}

.listItemBody {
	display: flex;
	flex-direction: row;
	align-items: center;
}

.listItemCheckbox {
	display: inline-flex !important;
	margin-bottom: 0 !important;
	margin-right: var(--spacing-2xs) !important;
}

.listItemStatus {
	margin-left: auto;
}

.footer {
	display: flex;
	flex-direction: row;
	justify-content: flex-end;
}
</style>