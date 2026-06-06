# Agent Voice Bridge Help

Agent Voice Bridge is a SIP/RTP phone bridge controlled by MCP. It is not an LLM agent. It handles phone audio, SIP registration, RTP media, STT transcripts, TTS playback, call state, webhooks, logs, and MCP tools.

The external agent owns reasoning. The agent decides what to say and controls calls through MCP.

## Basic Flow

```text
caller speaks
-> bridge receives RTP audio
-> STT creates transcript
-> optional webhook wakes agent
-> agent calls get_call_status(call_id)
-> agent decides what to do
-> agent calls say(call_id, text)
-> bridge speaks through TTS over RTP
```

The bridge only speaks when `say`, REST Say, the Calls tab Say button, or `make_call_and_say` is used.

## MCP HTTP Wrapper

Default MCP HTTP endpoint:

```text
http://127.0.0.1:8765
```

Useful endpoints:

```text
GET  /status
GET  /tools
POST /call
```

Example:

```bash
curl -s http://127.0.0.1:8765/call \
  -H "Content-Type: application/json" \
  -d '{"name":"list_active_calls","arguments":{}}'
```

## MCP Tools

### `list_active_calls()`

Returns active call summaries.

Use this when the agent needs to discover current calls.

### `get_call_status(call_id)`

Returns the full status for one call, including:

- caller/called numbers
- transcript state
- current partial transcript
- recent caller and agent messages
- STT/TTS status
- RTP counters
- call duration
- media diagnostics

Agents should call this after receiving a webhook event.

### `say(call_id, text)`

Speaks exact text into an active call using the configured TTS provider.

Use this for normal conversation after the call is active.

### `stop_speaking(call_id)`

Stops current or queued TTS audio.

Use this for interruption or barge-in handling.

### `send_message(call_id, text)`

Stores manual/debug text in call state.

This does not speak and does not trigger an LLM.

### `simulate_barge_in(call_id)`

Triggers the same interruption path as caller barge-in.

Useful for testing.

### `hangup_call(call_id)`

Hangs up an active SIP call when possible.

### `transfer_call(call_id, destination)`

Transfers a call when AMI support is available.

### `make_call(number)`

Starts an outbound SIP call and returns after the remote side answers.

Use this when no first spoken message is ready yet.

### `make_call_and_say(number, text)`

Starts an outbound SIP call, prepares TTS while ringing, speaks the first message after answer, and returns the `call_id`.

Best pattern for agent-started conversations:

```text
make_call_and_say(number, first_text) -> call_id
say(call_id, next_text)
say(call_id, next_text)
hangup_call(call_id)
```

## Agent Instructions

Give your agent instructions like this:

```text
You control phone calls through Agent Voice Bridge MCP.

When a webhook event arrives, first call get_call_status(call_id).
Read recent_messages, last_transcript, current_partial_transcript, and call state.
Decide the next action.
Use say(call_id, text) to speak.
Use stop_speaking(call_id) if the caller interrupts.
Use hangup_call(call_id) only when the conversation should end.

For outbound conversations started by the agent, use make_call_and_say(number, first_text), keep the returned call_id, then continue with say(call_id, text).
```

## Webhooks

Webhooks are wake-up notifications. They do not control the call by themselves.

Configure webhooks in:

```text
Configuration -> Agent
```

Common webhook URL example:

```text
http://127.0.0.1:8644/webhook/agent-voice-call
```

Events:

```text
call.started
transcript.final
call.ended
transcript.partial
```

`transcript.partial` is only sent when partial notifications are enabled.

Webhook payloads include:

- event type
- timestamp
- call_id
- caller number
- called number
- event text
- full call status
- MCP URLs

After receiving a webhook, the agent should call:

```text
get_call_status(call_id)
```

Then decide whether to call:

```text
say(call_id, text)
stop_speaking(call_id)
hangup_call(call_id)
transfer_call(call_id, destination)
```

## Inbound Calls

Inbound calls are answered by the bridge through SIP.

The bridge does not auto-greet.

If you want a greeting, the agent or FreePBX must provide it.

Typical inbound flow:

```text
caller calls bridge extension
bridge answers
STT starts
caller speaks
webhook sends transcript.final
agent calls get_call_status
agent calls say
```

## Outbound Calls

For agent-started conversations, prefer:

```text
make_call_and_say(number, first_text)
```

This prepares speech while the phone is ringing and reduces first-response delay.

Then continue the conversation with:

```text
say(call_id, text)
```

Use `make_call(number)` only if the agent wants to place the call first and decide what to say later.

## Web UI

Default web UI:

```text
http://127.0.0.1:8088
```

Tabs:

- Dashboard: runtime status, SIP, MCP, calls today, voice I/O
- Calls: active and recent calls, transcripts, Say, Stop Speaking, Hangup
- Logs: live logs
- Help: this help document
- Configuration: SIP, STT, TTS, MCP, Agent webhook, Web settings

## FreePBX Notes

Agent Voice Bridge registers as a normal SIP extension.

Recommended setup:

- PJSIP extension
- UDP transport
- PCMU/ulaw first
- PCMA/alaw optional
- Direct media disabled
- RTP anchored through Asterisk
- Advertise host set to the bridge LAN IP reachable by FreePBX

The bridge default SIP bind is:

```text
0.0.0.0:5062
```

Default RTP range:

```text
40000-40100
```

## Troubleshooting

### Caller hears nothing

Check:

- TTS provider is configured
- API key is set
- MCP `say` was actually called
- RTP packets are being sent
- call is still active

### First words are clipped

For outbound agent-started calls, use:

```text
make_call_and_say(number, first_text)
```

It prepares audio while ringing and waits for media before speech.

### STT does not hear normal speech

Check:

- STT provider is configured
- OpenAI realtime mode is selected for phone calls
- microphone/caller RTP is reaching the bridge
- Logs tab for STT errors

### SIP is disconnected

Check:

- FreePBX host
- extension username/password
- local SIP bind port
- advertise host
- firewall
- FreePBX PJSIP registration status

### Webhook fails

Check:

- webhook URL is reachable from the bridge
- webhook server returns HTTP 200
- auth token matches if configured
- Logs tab for delivery warnings
