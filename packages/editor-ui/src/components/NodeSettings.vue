<template>
	<div :class="{'node-settings': true, 'dragging': dragging}" @keydown.stop>
		<div :class="$style.header">
			<div class="header-side-menu">
				<NodeTitle class="node-name" :value="node.name" :nodeType="nodeType" @input="nameChanged" :readOnly="isReadOnly"></NodeTitle>
				<div
					v-if="!isReadOnly"
				>
					<NodeExecuteButton :nodeName="node.name" @execute="onNodeExecute" size="small" />
				</div>
			</div>
			<NodeSettingsTabs v-model="openPanel" :nodeType="nodeType" :sessionId="sessionId" />
		</div>
		<div class="node-is-not-valid" v-if="node && !nodeValid">
			<n8n-text>
				{{
					$locale.baseText(
						'nodeSettings.theNodeIsNotValidAsItsTypeIsUnknown',
						{ interpolate: { nodeType: node.type } },
					)
				}}
			</n8n-text>
		</div>
		<div class="node-parameters-wrapper" v-if="node && nodeValid">
			<div v-show="openPanel === 'params'">
				<node-webhooks
					:node="node"
					:nodeType="nodeType"
				/>
				<parameter-input-list
					:parameters="parametersNoneSetting"
					:hideDelete="true"
					:nodeValues="nodeValues" path="parameters" @valueChanged="valueChanged"
				>
					<node-credentials
						:node="node"
						@credentialSelected="credentialSelected"
					/>
				</parameter-input-list>
				<div v-if="parametersNoneSetting.length === 0" class="no-parameters">
					<n8n-text>
						{{ $locale.baseText('nodeSettings.thisNodeDoesNotHaveAnyParameters') }}
					</n8n-text>
				</div>

				<div v-if="isCustomApiCallSelected(nodeValues)" class="parameter-item parameter-notice">
					<n8n-notice
						:content="$locale.baseText(
							'nodeSettings.useTheHttpRequestNode',
							{ interpolate: { nodeTypeDisplayName: nodeType.displayName } }
						)"
					/>
				</div>

			</div>
			<div v-show="openPanel === 'settings'">
				<parameter-input-list :parameters="parametersSetting" :nodeValues="nodeValues" path="parameters" @valueChanged="valueChanged" />
				<parameter-input-list :parameters="nodeSettings" :hideDelete="true" :nodeValues="nodeValues" path="" @valueChanged="valueChanged" />
			</div>
		</div>
	</div>
</template>

<script lang="ts">
import Vue from 'vue';
import {
	INodeTypeDescription,
	INodeParameters,
	INodeProperties,
	NodeHelpers,
	NodeParameterValue,
} from 'n8n-workflow';
import {
	INodeUi,
	INodeUpdatePropertiesInformation,
	IUpdateInformation,
} from '@/Interface';

import NodeTitle from '@/components/NodeTitle.vue';
import ParameterInputFull from '@/components/ParameterInputFull.vue';
import ParameterInputList from '@/components/ParameterInputList.vue';
import NodeCredentials from '@/components/NodeCredentials.vue';
import NodeSettingsTabs from '@/components/NodeSettingsTabs.vue';
import NodeWebhooks from '@/components/NodeWebhooks.vue';
import { get, set, unset } from 'lodash';

import { externalHooks } from '@/components/mixins/externalHooks';
import { genericHelpers } from '@/components/mixins/genericHelpers';
import { nodeHelpers } from '@/components/mixins/nodeHelpers';

import mixins from 'vue-typed-mixins';
import NodeExecuteButton from './NodeExecuteButton.vue';

