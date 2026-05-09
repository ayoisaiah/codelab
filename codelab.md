summary: Build a Chrome extension that uses the Gemini API to instantly create Google Calendar events from any text you select on a web page. 
id: building-extensions-with-ai 
categories: Web, AI 
environments: 
Web 
status: Draft
feedback link: https://github.com/ayoisaiah/quick-event-extension/issues
authors: Ayooluwa Isaiah 
analytics account:

# Building Browser Extensions with AI

## Introduction

Duration: 2:00

You're browsing the web and you spot an event you want to attend. The date,
time, location, and description are all right there on the page. But to get it
into your calendar, you have to:

1. Copy the title
2. Open Google Calendar
3. Create a new event
4. Paste the title
5. Go back to the page, find the date, copy it
6. Go back to Calendar, set the date
7. Repeat for the time, location, description...

This is tedious, error-prone, and something you probably do every week. What if
your browser could just do it for you?

In this codelab, you'll build a Chrome extension called **Quick Event** that
lets you highlight any event description on a web page, right-click, and
instantly open Google Calendar with all the event details pre-filled, powered by
the Gemini API.

Along the way, you'll learn how to:

- Create a Chrome extension from scratch using Manifest V3
- Capture user-selected text using context menus
- Call the Gemini API from an extension to extract structured data
- Craft effective prompts for reliable JSON extraction
- Build an options page that lets users provide their own API key
- Construct Google Calendar URLs programmatically

### What you'll need

