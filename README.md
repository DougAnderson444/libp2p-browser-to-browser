# Libp2p Gossipsub Browser-to-Browser

Simple reproduction of the [browser-to-browser example](https://github.com/libp2p/js-libp2p/tree/master/packages/transport-webrtc/examples/browser-to-browser), but adding gossipsub protocol.

## Issue

Run the relay and connect the browsers in accordance with the example instructions. The browsers will connect to the relay and then to each other. When the relay drops, the `echo` still exchanges messages, but the `gossipsub` does not.

## Developing

The main app is in `src/routes/+page.svelte`.

Once you've created a project and installed dependencies with `npm install` (or `pnpm install` or `yarn`), start a development server:

```bash
npm run dev

# or start the server and open the app in a new browser tab
npm run dev -- --open
```

## Building

To create a production version of your app:

```bash
npm run build
```

You can preview the production build with `npm run preview`.

> To deploy your app, you may need to install an [adapter](https://kit.svelte.dev/docs/adapters) for your target environment.
