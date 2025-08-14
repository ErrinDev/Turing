# Turing
a server-client based voice assistant

(thanks Gemini, will update later but this is so i don't forget stuff)

Dynamic Wake Word Voice Assistant
A self-hosted, Python-based voice assistant where client agents dynamically fetch multiple, function-specific wake words from a central server. This allows for adding new voice commands without redeploying client code.
The system includes management APIs to reload configurations on the fly and a heartbeat system to monitor the health and status of all connected agents. It's built on open-source libraries to avoid vendor lock-in and API limits.
Core Concepts
 * Function-Specific Wake Words: The system listens for multiple phrases simultaneously. Each phrase can either trigger a direct action (e.g., "Computer, play my workout playlist") or activate a general-purpose assistant (e.g., "Hey Assistant, what's the weather like today?").
 * Dynamic Client Updates: Agents are "dumb" listeners. They download the latest library of wake word models from the server on startup. The server can also send a "push" notification to tell all active clients to refresh their libraries instantly.
 * Health Monitoring: Agents periodically send an asynchronous "heartbeat" signal to the server. This allows you to monitor the status of all your devices from a central dashboard or API endpoint.
System Architecture
The system uses a client-server model with management APIs and a heartbeat mechanism for monitoring.
Server (The Brain 🧠)
 * Manages & Serves Wake Words: Keeps a central configuration mapping wake word phrases to functions. It trains and serves these models to clients.
 * Processes Commands: Performs the heavy lifting: Speech-to-Text (STT), Natural Language Understanding (NLU), and Text-to-Speech (TTS).
 * Monitors Agent Health: Listens for heartbeats from clients and tracks their status (e.g., ONLINE, OFFLINE).
 * Exposes Management API: Provides endpoints to check system status, reload its own configuration, and trigger updates on all clients.
Client / Agent (The Ears 👂)
 * Listens for Wake Words: Loads multiple wake word models and listens for all of them at the same time.
 * Fetches Wake Words on Demand: Downloads the latest models from the server on startup and when prompted by the server.
 * Sends Heartbeats: Runs a non-blocking background task to send its status to the server at a regular interval (e.g., every 60 seconds).
 * Receives Push Updates: Runs a tiny web server to listen for "update" commands from the main server.
Recommended Tech Stack
 * Wake Word Engine: OpenWakeWord (Supports custom, multiple models).
 * Client-Server Communication: FastAPI (For both server and client-side APIs).
 * Asynchronous Tasks: Python's built-in asyncio library and httpx for making non-blocking API calls (perfect for heartbeats).
 * STT / TTS: Whisper (local STT) and Piper (local TTS).
Server API Endpoints
The server provides a REST API for management, monitoring, and core functions.
Client Management
 * POST /api/clients/register
   * Description: A new client calls this on startup to register itself with the server.
   * Body: { "client_id": "living_room_pi", "ip_address": "192.168.1.10" }
 * POST /api/clients/heartbeat
   * Description: Clients send this periodically to show they are still online.
   * Body: { "client_id": "living_room_pi" }
System Monitoring & Control
 * GET /api/system/status
   * Description: Get the health status of all registered clients. Ideal for a dashboard.
   * Response:
     {
  "clients": [
    {
      "client_id": "living_room_pi",
      "status": "ONLINE",
      "last_seen": "2025-08-14T13:50:00Z"
    },
    {
      "client_id": "kitchen_speaker",
      "status": "OFFLINE",
      "last_seen": "2025-08-14T12:30:00Z"
    }
  ]
}

 * POST /api/system/reload
   * Description: Tells the server to reload its wakewords.json file from disk without a full restart.
   * Body: (empty)
 * POST /api/system/update-clients
   * Description: Triggers the wake word update process on all currently ONLINE clients.
   * Body: (empty)
Core Functions
 * GET /api/wakewords: Returns the wake word configuration JSON.
 * GET /api/wakewords/models/{model_name}: Serves a specific .onnx model file.
 * POST /api/command: Processes a general voice command (receives audio, returns audio).
Client (Agent) Logic
The client script becomes slightly more complex to handle asynchronous tasks.
 * Startup Sequence:
   * Register with the server via POST /api/clients/register.
   * Call fetch_and_load_wakewords().
   * Start the client's own FastAPI server to listen for update pushes (POST /update).
   * Start the asynchronous heartbeat task using asyncio.create_task(send_heartbeats()).
   * Start the main listening_loop().
 * Asynchronous Heartbeat Task (send_heartbeats):
   * This function runs in an infinite async loop.
   * It sends a POST request to /api/clients/heartbeat with its ID.
   * It then sleeps for a configured interval (e.g., await asyncio.sleep(60)).
   * Because it uses await, it doesn't block the rest of the program.
 * Wake Word Fetching (fetch_and_load_wakewords):
   * Sends GET to /api/wakewords.
   * Downloads all required model files.
   * Reloads the OpenWakeWord instance with the new models.
 * Listening Loop (listening_loop):
   * Continuously listens to the microphone.
   * When OpenWakeWord detects a phrase, it checks the command type from its fetched config.
   * For a direct_action, it sends a POST request to the action's specific endpoint.
   * For a general_command, it records audio, sends it to /api/command, and plays the audio response.