- Google Chrome (version 138 or later)
- A text editor or IDE
- Basic knowledge of HTML, CSS, and JavaScript
- A Gemini API key (free, we'll walk through getting one)

## Getting set up

Duration: 5:00

### Get a Gemini API key

The Gemini API offers a generous free tier that's perfect for this project.
You'll need an API key to use it.

1. Go to [Google AI Studio](https://aistudio.google.com)
2. Sign in with your Google account
3. Click **"Get API key"** in the left navigation
4. Click **"Create API key"** and select a project (or create a new one)
5. Copy your API key and save it somewhere safe

**Note**: Treat your API key like a password. Never commit it to a public
repository, share it online, or hardcode it in published extension code. In this
codelab, users will enter their own key through a secure options page.

### Verify your API key works

To test your key, run this in your terminal (replace `YOUR_API_KEY` with your
actual key):

```console
curl "https://generativelanguage.googleapis.com/v1beta/models/gemini-3.1-flash-lite:generateContent" \
  -H "x-goog-api-key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -X POST \
  -d '{
    "contents": [{
      "parts": [{"text": "Say hello in three languages"}]
    }]
  }'
```

You should see a JSON response with the model's output. If you get a 403 or 401
error, double-check that you copied the full key.

### Get the code

We've set up a starter repository with the project scaffolding. Clone it and
open it in your editor:

```console
git clone https://github.com/ayoisaiah/quick-event-extension.git
cd quick-event-extension
```

The starter repo contains the basic file structure with placeholder comments
where you'll add code throughout this codelab.

### Load the extension in Chrome

1. Open Chrome and navigate to `chrome://extensions`
2. Enable **Developer mode** using the toggle in the top right corner
3. Click **"Load unpacked"**
4. Select the `quick-event-extension` folder
5. You should see "Quick Event" appear in your extensions list

You'll reload the extension each time you make changes by clicking the refresh
icon on the extension card.

## Understanding the extension anatomy

Duration: 3:00

Open the project folder. Here's what each file does:

```text
quick-event-extension/
├── manifest.json        # The extension's ID card
├── background.js        # The brain — runs in the background
├── options.html         # Settings page for the API key
├── options.js           # Logic for the settings page
└── icons/
    ├── icon16.png
    ├── icon32.png
    ├── icon48.png
    └── icon128.png
```

### manifest.json

Every Chrome extension starts with a `manifest.json` file. It tells Chrome what
permissions the extension needs, what scripts to run, and what the extension
looks like.

Create or update your `manifest.json` with the following:

```json
{
  "manifest_version": 3,
  "name": "Quick Event",
  "version": "1.0",
  "description": "Highlight event text on any web page and create a Google Calendar event instantly with AI.",
  "permissions": ["contextMenus", "storage", "tabs"],
  "background": {
    "service_worker": "background.js"
  },
  "action": {
    "default_icon": {
      "16": "icons/icon16.png",
      "48": "icons/icon48.png",
      "128": "icons/icon128.png"
    },
    "default_title": "Quick Event"
  },
  "options_ui": {
    "page": "options.html",
    "open_in_tab": false
  },
  "icons": {
    "16": "icons/icon16.png",
    "32": "icons/icon32.png",
    "48": "icons/icon48.png",
    "128": "icons/icon128.png"
  }
}
```

Here's what matters:

- `manifest_version: 3` is the latest format, required for new extensions.
- `permissions` lists only what we need: `contextMenus` to add a right-click
  menu item, and `storage` to save the user's API key securely.
- `background.service_worker` points to our background script, which runs as a
  [service worker](https://developer.chrome.com/docs/extensions/develop/concepts/service-workers).
- `options_ui` gives our extension a settings page where users can enter their
  API key.

Positive: Request only the permissions you need. Fewer permissions means more
trust from users, which means more installs. This extension doesn't need access
to any web page content or browsing history.

## Building the options page

Duration: 5:00

Before we can call the Gemini API, users need a way to provide their API key.
We'll build a simple options page for this.

### Create the options HTML

```html
<!-- options.html -->
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8" />
    <title>Quick Event Settings</title>
    <style>
      body {
        font-family: system-ui, sans-serif;
        padding: 20px;
        max-width: 480px;
        color: #333;
      }

      h1 {
        font-size: 18px;
        margin-bottom: 4px;
      }

      p {
        font-size: 13px;
        color: #666;
        margin-top: 0;
      }

      label {
        display: block;
        font-size: 13px
        font-weight: 600;
        margin-bottom: 6px;
      }

      input[type='text'] {
        width: 100%;
        padding: 8px 10px;
        border: 1px solid #ccc;
        border-radius: 6px;
        font-size: 14px;
        font-family: monospace;
        box-sizing: border-box;
      }

      button {
        margin-top: 12px;
        padding: 8px 20px;
        background: #1a73e8;
        color: white;
        border: none;
        border-radius: 6px;
        font-size: 13px;
        cursor: pointer;
      }

      button:hover {
        background: #1557b0;
      }

      .status {
        margin-top: 10px;
        font-size: 13px;
        color: #188038;
      }

      a {
        color: #1a73e8;
      }
    </style>
  </head>
  <body>
    <h1>Quick Event Settings</h1>
    <p>
      Enter your Gemini API key to enable AI-powered event extraction. You can
      get a free key from
      <a href="https://aistudio.google.com/apikey" target="_blank">
        Google AI Studio</a
      >.
    </p>

    <label for="apiKey">Gemini API Key</label>
    <input type="text" id="apiKey" placeholder="Enter your API key" />
    <button id="save">Save</button>
    <div id="status" class="status"></div>

    <script src="options.js"></script>
  </body>
</html>
```

### Create the options logic

Open `options.js` and add the following:

```js
// options.js
const apiKeyInput = document.getElementById('apiKey');
const saveButton = document.getElementById('save');
const status = document.getElementById('status');

// Load the saved key when the page opens
chrome.storage.local.get('geminiApiKey', (data) => {
  if (data.geminiApiKey) {
    apiKeyInput.value = data.geminiApiKey;
  }
});

// Save the key when the user clicks Save
saveButton.addEventListener('click', () => {
  const apiKey = apiKeyInput.value.trim();

  if (!apiKey) {
    status.textContent = 'Please enter an API key.';
    status.style.color = '#d93025';
    return;
  }

  chrome.storage.local.set({ geminiApiKey: apiKey }, () => {
    status.textContent = 'Settings saved.';
    status.style.color = '#188038';
    setTimeout(() => {
      status.textContent = '';
    }, 2000);
  });
});
```

When the page loads, it reads any previously saved key from storage and
pre-fills the input so the user doesn't have to re-enter it. When they click
Save, it validates the input, writes the key to storage, and briefly shows a
confirmation message.

A few things to notice:

- We use `chrome.storage.local` instead of `localStorage`. The Chrome extension
  storage API works in service workers (where `localStorage` does not), and it's
  designed specifically for extensions.
- The key is saved in the user's local extension storage and only sent to the
  Gemini API when making a request. It never touches a third-party server.

### Try it out

1. Reload your extension at `chrome://extensions`
2. Click the **"Details"** button on the Quick Event card
3. Click **"Extension options"** (or right-click the extension icon → Options)
4. Enter your Gemini API key and click Save
5. Verify you see the "Settings saved." confirmation

## Capturing context from the page

Duration: 5:00

Now for the core interaction: the user highlights text on a web page,
right-clicks, and sees a "Create Calendar Event" option.

Open `background.js` and add the context menu setup:

```js
// background.js
// Create the right-click menu item when the extension is installed
chrome.runtime.onInstalled.addListener(() => {
  chrome.contextMenus.create({
    id: 'createCalendarEvent',
    title: 'Create Calendar Event',
    contexts: ['selection'],
  });
});
```

- `onInstalled` runs once when the extension is first installed (or updated).
  It's the right place to set up one-time things like context menus.
- `contexts: ['selection']` means the menu item only appears when the user has
  highlighted text. If nothing is selected, the option won't show up. Good UX:
  don't show options that don't apply.

Now add the click handler that receives the selected text:

```js
// background.js
// Listen for clicks on our context menu item
chrome.contextMenus.onClicked.addListener(async (info) => {
  if (info.menuItemId === 'createCalendarEvent') {
    const selectedText = info.selectionText;
    console.log('Selected text:', selectedText);

    // try {
    //   await getApiKey();
    // } catch {
    //   chrome.runtime.openOptionsPage();
    //   return;
    // }

    // try {
    //   const event = await extractEventDetails(selectedText);
    //   const calendarUrl = buildCalendarUrl(event);
    //   chrome.tabs.create({ url: calendarUrl });
    // } catch (error) {
    //   console.error('Failed to create event:', error);
    // }
  }
});
```

This is the entire flow of the extension — the commented-out code is the full
implementation, which we'll uncomment as we build out each function in the next
sections. For now, the `console.log` lets us verify that the click handler is
firing and the selected text is being captured correctly.

### Try it out

1. Reload your extension at `chrome://extensions`
2. Go to any web page and highlight some text
3. Right-click — you should see "Create Calendar Event" in the context menu
4. Click it and open the service worker console to confirm the selected text is logged

**Note**: To see console output from your background script, go to
`chrome://extensions`, find Quick Event, and click the **"service worker"**
link. This opens DevTools for the extension's background script.

## Calling the Gemini API

Duration: 8:00

This is the core of the extension. We'll send the selected text to the Gemini
API and ask it to extract structured event details as JSON.

### Helper: get the API key from storage

First, we need a way to retrieve the API key the user saved in the options page.
Add this helper function to `background.js`:

```js
// background.js
function getApiKey() {
  return new Promise((resolve, reject) => {
    chrome.storage.local.get('geminiApiKey', (data) => {
      if (data.geminiApiKey) {
        resolve(data.geminiApiKey);
      } else {
        reject(
          new Error(
            'No API key found. Please set your Gemini API key in the extension options.'
          )
        );
      }
    });
  });
}
```

Now that `getApiKey()` is defined, go back to the click handler in `background.js`
and uncomment the first try/catch block:

```js
// background.js
// Listen for clicks on our context menu item
chrome.contextMenus.onClicked.addListener(async (info) => {
  if (info.menuItemId === 'createCalendarEvent') {
    const selectedText = info.selectionText;
    console.log('Selected text:', selectedText);

    // UNCOMMENT THIS:
    try {
      await getApiKey();
    } catch {
      chrome.runtime.openOptionsPage();
      return;
    }

    // try {
    //   const event = await extractEventDetails(selectedText);
    //   const calendarUrl = buildCalendarUrl(event);
    //   chrome.tabs.create({ url: calendarUrl });
    // } catch (error) {
    //   console.error('Failed to create event:', error);
    // }
  }
});
```

This guards the rest of the handler — if the user hasn't saved an API key yet,
it opens the options page and stops execution.

### The Gemini API call

Now add the function that calls the Gemini API to your `background.js`. We're using the REST API
directly, no SDK needed:

```js
// background.js
async function callGemini(prompt) {
  const apiKey = await getApiKey();

  const response = await fetch(
    `https://generativelanguage.googleapis.com/v1beta/models/gemini-3.1-flash-lite:generateContent`,
    {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'x-goog-api-key': apiKey,
      },
      body: JSON.stringify({
        contents: [
          {
            parts: [{ text: prompt }],
          },
        ],
        generationConfig: {
          temperature: 0,
          responseMimeType: 'application/json',
        },
      }),
    }
  );

  if (!response.ok) {
    const error = await response.json();
    throw new Error(
      error?.error?.message ?? `Gemini API error: ${response.status}`
    );
  }

  const data = await response.json();
  return data.candidates[0].content.parts[0].text;
}
```

The important bits:

- **`gemini-3.1-flash-lite`** — The Gemini API offers several models that trade
  off capability, speed, and cost. At one end are the Pro models, which are the
  most capable but slower and more expensive. At the other end are Flash Lite
  models, which are optimised for low latency and cost efficiency. For a task
  like extracting structured data from a short text snippet, a Flash Lite model
  is more than capable — there's no need to pay for a Pro model.
- **`temperature: 0`** gives us deterministic, consistent output. This is data
  extraction, not creative writing. A temperature of 0 means the model will
  produce the same answer for the same input.
- **`responseMimeType: 'application/json'`** tells the Gemini API to return
  valid JSON. This is much better than hoping the model outputs parseable JSON
  on its own. The API enforces it.
- **`x-goog-api-key`** puts the API key in the request header, not in the URL
  query string. This is the recommended approach.

Positive: We're calling the Gemini API directly from the extension's background
service worker using `fetch`. No server required. The API key is stored locally
in the user's browser and sent directly to Google's API over HTTPS. No
intermediary server ever sees it. This keeps the architecture simple: it's just
the extension and the API.

### The extraction function

Now add the function that constructs the prompt and parses the result:

```js
// background.js
async function extractEventDetails(text) {
  const prompt = `
Extract the event details from the following text. Return a JSON object with these fields:

- "title": the name of the event
- "description": a brief description
- "location": the venue or address
- "start_date": start date in YYYY-MM-DD format
- "start_time": start time in HH:MM format (24-hour)
- "end_date": end date in YYYY-MM-DD format
- "end_time": end time in HH:MM format (24-hour)

Rules:
- If no year is provided, assume ${new Date().getFullYear()}.
- If no end time is provided, set it to one hour after the start time.
- If a field cannot be determined from the text, set its value to null.
- Do not convert or adjust times. Use them exactly as written.

Text:
${text}
  `.trim();

  const result = await callGemini(prompt);
  return JSON.parse(result);
}
```

### Why this prompt works

This prompt is carefully designed. Each decision matters:

1. **Explicit field names and formats.** We tell the model exactly what fields
   we want and what format each should be in (`YYYY-MM-DD`, `HH:MM` 24-hour).
   Without this, the model might return "October 18th" or "6pm", which are hard
   to parse reliably in code.

2. **Rules for edge cases.** Real event text is messy. It often doesn't include
   the year. Sometimes there's no end time. The prompt handles these cases
   explicitly instead of hoping the model guesses correctly.

3. **"Do not convert or adjust times."** Without this, the model might convert
   time zones or switch between 12-hour and 24-hour formats. We want the raw
   times as written.

4. **`null` for missing fields.** This gives us a consistent data shape. Our
   code can check for `null` instead of handling missing keys, empty strings, or
   the model inventing values.

5. **`responseMimeType: 'application/json'`** (in the API call) tells Gemini to
   return valid JSON. Combined with our prompt, this eliminates the need for
   messy JSON-fixing code. The API guarantees valid JSON output.

## Building the calendar URL

Duration: 5:00

Now we need to turn the structured event data into a Google Calendar URL. Google
Calendar accepts pre-filled event details through URL parameters.

The URL format looks like this:

```text
https://calendar.google.com/calendar/render?action=TEMPLATE
  &text=Event+Title
  &dates=20251018T100000/20251018T180000
  &details=Event+description
  &location=Event+venue
