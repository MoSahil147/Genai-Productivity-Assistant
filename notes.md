## Multi-Agent Flow Example
Input: "Schedule a meeting tomorrow and create a task for it"

1. Coordinator reads input, identifies TWO intents:
   - Intent 1: Schedule meeting (→ Calendar sub-agent)
   - Intent 2: Create task (→ Task sub-agent)

2. Coordinator calls Calendar sub-agent FIRST
   - Calendar sub-agent creates meeting, returns meeting_id to shared state

3. Coordinator calls Task sub-agent SECOND
   - Task sub-agent reads meeting_id from shared state
   - Creates a linked task

4. Coordinator returns final combined response to user
    - It passes through shared state — a dictionary that both agents can read and write to. Sub-agent 1 writes meeting_id to state, sub-agent 2 reads it.

Key: Sub-agents communicate through SHARED STATE, not directly with each other.

## API Endpoint vs Cloud Run

Local run:
- Agent runs only on YOUR machine
- No one else can access it
- Stops when you close your laptop
- Useless for demos, useless for judges

API Endpoint:
- A URL that accepts requests and returns responses
- Example: POST https://myagent.com/run → returns agent output

Cloud Run:
- Google's serverless platform that hosts your code as a URL
- Auto-scales (handles 1 request or 1000 requests)
- Always on, accessible from anywhere
- Your agent becomes a real API endpoint
- Judges can call it, test it, evaluate it live

Why Cloud Run for hackathon:
- Live demo via URL = credibility
- No server management
- Free tier covers hackathon usage

Stateless means: each request to Cloud Run starts fresh — it remembers nothing from the previous request.

So if a user says "Schedule a meeting" and then says "Create a task for it" — Cloud Run doesn't remember the meeting from the first request.

## Cloud Run - Stateless Warning
- Cloud Run does NOT remember previous requests
- Each request is independent
- For our hackathon project:
  - Store task/session data in BigQuery or Firestore
  - Pass context in every request payload
  - Never rely on in-memory state between requests

  ## GCP APIs
- All GCP services are DISABLED by default
- You enable only what your project needs
- This lab needs: Cloud Run, Cloud Build, Gemini API
- Command: gcloud services enable <service-name>

Introducing the APIs
Cloud Run Admin API (run.googleapis.com) allows you to run frontend and backend services, batch jobs, or websites in a fully managed environment. It handles the infrastructure for deploying and scaling your containerized applications.
Artifact Registry API (artifactregistry.googleapis.com) provides a secure, private repository to store your container images. It is the evolution of Container Registry and integrates seamlessly with Cloud Run and Cloud Build.
Cloud Build API (cloudbuild.googleapis.com) is a serverless CI/CD platform that executes your builds on Google Cloud infrastructure. It is used to build your container image in the cloud from your Dockerfile.
Vertex AI API (aiplatform.googleapis.com) enables your deployed application to communicate with Gemini models to perform core AI tasks. It provides the unified API for all of Google Cloud's AI services.
Compute Engine API (compute.googleapis.com) provides secure and customizable virtual machines that run on Google's infrastructure. While Cloud Run is managed, the Compute Engine API is often required as a foundational dependency for various networking and compute resources.

## ADK Agent Structure (Zoo Example → maps to our project)

root_agent = coordinator (entry point, receives user input, delegates)
SequentialAgent = orchestrator (runs sub-agents in fixed order, no LLM)
sub-agents = specialists (each does one job)
output_key = saves agent output to shared state for next agent to read

Our hackathon mapping:
root_agent → coordinator agent (reads user input, routes to sub-agents)
SequentialAgent → workflow orchestrator
sub-agents → calendar agent, task agent, email agent
shared state → passes data between agents (e.g. meeting_id)

add_prompt_to_state 📝: This tool remembers what a zoo visitor asks. When a visitor asks, "Where are the lions?", this tool saves that specific question into the agent's memory so the other agents in the workflow know what to research.
How: It's a Python function that writes the visitor's prompt into the shared tool_context.state dictionary. This tool context represents the agent's short-term memory for a single conversation. Data saved to the state by one agent can be read by the next agent in the workflow.
LangchainTool 🌍: This gives the tour guide agent general world knowledge. When a visitor asks a question that isn't in the zoo's database, like "What do lions eat in the wild?", this tool lets the agent look up the answer on Wikipedia.
How: It acts as an adapter, allowing our agent to use the pre-built WikipediaQueryRun tool from the LangChain library.

he comprehensive_researcher agent is the "brain" of our operation. It takes the user's prompt from the shared State, examines it's the Wikipedia Tool, and decides which ones to use to find the answer.
The response_formatter agent's role is presentation. It takes the raw data gathered by the Researcher agent (passed via the State) and uses the LLM's language skills to transform it into a friendly, conversational response.

The workflow agent acts as the ‘back-office' manager for the zoo tour. It takes the research request and ensures the two agents we defined above perform their jobs in the correct order: first research, then formatting. This creates a predictable and reliable process for answering a visitor's question.
How: It's a SequentialAgent, a special type of agent that doesn't think for itself. Its only job is to run a list of sub_agents (the researcher and formatter) in a fixed sequence, automatically passing the shared memory from one to the next.

## Common ADK Mistakes to avoid
- load_dotenv() must be called before os.getenv() or variables return None
- SequentialAgent has NO model — it only orchestrates order
- Shared state { KEY } in instructions is how agents communicate
- root_agent must be named exactly "root_agent" — ADK looks for this name

## Service Account - Why it matters
- Cloud Run runs without you being logged in
- It needs its own identity to call GCP APIs (Gemini, BigQuery etc.)
- Service account = identity for your cloud service
- Principle of least privilege: give it ONLY the roles it needs
- In our hackathon: Cloud Run service needs roles/aiplatform.user to call Gemini

## Deployment Flow (what actually happened)
Step 1: ADK CLI packaged our code into a Docker image using a Dockerfile it created automatically
Step 2: Docker image was pushed to Artifact Registry (the private container store we enabled)
Step 3: Cloud Run pulled the image from Artifact Registry and deployed it as a live HTTPS service
Result: Our agent is now accessible via a public URL, auto-scales, always on

## ADK Agent Trace - What I learned
- Each user message triggers the full workflow from scratch
- You can see every tool call in the Trace panel (left side)
- This is how you debug agents in production
- In our hackathon project: use trace to verify coordinator is routing correctly

## Codelab 1 - Direct takeaways for hackathon
1. ADK Agent structure: root_agent + SequentialAgent + sub_agents pattern
   → We use this exact pattern for coordinator + calendar/task/email agents
2. Cloud Run deployment: adk deploy cloud_run command with service account + IAM
   → Our hackathon project deploys the same way
3. Shared state: output_key passes data between agents via state dictionary
   → Our agents pass meeting_id, task_id through shared state