# AI Voice Agent, IVR & Multichannel Messaging

A Python-based AI voice assistant and IVR system for customer conversations over the phone. The application connects telephone calls to an asynchronous speech-processing pipeline and integrates WhatsApp and SMS messaging to support customer interactions.

During a live call, the system can also send relevant images to the caller through WhatsApp. This supports product presentations, sales conversations, and visual clarification while the voice conversation continues.

## Architecture Overview

```
                         Caller
                            |
                      Phone Call
                            |
                      Twilio Voice
                            |
                  Bidirectional WebSocket
                            |
                            v
              +---------------------------+
              |     FastAPI Application   |
              |      Voice Orchestrator   |
              +---------------------------+
                            |
                            v
                  Voice Activity Detection
                       (Silero VAD)
                            |
                            v
                 Speech-to-Text (STT)
                 Groq Whisper Large-v3
                            |
                            v
              +-----------------------------+
              |  Semantic Cache (Redis)     |
              +-----------------------------+
                  |                    |
               cache hit           cache miss
                  |                    |
                  |                    v
                  |          Context Assembly
                  |       (recent turns + RAG retrieval)
                  |                    |
                  |                    v
                  |          Language Model (LLM)
                  |            Gemini 2.5 Flash
                  |                    |
                  |                    v
                  |         Store response in cache
                  |                    |
                  +----------+---------+
                             |
                             v
                  Text-to-Speech (TTS)
                             |
                             v
                   Twilio Voice Stream
                             |
                             v
                          Caller

       +------------------------------------------------+
       |          Messaging Integrations                |
       |                                                |
       |  FastAPI + Twilio SMS       WhatsApp Business  |
       |                             Meta Graph API     |
       +------------------------------------------------+
                    |                       |
                    v                       v
              SMS messages          Text and image messages
                                            |
                                            v
                                      Caller's WhatsApp
```

## Key Features

### 1. Real-Time Voice Conversation
- Handles telephone audio through Twilio Voice Media Streams.
- Uses WebSockets for bidirectional audio communication.
- Uses Python `asyncio` for asynchronous processing.
- Coordinates speech detection, transcription, LLM response generation, and speech output.

### 2. Voice Activity Detection
- Uses Silero VAD to detect speech in incoming audio frames.
- Helps identify when a caller starts and stops speaking.
- Avoids sending silence for transcription unnecessarily.
- Supports interruption handling when the caller speaks while the assistant is responding.

### 3. Barge-In and Interruption Handling
- Detects caller speech while assistant audio is being played.
- Uses an asynchronous interrupt signal to coordinate cancellation.
- Sends a Twilio `clear` event to clear queued playback audio when applicable.
- Stops or cancels active response/audio tasks so the system can process the caller's new input.

### 4. Speech Recognition
- Uses Groq Whisper Large-v3 for speech transcription.
- Supports an optional contextual prompt (for example, business or product terms) to improve transcription accuracy.

### 5. LLM-Based Response Generation
- Uses Google Gemini 2.5 Flash to generate conversational responses.
- Supplies the recognized user input and relevant conversation context to the model.
- Uses system instructions to guide response generation.

### 6. Conversation Context and RAG
- Includes recent conversation turns in the active prompt to keep the dialogue coherent.
- Uses RAG retrieval to bring relevant historical conversation context into the response workflow.

### 7. Caching
The system uses several layers of caching to reduce latency and repeated LLM calls.

**Shared semantic cache (Redis)**
- Stores previously generated responses in Redis, keyed by the meaning of the query rather than its exact text.
- When a new caller query is semantically similar to a cached one (for example, a common pricing question), the cached response is reused and the LLM call is skipped.
- The cache is shared across calls, so a response generated for one caller can be reused for others.
- On a cache miss, the response is generated normally and then stored for future use.
- Cache entries use a configurable TTL and similarity threshold, so stale content (such as outdated prices) expires and only close matches are reused.
- Intended for generic, repeatable queries. Caller-specific or sensitive responses should not be written to the shared cache.

**Prompt caching**
- Prompts are organized with static content (system instructions, reference material) first and dynamic content (recent turns, the current query) last.
- This layout lets provider-side prompt caching be used where available.

**Why it matters**
- Lower response latency for repeated questions, which matters in live voice conversations.
- Fewer LLM calls and lower per-call cost.

### 8. Streaming Text-to-Speech
- Converts generated or cached response text into audio.
- Streams audio output through the telephony connection.
- Works with the configured TTS provider and its supported streaming interface.

### 9. WhatsApp Messaging and Image Sharing During Calls
The system integrates WhatsApp messaging with the live voice conversation. While speaking with a caller, the application can send relevant images to the caller's WhatsApp number.