export default mixins(
	externalHooks,
	genericHelpers,
	nodeHelpers,
)
	.extend({
		name: 'NodeSettings',
		components: {
			NodeTitle,
			NodeCredentials,
			ParameterInputFull,
			ParameterInputList,
			NodeSettingsTabs,
			NodeWebhooks,
			NodeExecuteButton,
		},
		computed: {
			nodeType (): INodeTypeDescription | null {
				if (this.node) {
					return this.$store.getters.nodeType(this.node.type, this.node.typeVersion);
				}

				return null;
			},
			nodeTypeName(): string {
				if (this.nodeType) {
					const shortNodeType = this.$locale.shortNodeType(this.nodeType.name);

					return this.$locale.headerText({
						key: `headers.${shortNodeType}.displayName`,
						fallback: this.nodeType.name,
					});
				}

				return '';
			},
			nodeTypeDescription (): string {
				if (this.nodeType && this.nodeType.description) {
					const shortNodeType = this.$locale.shortNodeType(this.nodeType.name);

					return this.$locale.headerText({
						key: `headers.${shortNodeType}.description`,
						fallback: this.nodeType.description,
					});
				} else {
					return this.$locale.baseText('nodeSettings.noDescriptionFound');
				}
			},
			headerStyle (): object {
				if (!this.node) {
					return {};
				}

				return {
					'background-color': this.node.color,
				};
			},
			node (): INodeUi {
				return this.$store.getters.activeNode;
			},
			parametersSetting (): INodeProperties[] {
				return this.parameters.filter((item) => {
					return item.isNodeSetting;
				});
			},
			parametersNoneSetting (): INodeProperties[] {
				return this.parameters.filter((item) => {
					return !item.isNodeSetting;
				});
			},
			parameters (): INodeProperties[] {
				if (this.nodeType === null) {
					return [];
				}

				return this.nodeType.properties;
			},
		},
		props: {
			eventBus: {
			},
			dragging: {
				type: Boolean,
			},
			sessionId: {
				type: String,
			},
		},
		data () {
			return {
				nodeValid: true,
				nodeColor: null,
				openPanel: 'params',
				nodeValues: {
					color: '#ff0000',
					alwaysOutputData: false,
					executeOnce: false,
					notesInFlow: false,
					continueOnFail: false,
					retryOnFail: false,
					maxTries: 3,
					waitBetweenTries: 1000,
					notes: '',
					parameters: {},
				} as INodeParameters,

				nodeSettings: [
					{
						displayName: this.$locale.baseText('nodeSettings.notes.displayName'),
						name: 'notes',
						type: 'string',
						typeOptions: {
							rows: 5,
						},
						default: '',
						noDataExpression: true,
						description: this.$locale.baseText('nodeSettings.notes.description'),
					},
					{
						displayName: this.$locale.baseText('nodeSettings.notesInFlow.displayName'),
						name: 'notesInFlow',
						type: 'boolean',
						default: false,
						noDataExpression: true,
						description: this.$locale.baseText('nodeSettings.notesInFlow.description'),
					},
					{
						displayName: this.$locale.baseText('nodeSettings.alwaysOutputData.displayName'),
						name: 'alwaysOutputData',
						type: 'boolean',
						default: false,
						noDataExpression: true,
						description: this.$locale.baseText('nodeSettings.alwaysOutputData.description'),
					},
					{
						displayName: this.$locale.baseText('nodeSettings.executeOnce.displayName'),
						name: 'executeOnce',
						type: 'boolean',
						default: false,
						noDataExpression: true,
						description: this.$locale.baseText('nodeSettings.executeOnce.description'),
					},
					{
						displayName: this.$locale.baseText('nodeSettings.retryOnFail.displayName'),
						name: 'retryOnFail',
						type: 'boolean',
						default: false,
						noDataExpression: true,
						description: this.$locale.baseText('nodeSettings.retryOnFail.description'),
					},
					{
						displayName: this.$locale.baseText('nodeSettings.maxTries.displayName'),
						name: 'maxTries',
						type: 'number',
						typeOptions: {
							minValue: 2,
							maxValue: 5,
						},
						default: 3,
						displayOptions: {
							show: {
								retryOnFail: [
									true,
								],
							},
						},
						noDataExpression: true,
						description: this.$locale.baseText('nodeSettings.maxTries.description'),
					},
					{
						displayName: this.$locale.baseText('nodeSettings.waitBetweenTries.displayName'),
						name: 'waitBetweenTries',
						type: 'number',
						typeOptions: {
							minValue: 0,
							maxValue: 5000,
						},
						default: 1000,
						displayOptions: {
							show: {
								retryOnFail: [
									true,
								],
							},
						},
						noDataExpression: true,
						description: this.$locale.baseText('nodeSettings.waitBetweenTries.description'),
					},
					{
						displayName: this.$locale.baseText('nodeSettings.continueOnFail.displayName'),
						name: 'continueOnFail',
						type: 'boolean',
						default: false,
						noDataExpression: true,
						description: this.$locale.baseText('nodeSettings.continueOnFail.description'),
					},
				] as INodeProperties[],

			};
		},
		watch: {
			node (newNode, oldNode) {
				this.setNodeValues();
			},
		},
		methods: {
			onNodeExecute () {
				this.$emit('execute');
			},
			setValue (name: string, value: NodeParameterValue) {
				const nameParts = name.split('.');
				let lastNamePart: string | undefined = nameParts.pop();

				let isArray = false;
				if (lastNamePart !== undefined && lastNamePart.includes('[')) {
					// It incldues an index so we have to extract it
					const lastNameParts = lastNamePart.match(/(.*)\[(\d+)\]$/);
					if (lastNameParts) {
						nameParts.push(lastNameParts[1]);
						lastNamePart = lastNameParts[2];
						isArray = true;
					}
				}

				// Set the value via Vue.set that everything updates correctly in the UI
				if (nameParts.length === 0) {
					// Data is on top level
					if (value === null) {
						// Property should be deleted
						// @ts-ignore
						Vue.delete(this.nodeValues, lastNamePart);
					} else {
						// Value should be set
						// @ts-ignore
						Vue.set(this.nodeValues, lastNamePart, value);
					}
				} else {
					// Data is on lower level
					if (value === null) {
						// Property should be deleted
						// @ts-ignore
						let tempValue = get(this.nodeValues, nameParts.join('.')) as INodeParameters | NodeParameters[];
						Vue.delete(tempValue as object, lastNamePart as string);

						if (isArray === true && (tempValue as INodeParameters[]).length === 0) {
							// If a value from an array got delete and no values are left
							// delete also the parent
							lastNamePart = nameParts.pop();
							tempValue = get(this.nodeValues, nameParts.join('.')) as INodeParameters;
							Vue.delete(tempValue as object, lastNamePart as string);
						}
					} else {
						// Value should be set
						if (typeof value === 'object') {
							// @ts-ignore
							Vue.set(get(this.nodeValues, nameParts.join('.')), lastNamePart, JSON.parse(JSON.stringify(value)));
						} else {
							// @ts-ignore
							Vue.set(get(this.nodeValues, nameParts.join('.')), lastNamePart, value);
						}
					}
				}
			},
			credentialSelected (updateInformation: INodeUpdatePropertiesInformation) {
				// Update the values on the node
				this.$store.commit('updateNodeProperties', updateInformation);

				const node = this.$store.getters.getNodeByName(updateInformation.name);

				// Update the issues
				this.updateNodeCredentialIssues(node);

				this.$externalHooks().run('nodeSettings.credentialSelected', { updateInformation });
			},
			nameChanged(name: string) {
				// @ts-ignore
				this.valueChanged({
					value: name,
					name: 'name',
				});
			},
			valueChanged (parameterData: IUpdateInformation) {
				let newValue: NodeParameterValue;
				if (parameterData.hasOwnProperty('value')) {
					// New value is given
					newValue = parameterData.value;
				} else {
					// Get new value from nodeData where it is set already
					newValue = get(this.nodeValues, parameterData.name) as NodeParameterValue;
				}

				// Save the node name before we commit the change because
				// we need the old name to rename the node properly
				const nodeNameBefore = parameterData.node || this.node.name;
				const node = this.$store.getters.getNodeByName(nodeNameBefore);
				if (parameterData.name === 'name') {
					// Name of node changed so we have to set also the new node name as active

					// Update happens in NodeView so emit event
					const sendData = {
						value: newValue,
						oldValue: nodeNameBefore,
						name: parameterData.name,
					};
					this.$emit('valueChanged', sendData);

				} else if (parameterData.name.startsWith('parameters.')) {
					// A node parameter changed

					const nodeType = this.$store.getters.nodeType(node.type, node.typeVersion) as INodeTypeDescription | null;
					if (!nodeType) {
						return;
					}

					// Get only the parameters which are different to the defaults
					let nodeParameters = NodeHelpers.getNodeParameters(nodeType.properties, node.parameters, false, false, node);
					const oldNodeParameters = Object.assign({}, nodeParameters);

					// Copy the data because it is the data of vuex so make sure that
					// we do not edit it directly
					nodeParameters = JSON.parse(JSON.stringify(nodeParameters));

					// Remove the 'parameters.' from the beginning to just have the
					// actual parameter name
					const parameterPath = parameterData.name.split('.').slice(1).join('.');

					// Check if the path is supposed to change an array and if so get
					// the needed data like path and index
					const parameterPathArray = parameterPath.match(/(.*)\[(\d+)\]$/);

					// Apply the new value
					if (parameterData.value === undefined && parameterPathArray !== null) {
						// Delete array item
						const path = parameterPathArray[1];
						const index = parameterPathArray[2];
						const data = get(nodeParameters, path);

						if (Array.isArray(data)) {
							data.splice(parseInt(index, 10), 1);
							Vue.set(nodeParameters as object, path, data);
						}
					} else {
						if (newValue === undefined) {
							unset(nodeParameters as object, parameterPath);
						} else {
							set(nodeParameters as object, parameterPath, newValue);
						}
					}

					// Get the parameters with the now new defaults according to the
					// from the user actually defined parameters
					nodeParameters = NodeHelpers.getNodeParameters(nodeType.properties, nodeParameters as INodeParameters, true, false, node);

					for (const key of Object.keys(nodeParameters as object)) {
						if (nodeParameters && nodeParameters[key] !== null && nodeParameters[key] !== undefined) {
							this.setValue(`parameters.${key}`, nodeParameters[key] as string);
						}
					}

					// Update the data in vuex
					const updateInformation = {
						name: node.name,
						value: nodeParameters,
					};

					this.$store.commit('setNodeParameters', updateInformation);

					this.$externalHooks().run('nodeSettings.valueChanged', { parameterPath, newValue, parameters: this.parameters, oldNodeParameters });

					this.updateNodeParameterIssues(node, nodeType);
					this.updateNodeCredentialIssues(node);
				} else {
					// A property on the node itself changed

					// Update data in settings
					Vue.set(this.nodeValues, parameterData.name, newValue);

					// Update data in vuex
					const updateInformation = {
						name: node.name,
						key: parameterData.name,
						value: newValue,
					};
					this.$store.commit('setNodeValue', updateInformation);
				}
			},
			/**
			 * Sets the values of the active node in the internal settings variables
			 */
			setNodeValues () {
				if (!this.node) {
					// No node selected
					return;
				}

				if (this.nodeType !== null) {
					this.nodeValid = true;

					const foundNodeSettings = [];
					if (this.node.color) {
						foundNodeSettings.push('color');
						Vue.set(this.nodeValues, 'color', this.node.color);
					}

					if (this.node.notes) {
						foundNodeSettings.push('notes');
						Vue.set(this.nodeValues, 'notes', this.node.notes);
					}

					if (this.node.alwaysOutputData) {
						foundNodeSettings.push('alwaysOutputData');
						Vue.set(this.nodeValues, 'alwaysOutputData', this.node.alwaysOutputData);
					}

					if (this.node.executeOnce) {
						foundNodeSettings.push('executeOnce');
						Vue.set(this.nodeValues, 'executeOnce', this.node.executeOnce);
					}

					if (this.node.continueOnFail) {
						foundNodeSettings.push('continueOnFail');
						Vue.set(this.nodeValues, 'continueOnFail', this.node.continueOnFail);
					}

					if (this.node.notesInFlow) {
						foundNodeSettings.push('notesInFlow');
						Vue.set(this.nodeValues, 'notesInFlow', this.node.notesInFlow);
					}

					if (this.node.retryOnFail) {
						foundNodeSettings.push('retryOnFail');
						Vue.set(this.nodeValues, 'retryOnFail', this.node.retryOnFail);
					}

					if (this.node.maxTries) {
						foundNodeSettings.push('maxTries');
						Vue.set(this.nodeValues, 'maxTries', this.node.maxTries);
					}

					if (this.node.waitBetweenTries) {
						foundNodeSettings.push('waitBetweenTries');
						Vue.set(this.nodeValues, 'waitBetweenTries', this.node.waitBetweenTries);
					}

					// Set default node settings
					for (const nodeSetting of this.nodeSettings) {
						if (!foundNodeSettings.includes(nodeSetting.name)) {
							// Set default value
							Vue.set(this.nodeValues, nodeSetting.name, nodeSetting.default);
						}
					}

					Vue.set(this.nodeValues, 'parameters', JSON.parse(JSON.stringify(this.node.parameters)));
				} else {
					this.nodeValid = false;
				}
			},
		},
		mounted () {
			this.setNodeValues();
			if (this.eventBus) {
				(this.eventBus as Vue).$on('openSettings', () => {
					this.openPanel = 'settings';
				});
			}
		},
	});
