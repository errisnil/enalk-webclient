<script lang="ts">
	import Button from '$lib/components/button.svelte';
	import Input from '$lib/components/textinput.svelte';
	import { onMount } from 'svelte';

	const wsurl = (handle: string = ''): string => {
		let url = '';
		if (location.hostname.indexOf('localhost') > -1) {
			url = 'ws://localhost:7896/ws';
		} else if (location.hostname.indexOf('127.0.0.1') > -1) {
			url = 'ws://127.0.0.1:7896/ws';
		} else {
			url = 'wss://noise.enalk.com/ws';
		}
		const qs: string[] = [];
		if (user_client_id) {
			qs.push('clid=' + user_client_id);
		}
		if (handle) {
			qs.push('handle=' + handle);
		}
		if (qs.length) {
			url += '?' + qs.join('&');
		}
		return url;
	};
	import Mic from 'svelte-material-icons/Microphone.svelte';
	import MicOff from 'svelte-material-icons/MicrophoneOff.svelte';
	import StopCircle from 'svelte-material-icons/StopCircle.svelte';

	interface PeerStream {
		stream: MediaStream;
		handle: string;
		client_id: string;
	}

	class AudioPeerConn {
		private pc: RTCPeerConnection | undefined;
		private ws: WebSocket | undefined = undefined;
		private making_offer: boolean = false;
		private ready: boolean = false;
		private streams: { [x: string]: PeerStream } = {}; // peer_streams
		private self_stream: MediaStream = new MediaStream();
		public onNewTrack?: () => void;
		public onConnectionFailed?: () => void;
		private _conn_failed: boolean = false;
		private track_stream_map: { [x: string]: string } = {};
		private is_speaker: boolean = false;

		constructor(is_speaker: boolean = false) {
			this.is_speaker = is_speaker;
			this.pc = new RTCPeerConnection({
				iceServers: [
					{
						urls: 'stun:stun.l.google.com:19302'
					}
				]
			});

			this.pc.ontrack = async (ev: RTCTrackEvent) => {
				const client = ev.streams[0].id.split('-');
				const client_id = client[0];
				const client_handle = client.slice(1).join('-');
				if (client_id) {
					this.track_stream_map[ev.track.id] = client_id;
				}
				if (!(client_id in this.streams)) {
					this.streams[client_id] = {
						stream: ev.streams[0],
						client_id: client_id,
						handle: client_handle
					};
				} else {
					this.streams[client_id].stream.addTrack(ev.track);
				}

				// event callback
				if (this.onNewTrack) {
					this.onNewTrack();
				}
			};

			this.pc.onnegotiationneeded = async () => {
				if (this.making_offer) {
					return;
				}
				await this.make_offer();
			};

			this.pc.onicecandidate = (ev: RTCPeerConnectionIceEvent) => {
				if (typeof this.ws === 'undefined' || !ev || ev.candidate === null) return;
				this.ws.send(`candidate\n${btoa(JSON.stringify(ev.candidate.toJSON()))}`);
			};

			this.pc.oniceconnectionstatechange = () => {
				if (!this.pc) {
					return;
				}
				if (this.pc.iceConnectionState === 'failed') {
					this.pc.close();
					this._conn_failed = true;
					if (this.onConnectionFailed) {
						this.onConnectionFailed();
					}
				} else if (this.pc.iceConnectionState === 'connected') {
					if (states.pconn_connecting) {
						states.pconn_connecting = false;
					}
					if (states.pconn_interrupted) {
						states.pconn_interrupted = false;
					}
				}
			};
			this.pc.onsignalingstatechange = async () => {
				if (!this.pc) return;
				if (this.pc.signalingState === 'stable') {
					this.making_offer = false;
					await this.update_all_streams();
				} else {
					this.making_offer = true;
				}
			};
		}

		private async update_all_streams() {
			if (!this.pc) return;
			// console.log('[update_all_streams] -> fired');
			// const ts: { [x: string]: PeerStream } = {};
			console.log(this.streams);
			try {
				if (this.is_speaker) {
					for (const trecv of this.pc.getTransceivers()) {
						const track_id = trecv.receiver.track.id;
						if (!track_id) continue;
						const client_id = this.track_stream_map[track_id];
						if (!client_id) continue;
						if (!(client_id in this.streams)) continue;
						if (trecv.receiver.track.muted) {
							const stream = this.streams[client_id].stream;
							if (stream) {
								const track = stream.getTrackById(track_id);
								if (track !== null) {
									stream.removeTrack(track);
								}
								delete this.streams[client_id];
							}

							// continue;
						}
						// console.log(`[update_all_streams] -> track_id: "${track_id}"`);
						// if (client_id) {
						// 	if (!(client_id in ts)) {
						// 		ts[client_id] = this.streams[client_id];
						// 	}
						// 	const t = ts[client_id].stream.getTrackById(track_id);
						// 	if (t === null) {
						// 		ts[client_id].stream.addTrack(trecv.receiver.track);
						// 	}
						// }
					}
				} else {
					for (let recv of this.pc.getReceivers()) {
						const track_id = recv.track.id;
						if (!track_id) continue;
						const client_id = this.track_stream_map[track_id];
						if (!client_id) continue;
						if (!(client_id in this.streams)) continue;
						if (recv.track.muted) {
							const stream = this.streams[client_id].stream;
							if (stream) {
								const track = this.streams[client_id].stream.getTrackById(track_id);
								if (track !== null) {
									stream.removeTrack(track);
								}
								delete this.streams[client_id];
							}
							// continue;
						}
						// console.log(`[update_all_streams] -> track_id: "${track_id}"`);
						// if (client_id) {
						// 	if (!(client_id in ts)) {
						// 		ts[client_id] = this.streams[client_id];
						// 	}
						// 	const t = ts[client_id].stream.getTrackById(track_id);
						// 	if (t === null) {
						// 		ts[client_id].stream.addTrack(recv.track);
						// 	}
						// }
					}
				}

				this.streams = this.streams;
				if (this.onNewTrack) {
					this.onNewTrack();
				}
			} catch (e) {
				console.error('Updating tracks failed: ', e);
			}
		}

		private async make_offer() {
			if (typeof this.pc === 'undefined' || typeof this.ws === 'undefined') {
				return;
			}
			this.making_offer = true;

			try {
				const offer = await this.pc.createOffer();
				await this.pc.setLocalDescription(offer);
				this.ws.send(`description\n${btoa(JSON.stringify(offer))}`);
			} catch (e) {
				console.error('[new_offer] -> ', e);
			} finally {
				this.making_offer = true;
			}
		}

		public set_ws(ws: WebSocket) {
			this.ws = ws;
		}

		public async init(ws: WebSocket) {
			if (typeof this.pc === 'undefined') {
				return;
			}
			if (this.ready) {
				console.error('peer connection is already ready');
				return;
			}
			this.ws = ws;
			let m: MediaStream;
			try {
				m = await navigator.mediaDevices.getUserMedia({
					audio: {
						autoGainControl: true,
						channelCount: 1,
						echoCancellation: true,
						noiseSuppression: true
					},
					video: false
				});
			} catch (e) {
				console.error('getUserMedia failed: ', e);
				return;
			}

			for (const track of m.getTracks()) {
				this.pc.addTrack(track);
				this.self_stream.addTrack(track);
				this.self_stream = this.self_stream;
			}
			await this.make_offer();
			this.ready = true;
		}

		public async new_description(arg: string) {
			// console.log('[new_description] -> Fired');
			if (typeof this.pc === 'undefined' || typeof this.ws === 'undefined') {
				return;
			}

			let sdp: RTCSessionDescription = JSON.parse(atob(arg));
			try {
				if (sdp.type == 'offer' && (this.making_offer || this.pc.signalingState !== 'stable')) {
					// console.log('[new_description] -> rolling back');
					await Promise.all([
						this.pc.setLocalDescription({ type: 'rollback' }),
						this.pc.setRemoteDescription(sdp)
					]);
				} else {
					await this.pc.setRemoteDescription(sdp);
				}
			} catch (e) {
				console.log('[new_description] -> setting remote sdp failed: ', e);
			}
			try {
				if (sdp.type == 'offer') {
					// console.log('setting local answer');
					const answer = await this.pc.createAnswer();
					await this.pc.setLocalDescription(answer);
					this.ws.send(`description\n${btoa(JSON.stringify(answer))}`);
				}
			} catch (e) {
				console.error('[new_description] -> setting local sdp failed: ', e);
			}
		}

		public async new_candidate(arg: string) {
			let candidate: RTCIceCandidateInit = JSON.parse(atob(arg));
			// console.log('[new_candidate] -> ', candidate);
			if (this.pc) {
				this.pc.addIceCandidate(candidate);
			}
		}

		public close() {
			if (this.pc) {
				this.pc.close();
				this._conn_failed = false;
			}
			for (const track of this.self_stream.getTracks()) {
				track.stop();
				this.self_stream.removeTrack(track);
			}
			this.streams = {};
			this.ws = undefined;
			this.ready = false;
			this.self_stream = new MediaStream();
			this.making_offer = false;
			this.pc = undefined;
			this.onConnectionFailed = undefined;
			this.onNewTrack = undefined;
		}

		public has_connection_failed(): boolean {
			return this._conn_failed;
		}

		public is_connection_closed(): boolean {
			if (!this.pc) {
				return true;
			}
			if (this.pc.connectionState == 'closed') {
				return true;
			}
			return false;
		}

		public is_ready(): boolean {
			return this.ready;
		}

		public peer_streams(): { [x: string]: PeerStream } {
			return this.streams;
		}

		public mute_self() {
			for (const track of this.self_stream.getTracks()) {
				track.enabled = false;
			}
		}
		public unmute_self() {
			for (const track of this.self_stream.getTracks()) {
				track.enabled = true;
			}
		}
	}

	let apc: AudioPeerConn | undefined = undefined;
	let ws: WebSocket | undefined = undefined;

	const fresh_peer_conn = async (is_speaker: boolean = false) => {
		if (apc) {
			apc = undefined;
			// hopefully gc does something here
			await new Promise((r) => setTimeout(r, 1000));
		}
		if (!states.show_noise_view) {
			return;
		}

		apc = new AudioPeerConn(is_speaker);
		apc.onNewTrack = () => {
			if (apc) {
				console.log('[onNewTrack] -> reinit streams');
				streams = apc.peer_streams();
			}
		};
		apc.onConnectionFailed = () => {
			if (apc) {
				apc.close();
				// UI State
				states['pconn_interrupted'] = true;
				fresh_peer_conn();
			}
		};
		if (ws) {
			await apc.init(ws);
		}
	};

	const engage_peer_conn = async (ws: WebSocket, is_speaker: boolean = false) => {
		if (!apc || apc.has_connection_failed() || apc.is_connection_closed()) {
			await fresh_peer_conn(is_speaker);
		}
		if (apc) {
			if (apc.is_ready()) {
				apc.set_ws(ws);
			} else {
				apc.init(ws);
			}
		}
	};

	const handle_client_id = (data: string) => {
		user_client_id = data;
	};

	const new_ws_conn = async (is_speaker: boolean = false) => {
		ws = new WebSocket(wsurl(user_handle));
		ws.onopen = async () => {
			// console.log('WS Connection established. Try RTCPeerConn');
			try {
				await engage_peer_conn(ws as WebSocket, is_speaker);
			} catch (e) {
				console.error('establishing new peer conn failed: ', e);
				return;
			}
		};

		ws.onclose = async () => {
			if (!apc || !states.show_noise_view) {
				return;
			}
			new_ws_conn();
		};

		ws.onmessage = async (ev: MessageEvent) => {
			if (!apc) {
				console.error('[onmessage] -> apc not ready yet');
				return;
			}
			let args = ev.data.split('\n');
			if (args.length != 2) {
				console.error('invalid message received from ws');
			}
			switch (args[0]) {
				case 'description':
					apc.new_description(args[1]);
					break;
				case 'candidate':
					apc.new_candidate(args[1]);
					break;
				case 'speaking':
					update_speaking(args[1]);
					break;
				case 'muted':
					update_mute_state_local(args[1], true);
					break;
				case 'unmuted':
					update_mute_state_local(args[1], false);
					break;
				case 'clid':
					handle_client_id(args[1]);
					break;
				default:
					console.error(`[onmessage] -> we don't handle "${args[0]}" yet`);
			}
		};

		ws.onerror = async (e) => {
			console.error('websocket error: ', e);
		};
	};

	const states: { [x: string]: boolean } = {
		show_noise_view: false,
		self_muted: false,
		pconn_interrupted: false,
		pconn_connecting: false
	};

	const join_as_yapper = () => {
		new_ws_conn(true);
		states.show_noise_view = true;
	};

	const close_noise = () => {
		states.show_noise_view = false;
		if (apc) apc.close();
		apc = undefined;
		if (ws) {
			ws.close();
			ws = undefined;
		}
	};

	const srcObject = (elem: HTMLAudioElement, stream: MediaStream) => {
		elem.srcObject = stream;
		return {
			update(stream: MediaStream) {
				if (elem.srcObject != stream) {
					elem.srcObject = stream;
				}
			}
		};
	};

	let streams: { [x: string]: PeerStream } = {};
	let speaking_peers: { [x: string]: boolean } = {};
	let muted_peers: { [x: string]: boolean } = {};
	let user_handle: string = '';
	let user_handle_field: any;
	let user_client_id: string;
	const valid_user_handle = (handle: string): boolean => {
		// console.log('[valid_user_handle] -> Fired');
		let re = new RegExp('^[a-zA-Z][a-zA-Z0-9]{2,}$');
		return re.test(handle);
	};

	const update_speaking = (data: string) => {
		const speakers = data.split(',');
		const smap: { [x: string]: boolean } = {};
		for (const speaker of speakers) {
			smap[speaker] = true;
		}
		speaking_peers = smap;
	};

	const update_mute_state_local = (data: string, muted: boolean) => {
		// console.log(`peer: "${data}" is muted`);
		if (muted) {
			muted_peers[data] = true;
		} else {
			delete muted_peers[data];
		}
		muted_peers = muted_peers;
	};

	const send_mute_state = () => {
		let msg = 'unmuted\n';
		if (states.self_muted) {
			msg = 'muted\n';
		}
		if (ws) {
			ws.send(msg);
		}
	};

	const toggle_mute = () => {
		states.self_muted = !states.self_muted;
		if (apc) {
			if (states.self_muted) {
				apc.mute_self();
			} else {
				apc.unmute_self();
			}
		}
		send_mute_state();
	};

	onMount(() => {
		user_handle_field.focus();
	});
