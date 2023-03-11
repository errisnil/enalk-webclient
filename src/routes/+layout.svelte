<nav>
	<div id="logo">
		<img src="/logo.svg" alt="Enalk Logo" />
		<span>nalk</span>
	</div>
</nav>

<slot />

<style lang="scss">
	nav {
		height: max-content;
		width: 100%;
		padding: 10px;
		display: flex;
		border-bottom: 1px solid #ff6700;
		#logo {
			font-family: 'Source Code Pro', sans-serif;
			img {
				width: 40px;
			}
			display: flex;
			align-items: center;
			span {
				font-size: 20px;
				color: #ff6700;
				margin-left: 5px;
			}
		}
	}
</style>
