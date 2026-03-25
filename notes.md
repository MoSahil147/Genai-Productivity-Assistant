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