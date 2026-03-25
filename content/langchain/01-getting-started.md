## Why Valkey for LangGraph?

LangGraph agents are stateless by default — every invocation starts fresh. `ValkeySaver` adds persistent checkpointing backed by Valkey:

  * **Sub-millisecond reads** — checkpoint retrieval in ~0.1ms
  * **Atomic writes** — no partial state corruption
  * **Built-in TTL** — sessions auto-expire, no cleanup jobs
  * **ElastiCache ready** — same code works locally and in production

## Prerequisites

  * Docker installed
  * Python 3.10+

## Step 1: Start Valkey
    
    
    docker run -d --name valkey -p 6379:6379 valkey/valkey-bundle:latest

The `valkey-bundle` image includes JSON and Search modules needed for ValkeyStore. Verify:
    
    
    docker exec valkey valkey-cli PING
    # PONG

## Step 2: Install the Package
    
    
    pip install 'langgraph-checkpoint-aws[valkey]' langchain-aws

This installs `ValkeySaver`, `ValkeyStore`, `ValkeyCache`, and the Bedrock integrations.

## Step 3: Understand the Data Model

`ValkeySaver` stores each checkpoint as a JSON document in Valkey:
    
    
    # Key format: checkpoint:{thread_id}:{checkpoint_ns}:{checkpoint_id}
    # Each checkpoint contains:
    {
        "v": 1,
        "ts": "2026-03-12T10:00:00+00:00",
        "channel_values": {"messages": [...]},
        "channel_versions": {"__start__": 2, "messages": 3},
        "versions_seen": {...}
    }

**Under the Hood:** `ValkeySaver` uses Valkey JSON (`JSON.SET`) for structured storage and RediSearch (`FT.CREATE`, `FT.SEARCH`) for indexing checkpoints by thread ID and namespace. TTL is applied via `EXPIRE`.

## Step 4: Persist a LangGraph Agent
    
    
    from langgraph.graph import StateGraph, MessagesState
    from langgraph_checkpoint_aws import ValkeySaver
    from langchain_aws import ChatBedrockConverse
    from langchain_core.messages import HumanMessage
    
    
    # 1. Create a simple chatbot graph
    model = ChatBedrockConverse(
        model="us.anthropic.claude-sonnet-4-20250514-v1:0",
        region_name="us-west-2",
    )
    
    def chatbot(state: MessagesState):
        return {"messages": [model.invoke(state["messages"])]}
    
    builder = StateGraph(MessagesState)
    builder.add_node("chatbot", chatbot)
    builder.set_entry_point("chatbot")
    builder.set_finish_point("chatbot")
    
    # 2. Compile with ValkeySaver — this is the key line
    with ValkeySaver.from_conn_string(
        "valkey://localhost:6379",
        ttl_seconds=3600,
    ) as checkpointer:
        graph = builder.compile(checkpointer=checkpointer)
    
        # 3. Invoke with a thread ID
        config = {"configurable": {"thread_id": "session-1"}}
        result = graph.invoke(
            {"messages": [HumanMessage(content="What is Valkey?")]},
            config,
        )
        print(result["messages"][-1].content)
    
        # 4. Continue the conversation — state is persisted!
        result = graph.invoke(
            {"messages": [HumanMessage(content="How fast is it?")]},
            config,
        )
        # The agent remembers the previous message about Valkey
        print(result["messages"][-1].content)

## Step 5: Verify Persistence

Stop and restart your script — the conversation state is still in Valkey:
    
    
    # In a NEW Python process:
    with ValkeySaver.from_conn_string("valkey://localhost:6379") as checkpointer:
        graph = builder.compile(checkpointer=checkpointer)
        config = {"configurable": {"thread_id": "session-1"}}
    
        # List checkpoints for this thread
        for cp in checkpointer.list(config):
            print(cp.metadata)  # Shows previous conversation

**Valkey Commands Fired:**
    
    
    JSON.SET checkpoint:session-1:__empty__:abc123 $ '{...}'
    EXPIRE checkpoint:session-1:__empty__:abc123 3600
    FT.SEARCH checkpoints_idx '@thread_id:{session-1}' SORTBY ts DESC LIMIT 0 1

## Next Steps

You now have a LangGraph agent with persistent state. Next, let's add LLM response caching to cut costs and latency.

[Next: 02 LLM Response Caching →](<02-llm-caching.html>)