This supports use cases such as:

- **Product sales:** Send product images while explaining product features over the phone.
- **Visual clarification:** Share an image to help the caller understand a product, option, or item being discussed.
- **Customer assistance:** Send a relevant visual reference during the conversation.
- **Multichannel interaction:** Continue the voice conversation while the caller views the image in WhatsApp.

The WhatsApp integration uses the Meta Graph API.

### 10. SMS Integration
- Provides an inbound SMS webhook using FastAPI.
- Sends outbound SMS messages through Twilio Messaging.
- Supports notification and follow-up messaging use cases.

## Technology Stack

| Area | Technologies |
|---|---|
| Programming language | Python |
| Backend framework | FastAPI, Uvicorn |
| Asynchronous processing | asyncio, aiohttp |
| Real-time communication | WebSockets |
| Telephony | Twilio Voice |
| Voice activity detection | Silero VAD, PyTorch |
| Speech-to-text | Groq Whisper Large-v3 |
| LLM | Google Gemini 2.5 Flash |
| Retrieval and context | RAG retrieval, recent-turn conversation memory |
| Caching | Redis (shared semantic cache), provider-side prompt caching |
| Text-to-speech | Configured streaming TTS provider |
| SMS | Twilio Messaging |
| WhatsApp | WhatsApp Business Platform, Meta Graph API |
| Local webhook testing | Cloudflare Tunnel or ngrok |

## Project Structure

The following is a logical overview of the application components. Adapt the paths to match the actual source-code repository.

```
project/
├── core/
│   ├── vad.py                  # Voice activity detection
│   ├── stt.py                  # Speech-to-text integration
│   ├── llm.py                  # LLM response generation
│   ├── tts.py                  # Text-to-speech integration
│   ├── rag.py                  # Retrieval of relevant conversation context
│   └── cache.py                # Redis semantic cache and prompt layout
│
├── telephony/
│   ├── voice_socket.py         # Twilio voice WebSocket handling
│   └── interrupt_handler.py    # Barge-in and task interruption
│
├── whatsapp/
│   ├── send_message.py         # WhatsApp text messaging
│   └── send_media.py           # WhatsApp image sharing
│
├── sms/
│   ├── receive_sms.py          # Incoming SMS webhook
│   └── send_sms.py             # Outgoing SMS
│
├── config.py                   # Application configuration
├── requirements.txt            # Python dependencies
└── main.py                     # FastAPI application entry point
```

## Code Explanation

**`main.py` — FastAPI Application**
Initializes the FastAPI server and registers the routes for voice calls, WebSocket media streams, and messaging webhooks.

**`telephony/voice_socket.py` — Voice Stream Handler**
Handles the bidirectional audio stream from Twilio. It receives audio messages, coordinates the speech-processing pipeline, and sends generated audio back to the caller.

**`core/vad.py` — Voice Activity Detection**
Processes incoming audio frames with Silero VAD to identify speech activity. Its output helps decide when to process caller audio and when to interrupt assistant playback.

**`core/stt.py` — Speech-to-Text**
Sends caller audio for transcription using Groq Whisper Large-v3, with an optional contextual prompt.

**`core/cache.py` — Semantic Cache**
Looks up the caller's query in the shared Redis semantic cache before any LLM call. On a hit, the cached response is returned directly. On a miss, the generated response is stored with a TTL. It also keeps prompts ordered (static content first) to support provider-side prompt caching.

**`core/rag.py` — Context Retrieval**
Retrieves relevant historical conversation context and combines it with recent turns for the active prompt.

**`core/llm.py` — LLM Response Generation**
Sends the transcription and assembled context to Gemini 2.5 Flash. The model generates the response according to the configured system instructions.

**`core/tts.py` — Text-to-Speech**
Converts the response into speech using the configured TTS service and provides audio chunks to the voice-streaming component.

**`telephony/interrupt_handler.py` — Barge-In Handling**
Coordinates interruption when the caller begins speaking during assistant playback. It signals active tasks to stop and clears queued audio from the Twilio stream where supported.

**`whatsapp/send_message.py` — WhatsApp Text Messaging**
Uses the Meta Graph API to send WhatsApp text messages to the caller.

**`whatsapp/send_media.py` — WhatsApp Image Sharing**
Sends image media through WhatsApp. The voice conversation can trigger this when a product image or visual reference is useful for sales or clarification.

**`sms/receive_sms.py` and `sms/send_sms.py` — SMS Integration**
Handle inbound SMS webhook events and outbound SMS messages through Twilio Messaging.

## Setup and Installation

### 1. Clone the Repository
Replace the URL with the actual repository URL.

