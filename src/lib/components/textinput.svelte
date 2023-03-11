<script lang="ts">
	import { v4 as uuidv4 } from 'uuid';
	export let large = false;
	export let regular = true;
	export let small = false;
	export let value: string = '';
	export let label: string = '';
	const id: string = uuidv4().replaceAll('-', '');
	let input_focus: boolean = false;
	let input_field: HTMLInputElement;

	export function focus() {
		input_field.focus();
	}
</script>

<label for={id} class="label" class:input_focus>
	{label}
</label>
<input
	bind:this={input_field}
	{id}
	class:large
	class:regular
	class:small
	class:input={true}
	type="text"
	on:focus={() => {
		input_focus = true;
	}}
	on:blur={() => {
		input_focus = false;
	}}
	bind:value
/>

<style lang="scss">
	.label {
		height: 1em;
		font-size: 0.8em;
		color: rgba(255, 255, 255, 0.8);
	}
	.input_focus {
		color: #ff6700;
	}
	.input {
		outline: none;
		color: #fff;
		border: 1px solid rgba(255, 255, 255, 0.3);
		font-family: 'Source Code Pro', monospace;
		border-radius: 5px;
		padding: 10px 10px;
		min-width: 200px;
		background-color: #222;
		font-size: 1.2em;
		&:focus {
			border: 1px solid #ff6700;
		}
	}
	.regular {
		min-width: 300px;
	}
	.large {
		min-width: 500px;
	}
</style>
