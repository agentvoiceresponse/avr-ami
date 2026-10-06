# AVR-AMI: Agent Voice Response and Asterisk Manager Interface Integration

## Overview

AVR-AMI is a Node.js application that provides a seamless integration with Asterisk Manager Interface (AMI) for call control operations. It enables the management of calls through a simple API interface, allowing for call transfers, hangups, and new call origination. This service is designed to work in conjunction with Agent Voice Response's Large Language Models (LLMs) to provide intelligent call handling capabilities.

## Features

- **Call Control Operations**:
  - Transfer active calls to different extensions
  - Hang up ongoing calls
  - Originate new outbound calls
- **UUID-based Call Tracking**: Each call is tracked using a unique UUID generated before the Asterisk AudioSocket Application is invoked
- **Simple REST API**: Easy-to-use endpoints for call management
- **LLM Integration**: Seamless integration with Agent Voice Response's Large Language Models for intelligent call handling

## Prerequisites

- Node.js 
- Asterisk server with Manager Interface enabled

## Installation

1. Clone the repository:
   ```bash
   git clone https://github.com/agentvoiceresponse/avr-ami.git
   cd avr-ami
   ```

2. Install dependencies:
   ```bash
   npm install
   ```

3. Configure your environment variables in `.env`:
   ```
   PORT=6006
   HOST=127.0.0.1
   ALLOWED_CONTEXTS=avr-transfer
   ALLOWED_EXTENSIONS=100,200
   AMI_HOST=127.0.0.1
   AMI_PORT=5038
   AMI_USERNAME=avr
   AMI_PASSWORD=avr
   ```

## API Endpoints

These endpoints are primarily used by Agent Voice Response's Large Language Models to control calls through natural language processing:

### POST /transfer

Transfer an active call to a different extension.
Only contexts and extensions in `ALLOWED_CONTEXTS` / `ALLOWED_EXTENSIONS` are accepted when they are set (see [Security](#security)); otherwise `403`.

**Request Body:**
```json
{
  "uuid": "call-uuid",
  "exten": "1234",
  "context": "from-internal",
  "priority": 1
}
```

### POST /hangup

Hang up an active call.

**Request Body:**
```json
{
  "uuid": "call-uuid"
}
```

### POST /originate

Initiate a new outbound call.

**Request Body:**
```json
{
  "channel": "SIP/trunk/1234567890",
  "exten": "1234",
  "context": "from-internal",
  "priority": 1,
  "callerid": "Agent Voice Response <avr>"
}
```

### POST /setvar

Set a channel variable on an active call, for example so the dialplan a later `/transfer`
leads to knows something about the call (a redirect keeps the channel and its variables).

**Request Body:**
```json
{
  "uuid": "call-uuid",
  "variable": "CALL_LANGUAGE",
  "value": "nl"
}
```

`variable` may contain letters, digits and underscores; `value` up to 64 letters, digits,
spaces, `-`, `_` and `.` -- nothing the dialplan would expand (`${...}`, `$[...]`) or split on.

## Security

The API has no authentication: set `HOST=127.0.0.1` (default `0.0.0.0`) when AVR runs on the
same host, so only local services can hang up, transfer or set variables on calls.

### Transfer allowlist

Transfer destinations usually come from an AI agent, and callers can steer what the agent asks
for. If `/transfer` accepts any context, a caller can talk the agent into a transfer to a context
that dials out (e.g. `from-internal`) and to an international or premium-rate number: toll fraud
by phone call. Restrict it:

- `ALLOWED_CONTEXTS`: comma-separated contexts `/transfer` may use
- `ALLOWED_EXTENSIONS`: comma-separated extensions `/transfer` may use

When either is set, priority must be `1` (so a transfer can't skip the first steps of an
extension), and any other request is refused with `403` and a message saying what is allowed,
which the agent can pass on to the caller. Unset, nothing is restricted and a warning is logged at
startup.

Use a dedicated context that contains only the extensions callers may reach, for example:

```
[avr-transfer]
exten => 100,1,Dial(PJSIP/100,30)
exten => 200,1,Dial(PJSIP/200,30)
```

with `ALLOWED_CONTEXTS=avr-transfer` and `ALLOWED_EXTENSIONS=100,200`. Then even the context alone
can't dial out.

`/originate` can place any call and is not restricted by these settings: keep the API reachable
only by AVR's own services (`HOST=127.0.0.1` or a firewall on its port).

## How It Works

1. **Call Identification**:
   - Each call is assigned a unique UUID before the AudioSocket Application is invoked in the Asterisk dialplan
   - The UUID is used to track and manage the call throughout its lifecycle

2. **AMI Connection**:
   - The application maintains a persistent connection to Asterisk Manager Interface
   - All call control operations are executed through AMI actions

3. **Call Management**:
   - Calls can be transferred between extensions
   - Active calls can be terminated
   - New outbound calls can be initiated
   - All operations are performed using the call's UUID for identification

4. **LLM Integration**:
   - Agent Voice Response's Large Language Models process natural language commands
   - The LLMs determine the appropriate call control action based on the conversation context
   - Actions are executed through the API endpoints to control the call flow
   - The system provides real-time feedback to the LLMs about the success or failure of operations

## Error Handling

The application includes robust error handling:
- Validates all incoming requests
- Provides clear error messages for failed operations
- Ensures proper cleanup of resources
- Maintains connection stability with Asterisk

