# Blackfire Profiling

For information on what Blackfire is, please see the [introduction to Blackfire](https://blackfire.io/docs/introduction) in Blackfire documentation.

Blackfire may be enabled on both `magento1` and `magento2` env types by adding the following to the project's `.env` file (or exporting them to environment variables prior to starting the environment):

```
WARDEN_BLACKFIRE=1

BLACKFIRE_CLIENT_ID=<client_id>
BLACKFIRE_CLIENT_TOKEN=<client_token>
BLACKFIRE_SERVER_ID=<server_id>
BLACKFIRE_SERVER_TOKEN=<server_token>
```

Note: You can obtain the IDs and Tokens used in the above from within your Blackfire account under Account Settings -> Credentials or from the credentials are of the environment you're pushing profile information into.

## CLI Tool

To use the Blackfire CLI Tool, you can run `warden blackfire [arguments]`.

For more information on the CLI tool, please see [Profiling CLI Commands](https://blackfire.io/docs/cookbooks/profiling-cli) in Blackfire's documentation.

## Browser Profiling

Blackfire provides browser extensions that let you profile any page directly from the toolbar. Install the extension for your browser, log in to your Blackfire account, navigate to the page you want to profile, and click the **Profile** button in the extension.

For a full walkthrough, see [Profiling HTTP Requests with a Browser](https://docs.blackfire.io/profiling-cookbooks/profiling-http-via-browser) in Blackfire's documentation.

### Browser extensions

  * [Firefox extension](https://docs.blackfire.io/integrations/browsers/firefox) — **recommended**, supports all profiling features including **Profile all requests** (POST, Ajax, API calls)
  * [Chrome extension](https://docs.blackfire.io/integrations/browsers/chrome) — single-page profiling works, but **Profile all requests** is unavailable (see below)

### Chrome limitation

Chrome's Manifest V3 restricts extensions from modifying outgoing request headers, which breaks the **Profile all requests** feature. This means Chrome can only profile the initial page load — it cannot capture follow-up POST requests, Ajax calls, or API requests within a session.

Single-page profiling (clicking **Profile** on a specific URL) still works normally in Chrome.

If your workflow relies on profiling all requests in a session, **use Firefox** where this feature remains fully supported. For more details, see [Announcing changes to the "Profile all requests" feature on Chrome](https://blog.blackfire.io/announcing-changes-to-the-profile-all-requests-feature-on-chrome.html) on the Blackfire blog.
