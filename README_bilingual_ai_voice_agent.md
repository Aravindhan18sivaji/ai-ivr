# Bilingual AI Voice Agent, IVR & Multichannel Messaging

A Python-based AI voice assistant and IVR system for customer conversations in English and Khaleeji Arabic. The application connects telephone calls to an asynchronous speech-processing pipeline and integrates WhatsApp and SMS messaging to support customer interactions.

During a live call, the system can also send relevant images to the caller through WhatsApp. This can support product presentations, sales conversations, and visual clarification while the voice conversation continues.

## Architecture Overview

```text
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
                 Language Model (LLM)
                   Gemini 2.5 Flash
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
                                      Caller’s WhatsApp
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
- Supports interruption handling when the caller speaks while the assistant is responding.

### 3. Barge-In and Interruption Handling

- Detects caller speech while assistant audio is being played.
- Uses an asynchronous interrupt signal to coordinate cancellation.
- Sends a Twilio `clear` event to clear queued playback audio when applicable.
- Stops or cancels active response/audio tasks so the system can process the caller's new input.

### 4. Bilingual Speech Recognition

- Uses Groq Whisper Large-v3 for speech transcription.
- Supports English and Khaleeji Arabic conversations.
- Uses contextual prompt terms, including common Khaleeji expressions, to provide the transcription model with dialect context.

Example contextual prompt:

```python
arabic_prompt = (
    "زين، بس، يلا، صح، ماكو مشكلة، شلونك. "
    "This is a bilingual Arabic and English call."
)
```

### 5. LLM-Based Response Generation

- Uses Google Gemini 2.5 Flash to generate conversational responses.
- Supplies the recognized user input and relevant conversation context to the model.
- Uses system instructions to guide bilingual response generation.

### 6. Streaming Text-to-Speech

- Converts generated response text into audio.
- Streams audio output through the telephony connection.
- Works with the configured TTS provider and its supported streaming interface.

### 7. WhatsApp Messaging and Image Sharing During Calls

The system integrates WhatsApp messaging with the live voice conversation. While speaking with a caller, the application can send relevant images to the caller's WhatsApp number.

This supports use cases such as:

- **Product sales:** Send product images while explaining product features over the phone.
- **Visual clarification:** Share an image to help the caller understand a product, option, or item being discussed.
- **Customer assistance:** Send a relevant visual reference during the conversation.
- **Multichannel interaction:** Continue the voice conversation while the caller views the image in WhatsApp.

The WhatsApp integration uses the Meta Graph API. The application connects the voice conversation flow with the messaging functionality so that an image can be sent when required by the interaction.

### 8. SMS Integration

- Provides an inbound SMS webhook using FastAPI.
- Sends outbound SMS messages through Twilio Messaging.
- Supports notification and follow-up messaging use cases.

## Technology Stack

| Area | Technologies |
|---|---|
| Programming language | Python |
| Backend framework | FastAPI, Uvicorn |
| Asynchronous processing | `asyncio`, `aiohttp` |
| Real-time communication | WebSockets |
| Telephony | Twilio Voice |
| Voice activity detection | Silero VAD, PyTorch |
| Speech-to-text | Groq Whisper Large-v3 |
| LLM | Google Gemini 2.5 Flash |
| Text-to-speech | Configured streaming TTS provider |
| SMS | Twilio Messaging |
| WhatsApp | WhatsApp Business Platform, Meta Graph API |
| Local webhook testing | Cloudflare Tunnel or ngrok |

## Project Structure

The following is a logical overview of the application components. Adapt the paths to match the actual source-code repository.

```text
project/
├── core/
│   ├── vad.py                  # Voice activity detection
│   ├── stt.py                  # Speech-to-text integration
│   ├── llm.py                  # LLM response generation
│   └── tts.py                  # Text-to-speech integration
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

### `main.py` — FastAPI Application

