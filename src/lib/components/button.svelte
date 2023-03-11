<script lang="ts">
	export let label: string = 'button';
	export let disabled = false;
</script>

<button class:disabled on:click>{label}</button>

<style lang="scss">
	button {
		font-family: 'Source Code Pro', monospace;
		border-radius: 5px;
		outline: none;
		border: none;
		background-color: rgba(255, 103, 0, 0.3);
		color: rgb(255, 103, 0);
		padding: 10px;
		min-width: 200px;
		height: min-content;
		text-transform: uppercase;
		transition: color 0.1s linear 0s;
		transition: background-color 0.1s linear 0s;
		&:hover {
			background-color: rgb(255, 103, 0);
			color: #222;
			cursor: pointer;
		}
	}
	.disabled {
		background-color: rgba(255, 255, 255, 0.2);
		color: #fff;
		cursor: default;
		&:hover {
			cursor: default;
			background-color: rgba(255, 255, 255, 0.2);
			color: #fff;
		}
	}
</style>
