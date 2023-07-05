<script>
	import { onMount } from 'svelte';

	import { multiaddr, protocols } from '@multiformats/multiaddr';
	import { pipe } from 'it-pipe';
	import { fromString, toString } from 'uint8arrays';
	import { webRTC, webRTCDirect } from '@libp2p/webrtc';
	import { webSockets } from '@libp2p/websockets';
	import * as filters from '@libp2p/websockets/filters';
	import { pushable } from 'it-pushable';
	import { mplex } from '@libp2p/mplex';
	import { createLibp2p } from 'libp2p';
	import { circuitRelayTransport } from 'libp2p/circuit-relay';
	import { noise } from '@chainsafe/libp2p-noise';
	import { identifyService } from 'libp2p/identify';
	import { gossipsub } from '@chainsafe/libp2p-gossipsub';
	import { sha256 } from 'multiformats/hashes/sha2';

	const CHAT_TOPIC = 'universal-connectivity';

	onMount(async () => {
		const WEBRTC_CODE = protocols('webrtc').code;

		const output = document.getElementById('output');
		const sendSection = document.getElementById('send-section');
		const appendOutput = (line) => {
			const div = document.createElement('div');
			div.appendChild(document.createTextNode(line));
			output.append(div);
		};
		const clean = (line) => line.replaceAll('\n', '');
		const sender = pushable();

		const node = await createLibp2p({
			addresses: {
				listen: ['/webrtc']
			},
			transports: [
				webSockets({
					filter: filters.all
				}),
				webRTC(),
				webRTCDirect(),
				circuitRelayTransport({
					discoverRelays: 1
				})
			],
			connectionEncryption: [noise()],
			streamMuxers: [mplex()],
			connectionGater: {
				denyDialMultiaddr: () => {
					// by default we refuse to dial local addresses from the browser since they
					// are usually sent by remote peers broadcasting undialable multiaddrs but
					// here we are explicitly connecting to a local node so do not deny dialing
					// any discovered address
					return false;
				}
			},
			services: {
				identify: identifyService(),
				pubsub: gossipsub({
					allowPublishToZeroPeers: true,
					msgIdFn: msgIdFnStrictNoSign,
					ignoreDuplicatePublishError: true,
					emitSelf: true
				})
			}
		});

		await node.start();

		// handle the echo protocol
		await node.handle('/echo/1.0.0', ({ stream }) => {
			pipe(
				stream,
				async function* (source) {
					for await (const buf of source) {
						const incoming = toString(buf.subarray());
						appendOutput(`Received message '${clean(incoming)}'`);
						yield buf;
					}
				},
				stream
			);
		});

		function updateConnList() {
			// Update connections list
			const connListEls = node.getConnections().map((connection) => {
				if (connection.remoteAddr.protoCodes().includes(WEBRTC_CODE)) {
					sendSection.style.display = 'block';
				}

				const el = document.createElement('li');
				el.textContent = connection.remoteAddr.toString();
				return el;
			});
			document.getElementById('connections').replaceChildren(...connListEls);
		}

		node.addEventListener('connection:open', (event) => {
			updateConnList();
		});
		node.addEventListener('connection:close', (event) => {
			updateConnList();
		});

		node.addEventListener('self:peer:update', (event) => {
			// Update multiaddrs list
			const multiaddrs = node.getMultiaddrs().map((ma) => {
				const el = document.createElement('li');
				el.textContent = ma.toString();
				return el;
			});
			document.getElementById('multiaddrs').replaceChildren(...multiaddrs);
		});

		const isWebrtc = (ma) => {
			return ma.protoCodes().includes(WEBRTC_CODE);
		};

		window.connect.onclick = async () => {
			const ma = multiaddr(window.peer.value);
			appendOutput(`Dialing '${ma}'`);
			const connection = await node.dial(ma);

			if (isWebrtc(ma)) {
				const outgoing_stream = await connection.newStream(['/echo/1.0.0', '/meshsub/1.1.0']);
				pipe(sender, outgoing_stream, async (src) => {
					for await (const buf of src) {
						const response = toString(buf.subarray());
						appendOutput(`Received message '${clean(response)}'`);
					}
				});
			}

			appendOutput(`Connected to '${ma}'`);
		};

		window.send.onclick = async () => {
			const message = `${window.message.value}\n`;
			appendOutput(`Sending message '${clean(message)}'`);
			sender.push(fromString(message));

			// also publish to pubsub
			try {
				const res = await node.services.pubsub.publish(
					CHAT_TOPIC,
					new TextEncoder().encode(message)
				);

				console.log(
					'sent pubsub message to: ',
					res.recipients.map((peerId) => peerId.toString())
				);
			} catch (e) {
				console.error(e);
				throw e;
			}
		};

		node.services.pubsub.subscribe(CHAT_TOPIC);

		node.services.pubsub.addEventListener('message', (evt) => {
			console.log('pubsub message received: ', evt);
			const { topic, from, data } = evt.detail;
			const msg = new TextDecoder().decode(data);
			console.log(`${topic}: ${msg}`);
			appendOutput(`Pubsub Received message ${topic}: '${clean(msg)}'`);
		});
	});

	async function msgIdFnStrictNoSign(msg) {
		var enc = new TextEncoder();

		const signedMessage = msg;
		const encodedSeqNum = enc.encode(signedMessage.sequenceNumber.toString());
		return await sha256.encode(encodedSeqNum);
	}
</script>

<div>
	<div>
		<label for="peer">Remote MultiAddress:</label>
		<input type="text" id="peer" />
		<button id="connect">Connect</button>
	</div>
	<div id="send-section">
		<label for="message">Message:</label>
		<input type="text" id="message" value="hello" />
		<button id="send">Send</button>
	</div>
	<div id="connectionsWrapper">
		<h3>Active Connections:</h3>
		<ul id="connections" />
	</div>
	<div id="listeningAddressesWrapper">
		<h3>Listening addresses:</h3>
		<ul id="multiaddrs" />
	</div>
	<div id="output" />
</div>

<style>
	:root {
		font-family: sans-serif;
	}
	label,
	button {
		display: block;
		font-weight: bold;
		margin: 5px 0;
	}
	div {
		margin-bottom: 20px;
	}
	#send-section {
		display: none;
	}
	input[type='text'] {
		width: 800px;
	}
</style>