</script>

<div id="yap-cont">
	{#if states.show_noise_view}
		<div id="noise-view">
			<div id="noise-view-tracks">
				{#each Object.keys(streams) as speaker}
					<div class="speaker">
						<span>{streams[speaker].handle}</span>
						{#if speaker in muted_peers}
							<span> is muted</span>
						{/if}
						{#if speaker in speaking_peers && !(speaker in muted_peers)}
							<span> is speaking</span>
						{/if}
						<audio use:srcObject={streams[speaker].stream} autoplay />
					</div>
				{/each}
				<div id="noise-controls">
					<div id="mute-toggle">
						{#if states.self_muted}
							<span on:click={toggle_mute}
								><MicOff viewBox="0 0 24 24" size="4em" width="4em" /></span
							>
						{:else}
							<span on:click={toggle_mute}><Mic viewBox="0 0 24 24" size="4em" width="4em" /></span>
						{/if}
					</div>
					<div id="stop-noise">
						<span on:click={close_noise}
							><StopCircle viewBox="0 0 24 24" size="4em" width="4em" /></span
						>
					</div>
				</div>
			</div>
		</div>
	{:else}
		<div class="handle-cont">
			<Input bind:value={user_handle} label="Enter your handle" bind:this={user_handle_field} />
			<Button
				label="join as speaker"
				disabled={!valid_user_handle(user_handle)}
				on:click={() => {
					if (valid_user_handle(user_handle)) join_as_yapper();
				}}
			/>
		</div>
	{/if}
</div>

<style lang="scss">
	#yap-cont {
		width: 100%;
		min-height: 100%;
		display: flex;
		align-items: center;
		justify-content: center;
		.handle-cont {
			display: flex;
			flex-direction: column;
			row-gap: 20px;
		}

		#noise-view {
			display: flex;
			#noise-view-tracks {
				width: 720px;
				height: 405px;
				background-color: #000;
				border: 1px solid #ff6700;
				color: #ff6700;
				position: relative;

				#noise-controls {
					position: absolute;
					bottom: 0;
					width: 100%;
					display: flex;
					justify-content: center;
					padding: 10px 0;

					#mute-toggle {
						width: min-content;
						display: flex;
						align-items: center;
						justify-content: center;
						padding: 10px;
						border: 1px solid;
						border-radius: 100px;
						span {
							cursor: pointer;
						}
					}
					#stop-noise {
						width: min-content;
						display: flex;
						align-items: center;
						justify-content: center;
						padding: 10px;
						border: 1px solid;
						border-radius: 100px;
						span {
							cursor: pointer;
						}
					}
				}
			}
		}
	}
</style>
