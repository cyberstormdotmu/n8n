<template>
	<ResizeObserver :breakpoints="[{ bp: 'md', width: 500 }]">
		<template #default="{ bp }">
			<div :class="bp === 'md' || columnView ? $style.grid : $style.gridMulti">
				<div
					v-for="(input, index) in filteredInputs"
					:key="input.name"
					:class="{ [`mt-${verticalSpacing}`]: index > 0 }"
				>
					<n8n-text
						color="text-base"
						v-if="input.properties.type === 'info'"
						tag="div"
						:size="input.properties.labelSize"
						:align="input.properties.labelAlignment"
						class="form-text"
					>
						{{ input.properties.label }}
					</n8n-text>
					<n8n-form-input
						v-else
						v-bind="input.properties"
						:name="input.name"
						:label="input.properties.label || ''"
						:value="values[input.name]"
						:data-test-id="input.name"
						:showValidationWarnings="showValidationWarnings"
						@input="(value) => onInput(input.name, value)"
						@validate="(value) => onValidate(input.name, value)"
						@change="(value) => onInput(input.name, value)"
						@enter="onSubmit"
					/>
				</div>
			</div>
		</template>
	</ResizeObserver>
</template>

<script lang="ts">
import Vue, { PropType } from 'vue';
import N8nFormInput from '../N8nFormInput';
import type { IFormInput } from '../../types';
import ResizeObserver from '../ResizeObserver';
import { EventBus } from '@/utils';

export default Vue.extend({
	name: 'n8n-form-inputs',
	components: {
		N8nFormInput,
		ResizeObserver,
	},
	props: {
		inputs: {
			type: Array,
			default() {
				return [[]];
			},
		},
		eventBus: {
			type: Object as PropType<EventBus>,
		},
		columnView: {
			type: Boolean,
			default: false,
		},
		verticalSpacing: {
			type: String,
			required: false,
			validator: (value: string): boolean => ['xs', 's', 'm', 'm', 'l', 'xl'].includes(value),
		},
	},
	data() {
		return {
			showValidationWarnings: false,
			values: {} as { [key: string]: unknown },
			validity: {} as { [key: string]: boolean },
		};
	},
	mounted() {
		(this.inputs as IFormInput[]).forEach((input) => {
			if (input.hasOwnProperty('initialValue')) {
				Vue.set(this.values, input.name, input.initialValue);
			}
		});

		if (this.eventBus) {
			this.eventBus.on('submit', this.onSubmit); // eslint-disable-line @typescript-eslint/unbound-method
		}
	},
	computed: {
		filteredInputs(): IFormInput[] {
			return (this.inputs as IFormInput[]).filter((input) =>
				typeof input.shouldDisplay === 'function' ? input.shouldDisplay(this.values) : true,
			);
		},
		isReadyToSubmit(): boolean {
			for (const key in this.validity) {
				if (!this.validity[key]) {
					return false;
				}
			}

			return true;
		},
	},
	methods: {
		onInput(name: string, value: unknown) {
			this.values = {
				...this.values,
				[name]: value, // eslint-disable-line @typescript-eslint/no-unsafe-assignment
			};
			this.$emit('input', { name, value }); // eslint-disable-line @typescript-eslint/no-unsafe-assignment
		},
		onValidate(name: string, valid: boolean) {
			Vue.set(this.validity, name, valid);
		},
		onSubmit() {
			this.showValidationWarnings = true;
			if (this.isReadyToSubmit) {
				const toSubmit = this.filteredInputs.reduce<{ [key: string]: unknown }>((accu, input) => {
					if (this.values[input.name]) {
						accu[input.name] = this.values[input.name];
					}
					return accu;
				}, {});
				this.$emit('submit', toSubmit);
			}
		},
	},
	watch: {
		isReadyToSubmit(ready: boolean) {
			this.$emit('ready', ready);
		},
	},
});
</script>

<style lang="scss" module>
.grid {
	display: grid;
	grid-row-gap: var(--spacing-s);
	grid-column-gap: var(--spacing-2xs);
}

.gridMulti {
	composes: grid;
	grid-template-columns: repeat(2, 1fr);
}
</style>
