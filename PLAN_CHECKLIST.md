# Stacy — implementation checklist

The implementation follows this checklist in order, before adding application code.

## Plan

- [x] Create a new public GitHub repository named `stacy-chat` with a default `main` branch.
- [x] Record the build and deployment checklist before writing application files.
- [ ] Build the app with Vue 3 + Vite (not a single HTML file).
- [ ] Create two clear top-level tabs: **Chat** and **API Key**.
- [ ] Keep the chat screen familiar: conversation list, new chat, message composer, Enter-to-send, Shift+Enter newline, stop, regenerate, copy, and clear/delete conversations.
- [ ] Use a restrained black / off-white / light-grey palette; avoid gradient-heavy AI-SaaS templates and over-designed editorial layouts.
- [ ] Use the Gemini API key only in browser memory; never embed it in source files or persist it to chat history/local storage.
- [ ] On the API Key tab, support key entry, show/hide, connection test, and dynamic discovery of models that support `generateContent`.
- [ ] Support multi-turn Gemini conversations, model selection, temperature, readable error feedback, and safe Markdown rendering.
- [ ] Persist conversation content locally in the browser, without storing the API key.
- [ ] Make layouts usable on phones and desktop, including keyboard/accessibility states.
- [ ] Add project documentation and Vercel/Vite configuration.
- [ ] Validate file structure/build configuration and commit the implementation through Composio.
- [ ] Create/link the Vercel project to the GitHub repository and trigger a deployment.
- [ ] Verify deployment state and report repository/deployment URLs plus any limitations.

## Safety and implementation notes

- Browser-side API keys are visible to someone who can inspect the browser session. Use this only as a personal tool; restrict the key in Google AI Studio/Cloud where supported and monitor quota.
- The app is a client-side frontend; no API key is included as a Vercel environment variable because requests originate from the user's browser.
- Google API model availability varies by key/project and changes over time, so populate the list from the Models endpoint instead of hardcoding questionable model IDs.