</script>

<style lang="scss" module>
.header {
	background-color: var(--color-background-base);
}
</style>

<style lang="scss">
.node-settings {
	overflow: hidden;
	min-width: 360px;
	max-width: 360px;
	background-color: var(--color-background-xlight);
	height: 100%;
	border: var(--border-base);
	border-radius: var(--border-radius-large);
	box-shadow: 0 4px 16px rgb(50 61 85 / 10%);

	.no-parameters {
		margin-top: var(--spacing-xs);
	}

	.header-side-menu {
		padding: var(--spacing-s) var(--spacing-s) var(--spacing-s) var(--spacing-s);
		font-size: var(--font-size-l);
		display: flex;

		.node-name {
			padding-top: var(--spacing-5xs);
			flex-grow: 1;
		}
	}

	.node-is-not-valid {
		padding: 10px;
	}

	.node-parameters-wrapper {
		height: 100%;
		overflow-y: auto;
		padding: 0 20px 200px 20px;
	}

	&.dragging {
		border-color: var(--color-primary);
		box-shadow: 0px 6px 16px rgba(255, 74, 51, 0.15);
	}
}

.parameter-content {
	font-size: 0.9em;
	margin-right: -15px;
	margin-left: -15px;
	input {
		width: calc(100% - 35px);
		padding: 5px;
	}
	select {
		width: calc(100% - 20px);
		padding: 5px;
	}

	&:before {
		display: table;
		content: " ";
		position: relative;
		box-sizing: border-box;
		clear: both;
	}
}

.parameter-wrapper {
	padding: 0 1em;
}

.color-reset-button-wrapper {
	position: relative;

}
.color-reset-button {
	position: absolute;
	right: 7px;
	top: -25px;
}

.parameter-value {
	input.expression {
		border-style: dashed;
		border-color: #ff9600;
		display: inline-block;
		position: relative;
		width: 100%;
		box-sizing:border-box;
		background-color: #793300;
	}
}

</style>