```bash
git clone https://github.com/<your-username>/<your-repository>.git
cd <your-repository>
```

### 2. Create a Virtual Environment
```bash
python -m venv venv
```

Windows:
```bash
venv\Scripts\activate
```

Linux or macOS:
```bash
source venv/bin/activate
```

### 3. Install Dependencies
```bash
pip install -r requirements.txt
```

Example dependency list — confirm package versions and add any provider-specific TTS, embedding, or WhatsApp dependencies required by your implementation:

```
fastapi
uvicorn[standard]
aiohttp
python-multipart
twilio
numpy
scipy
torch
groq
google-genai
redis
python-dotenv
```

### 4. Start Redis
The semantic cache requires a running Redis instance (local or hosted). For local development with Docker:

```bash
docker run -d --name redis -p 6379:6379 redis:7
```

If you use Redis vector search for semantic matching, use a Redis distribution that supports it (for example, Redis Stack).

## Environment Configuration

Create a `.env` file in the project root and configure the credentials used by your integrations.

```env
# Twilio
TWILIO_ACCOUNT_SID=your_twilio_account_sid
TWILIO_AUTH_TOKEN=your_twilio_auth_token
TWILIO_PHONE_NUMBER=your_twilio_phone_number
TWILIO_MESSAGING_SERVICE_SID=your_messaging_service_sid

# Speech-to-text
GROQ_API_KEY=your_groq_api_key

# Language model
GEMINI_API_KEY=your_gemini_api_key

# Redis / semantic cache
REDIS_URL=redis://localhost:6379
CACHE_TTL_SECONDS=3600
CACHE_SIMILARITY_THRESHOLD=0.90

# WhatsApp Business Platform
WHATSAPP_ACCESS_TOKEN=your_whatsapp_access_token
WHATSAPP_PHONE_NUMBER_ID=your_whatsapp_phone_number_id
WHATSAPP_API_VERSION=your_graph_api_version
```

Use the exact environment variable names expected by your application code. Tune the cache TTL and similarity threshold to your content: a higher threshold means fewer, safer cache hits.

## Running the Application

Start the FastAPI server using the actual module and application variable in your project:

```bash
uvicorn main:app --reload --port 8000
```

### Exposing Local Webhooks

For local development and webhook testing, expose the FastAPI server using Cloudflare Tunnel:

```bash
cloudflared tunnel --url http://localhost:8000
```

Or use ngrok:

```bash
ngrok http 8000
```

Configure the generated public endpoint in the relevant Twilio and WhatsApp webhook settings. Use the actual route paths implemented in your application.

## Twilio Configuration

**Voice Calls**
Configure the Twilio phone number's Voice webhook to point to the application's voice entry endpoint. The endpoint should return the TwiML required to connect the call to the WebSocket media stream.

**SMS**
Configure the incoming SMS webhook URL to point to the FastAPI route that receives Twilio messaging events.

## WhatsApp Configuration

Configure the WhatsApp Business Platform integration with the Meta Graph API credentials used by the application.

The application uses the configured WhatsApp sender to deliver text messages and image media to the caller's WhatsApp number. The voice workflow can invoke the image-sharing component when a product image or visual explanation is needed during the call.

## Voice Conversation Flow

1. Twilio connects the caller to the FastAPI application.
2. The application receives the caller's audio through a WebSocket media stream.
3. Silero VAD detects speech activity.
4. The audio is sent to the STT service for transcription.
5. The transcript is checked against the Redis semantic cache.
   - **Cache hit:** the cached response is used and the LLM call is skipped.
   - **Cache miss:** recent conversation turns and RAG-retrieved context are assembled and passed to Gemini. The generated response is then stored in the cache.
6. The response is converted into audio by the TTS component.
7. The audio is streamed back to the caller through Twilio.
8. If the caller speaks during playback, the interruption handler stops the current response and clears queued audio where supported.
9. When visual information is useful, the application invokes the WhatsApp integration to send a relevant image to the caller while the voice conversation continues.

## WhatsApp Image-Sharing Flow

1. The caller discusses a product or asks for visual clarification during the phone conversation.
2. The application identifies the need to share a visual reference through its conversation workflow.
3. The WhatsApp media component sends the selected image using the Meta Graph API.
4. The caller receives the image in WhatsApp and can view it while continuing the telephone conversation.
5. The voice assistant continues handling the caller's questions.

## Notes

- Replace placeholder repository URLs, environment variables, and webhook paths with the values used by the actual implementation.
- Keep API credentials out of the README and source-control repository.
- The project structure and dependency list above are documentation examples; adjust them to match the code that is actually published.
