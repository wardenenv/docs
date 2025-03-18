# Warden Laravel Environment

## Laravel + Vite

Starting with Laravel 9.x Vite was added as an asset bundler. Starting with Warden
0.15.1 you can run the vite development server within the Warden container.

To fully support running the vite development server within the Warden container
you need to adjust your `vite.config.js` file:

```javascript
export default defineConfig({
    server: {
        host: true, // Tell's Vite to listen on all IP addresses; could also use '0.0.0.0'
        port: 5173,
        strictPort: true, // Don't let Vite choose a different port
        origin: `https://vite.<Warden Env Name>.test`, // Replace <Warden Env Name> with your Warden environment name
        allowedHosts: ['.<Warden Env Name>.test'], // Replace <Warden Env Name> with your Warden environment name
        cors: {
            origin: /https?:\/\/([A-Za-z0-9\-\.]+)?(.+\.test)(?::\d+)?$/, // Allow any `.test` domain
        }
    },
    // ... The rest of your existing configuration ...
});
```
