# Stacy

A minimal, phone-friendly Gemini chat app built with Vue 3 + Vite. Stacy has two main sections: Chat and API key.

## Features

- Familiar conversation UI with a searchable chat list, new chats, local history, delete controls and clear-all action.
- Separate API Key tab with show/hide, connection check, and dynamic discovery of models that support Google's generateContent method.
- Multi-turn conversations, model selector, temperature control, stop generation, regenerate, copy response, Markdown rendering and request error feedback.
- Responsive mobile layout with a slide-out conversation list.
- API key stays in component memory only and is not saved to local storage.
- Conversations are stored in the browser's local storage and are never sent to this app's hosting server.

## Run locally

Requires a current Node.js LTS release.

    npm install
    npm run dev

Create a production build with:

    npm run build
    npm run preview

## Gemini setup

1. Create or manage a key at https://aistudio.google.com/apikey.
2. Open Stacy and choose **API key**.
3. Paste the key and press **Connect key**. Stacy asks Google's Models API for the models available to that key.
4. Return to **Chat**, choose an available model and start chatting.

Model availability, usage limits and billing depend on the Google account/project and may change over time.

## Security notes

This is a browser-only personal tool. The API key is sent directly to generativelanguage.googleapis.com and is not stored by Stacy or injected at build time. A key used in a browser can still be inspected by a person with access to that browser session. Restrict the key in Google settings where possible, monitor its quota and do not use a key with broad permissions on a public/shared device.

The local conversation store is not encrypted. Do not store sensitive conversations on a shared device.

## Deployment

This is a static Vite app. Vercel should use the Vue/Vite framework preset, with npm run build as the build command and dist as the output directory. No server environment variable is required because users enter their own key in the browser.

See PLAN_CHECKLIST.md for the original implementation checklist.