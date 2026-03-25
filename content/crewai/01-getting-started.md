## Why Valkey for CrewAI?

CrewAI agents need persistent memory to learn across executions. Valkey is ideal because:

  * **Sub-millisecond reads** — memory recall in ~0.1ms via GLIDE
  * **Vector search** — `FT.SEARCH` with HNSW for semantic recall
  * **JSON storage** — `JSON.SET` stores structured memory records natively
  * **TTL** — memories auto-expire with `EXPIRE`

## Prerequisites

  * Docker installed
  * Python 3.10+
  * AWS credentials for Bedrock (for later guides)

## Step 1: Start Valkey
    
    
    docker run -d --name valkey -p 6379:6379 valkey/valkey-bundle:latest

The `valkey-bundle` image includes JSON and Search modules. Verify:
    
    
    docker exec valkey valkey-cli PING
    # PONG

## Step 2: Install Dependencies
    
    
    pip install crewai valkey-glide boto3

**Why GLIDE?** Valkey GLIDE is the official Valkey client with a Rust core for high performance. It supports both standalone Valkey and ElastiCache for Valkey clusters. Unlike `redis-py`, GLIDE is purpose-built for Valkey and supports all Valkey-specific features including `FT.*` search commands.

## Step 3: Connection Configuration
    
    
    from dataclasses import dataclass
    import os
    
    @dataclass(frozen=True)
    class ValkeyConfig:
        host: str
        port: int
        use_tls: bool
        password: str | None
    
    def get_valkey_config() -> ValkeyConfig:
        return ValkeyConfig(
            host=os.environ.get("VALKEY_HOST", "localhost"),
            port=int(os.environ.get("VALKEY_PORT", "6379")),
            use_tls=os.environ.get("VALKEY_TLS", "false").lower() in ("true", "1"),
            password=os.environ.get("VALKEY_PASSWORD") or None,
        )

## Step 4: Connect and Test
    
    
    import asyncio
    from glide import GlideClient, GlideClientConfiguration, NodeAddress
    
    async def test_connection():
        config = get_valkey_config()
        glide_config = GlideClientConfiguration(
            [NodeAddress(config.host, config.port)],
            use_tls=config.use_tls,
        )
        client = await GlideClient.create(glide_config)
    
        # Test basic operations
        await client.set("test:hello", "world")
        val = await client.get("test:hello")
        print(f"✅ Connected! Got: {val.decode()}")
    
        # Clean up
        await client.delete(["test:hello"])
    
    asyncio.run(test_connection())
    # ✅ Connected! Got: world

**Valkey Commands Fired:**
    
    
    SET test:hello "world"
    GET test:hello
    DEL test:hello

## Next Steps

Connection works. Next, we'll build the `ValkeyStorage` backend that implements CrewAI's storage protocol.

[Next: 02 Memory Storage Backend →](<02-memory-storage.html>)