The application entry point initializes the FastAPI server and registers the required routes for voice calls, WebSocket media streams, and messaging webhooks.

### `telephony/voice_socket.py` — Voice Stream Handler

Handles the bidirectional audio stream from Twilio. It receives audio messages, coordinates the speech-processing pipeline, and sends generated audio back to the caller.

### `core/vad.py` — Voice Activity Detection

Processes incoming audio frames with Silero VAD to identify speech activity. Its output helps the application decide when to process caller audio and when to interrupt assistant playback.

### `core/stt.py` — Speech-to-Text

Sends caller audio for transcription using Groq Whisper Large-v3. Contextual prompt terms help provide language and dialect context for bilingual conversations.

### `core/llm.py` — LLM Response Generation

Sends the transcription and relevant conversation context to Gemini 2.5 Flash. The model generates the response according to the configured system instructions.

### `core/tts.py` — Text-to-Speech

Converts the generated response into speech using the configured TTS service and provides audio chunks to the voice-streaming component.

### `telephony/interrupt_handler.py` — Barge-In Handling

Coordinates interruption when the caller begins speaking during assistant playback. It signals active tasks to stop and clears queued audio from the Twilio stream where supported.

### `whatsapp/send_message.py` — WhatsApp Text Messaging

Uses the Meta Graph API integration to send WhatsApp text messages to the caller.

### `whatsapp/send_media.py` — WhatsApp Image Sharing

Sends image media through WhatsApp. The voice conversation can trigger this functionality when a product image or visual reference is useful for sales or clarification.

### `sms/receive_sms.py` and `sms/send_sms.py` — SMS Integration

Handle inbound SMS webhook events and outbound SMS messages through Twilio Messaging.

## Setup and Installation

### 1. Clone the Repository

Replace the URL with the actual repository URL after creating the GitHub repository.

```bash
git clone https://github.com/<your-username>/<your-repository>.git
cd <your-repository>
```

### 2. Create a Virtual Environment

```bash
python -m venv venv
```

Activate it on Windows:

```bash
venv\Scripts\activate
```

Activate it on Linux or macOS:

```bash
source venv/bin/activate
```

### 3. Install Dependencies

```bash
pip install -r requirements.txt
```

Example dependency list — confirm package versions and add any provider-specific TTS or WhatsApp dependencies required by your implementation:

```text
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
python-dotenv
```

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

# WhatsApp Business Platform
WHATSAPP_ACCESS_TOKEN=your_whatsapp_access_token
WHATSAPP_PHONE_NUMBER_ID=your_whatsapp_phone_number_id
WHATSAPP_API_VERSION=your_graph_api_version
```

Use the exact environment variable names expected by your application code.

## Running the Application

Start the FastAPI server using the actual module and application variable in your project.

For example:

```bash
uvicorn main:app --reload --port 8000
```

## Exposing Local Webhooks

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

### Voice Calls

Configure the Twilio phone number's Voice webhook to point to the application's voice entry endpoint. The endpoint should return the TwiML required to connect the call to the WebSocket media stream.

### SMS

Configure the incoming SMS webhook URL to point to the FastAPI route that receives Twilio messaging events.

## WhatsApp Configuration

Configure the WhatsApp Business Platform integration with the Meta Graph API credentials used by the application.

The application uses the configured WhatsApp sender to deliver text messages and image media to the caller's WhatsApp number. The voice workflow can invoke the image-sharing component when a product image or visual explanation is needed during the call.

## Voice Conversation Flow

1. Twilio connects the caller to the FastAPI application.
2. The application receives the caller's audio through a WebSocket media stream.
3. Silero VAD detects speech activity.
4. The audio is sent to the STT service for transcription.
5. The transcript and relevant conversation context are passed to Gemini.
6. The generated response is converted into audio by the TTS component.
7. The audio is streamed back to the caller through Twilio.
8. If the caller speaks during playback, the interruption handler coordinates stopping the current response and clearing queued audio where supported.
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