```

The `dates` parameter uses the format `YYYYMMDDTHHmmss/YYYYMMDDTHHmmss`
(start/end).

Add the URL builder function to `background.js`:

```js
// background.js
function buildCalendarUrl(event) {
  const url = new URL('https://calendar.google.com/calendar/render');

  const params = url.searchParams;
  params.set('action', 'TEMPLATE');

  if (event.title) {
    params.set('text', event.title);
  }

  if (event.description) {
    params.set('details', event.description);
  }

  if (event.location) {
    params.set('location', event.location);
  }

  if (event.start_date && event.start_time) {
    const startFormatted = formatDateTime(event.start_date, event.start_time);
    const endFormatted = formatDateTime(
      event.end_date ?? event.start_date,
      event.end_time ?? event.start_time
    );
    params.set('dates', `${startFormatted}/${endFormatted}`);
  }

  return url.toString();
}

function formatDateTime(date, time) {
  // date: "YYYY-MM-DD", time: "HH:MM"
  // output: "YYYYMMDDTHHmmss"
  const [year, month, day] = date.split('-');
  const [hours, minutes] = time.split(':');
  return `${year}${month}${day}T${hours}${minutes}00`;
}
```

Notice how clean this is:

- We use the `URL` API to construct the URL safely. No manual string
  concatenation that could break with special characters.
- We handle null values gracefully. If the AI couldn't determine the end date,
  we fall back to the start date.
- `formatDateTime` is trivial. Because we told the AI to give us dates in
  `YYYY-MM-DD` and times in `HH:MM` format, converting to the Google Calendar
  format is just string manipulation. No date parsing library needed.

This split is worth calling out: let AI do what AI is good at (understanding
natural language, extracting meaning from messy text), and let code do what code
is good at (formatting strings, building URLs, handling null values).

Now that all three functions are defined, go back to the click handler and
uncomment the second try/catch block:

```js
// background.js
// Listen for clicks on our context menu item
chrome.contextMenus.onClicked.addListener(async (info) => {
  if (info.menuItemId === 'createCalendarEvent') {
    const selectedText = info.selectionText;
    console.log('Selected text:', selectedText);

    try {
      await getApiKey();
    } catch {
      chrome.runtime.openOptionsPage();
      return;
    }

    // UNCOMMENT this:
    try {
      const event = await extractEventDetails(selectedText);
      const calendarUrl = buildCalendarUrl(event);
      chrome.tabs.create({ url: calendarUrl });
    } catch (error) {
      console.error('Failed to create event:', error);
    }
  }
});
```

## Try it out

Duration: 3:00

Your extension is complete. Let's test it end to end.

1. Reload the extension at `chrome://extensions`
2. Make sure you've set your API key in the extension options
3. Go to any web page with event information. Here's one you can try:
   [Build With AI Onitsha 2026](https://gdg.community.dev/events/details/google-gdg-onitsha-presents-build-with-ai-2026-2nd-edition/)
4. Highlight the event text
5. Right-click → **Create Calendar Event**
6. Google Calendar should open in a new tab with the event details pre-filled!

### Verify your results

For the Build With AI Onitsha event, check that:

- The title is "Build With AI 2026 (2nd Edition)"
- The date is May 9, 2026
- The start time is 10:00am
- The end time is 05:00pm
- The location includes "Innovation Growth Hub, 39 New Market Road, Onitsha, 434212"
- The description is populated

### Try different event formats

Test with different kinds of event text to see how the AI handles variety:

- A meetup page with informal date formatting ("next Saturday at 3pm")
- An event in a different language
- An event with minimal information (just a title and date)
- An event with lots of extra text around the details

Any page with event information will work. The AI is surprisingly good at
picking out the relevant details even when there's a lot of surrounding noise.

**Note**: The AI won't always extract every field correctly, especially from
very informal or ambiguous text. AI is probabilistic, not deterministic. The
prompt design minimizes errors, but edge cases will always exist.

## What's next

Duration: 2:00

You've built a working AI-powered Chrome extension. Here are some ways to take
it further.

### Explore Chrome's built-in AI APIs

Chrome now ships several AI APIs that run entirely on-device, with no API key or
internet connection required:

- **Summarizer API** — condense long content into key points
- **Translator API** — translate text between languages on-device
- **Language Detector API** — identify what language text is written in
- **Writer & Rewriter APIs** — generate and refine text
- **Proofreader API** — grammar and spelling correction (origin trial)
- **Prompt API** — open-ended prompting with Gemini Nano (extensions only)

These APIs are available from Chrome 138+ and are ideal for building extensions
that work offline and preserve user privacy.

### Learn about WebMCP

WebMCP is a new proposal that lets web pages expose structured tools directly to
AI agents in the browser. It's available behind a flag in Chrome Canary 146+.
Instead of extensions scraping pages, pages would tell AI agents what actions
are available.

### Resources

- [Chrome Built-in AI documentation](https://developer.chrome.com/docs/ai)
- [Chrome Extensions documentation](https://developer.chrome.com/docs/extensions)
- [Gemini API documentation](https://ai.google.dev/gemini-api/docs)
- [Chrome Built-in AI Early Preview Program](https://developer.chrome.com/docs/ai/built-in#get_started)
- [Source code for this codelab](https://github.com/ayoisaiah/quick-event-extension)

## Congratulations

Duration: 0:00

You've built an AI-powered Chrome extension. You selected text on a web page,
sent it to the Gemini API, received structured data back, and opened Google
Calendar with the event pre-filled.

Here's what you covered:

- How Chrome extensions are structured (manifest, background scripts, options
  pages)
- How to capture user context with the context menus API
- How to call the Gemini API from an extension using the REST endpoint
- How to write prompts that produce reliable structured output
- Why the boundary between AI and code matters
- How to let users securely provide their own API key

The whole extension is about 100 lines of JavaScript. No frameworks, no build
tools, no server. Browser extensions are just web technologies you already know,
applied in a new context.

### Further reading

- [Manifest V3 migration guide](https://developer.chrome.com/docs/extensions/develop/migrate)
- [Chrome Extensions samples](https://github.com/GoogleChrome/chrome-extensions-samples)
- [Gemini API structured output](https://ai.google.dev/gemini-api/docs/structured-output)
- [Service worker lifecycle](https://developer.chrome.com/docs/extensions/develop/concepts/service-workers/lifecycle)
