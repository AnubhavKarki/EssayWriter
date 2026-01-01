# LangGraph Essay Writer

A multi-stage AI essay writing system built with LangGraph. Generates high-quality 5-paragraph essays through planning, research, iterative writing, and self-critique.

## Features

- **Structured workflow**: Planning → Research → Drafting → Reflection → Revision
- **Iterative improvement**: Up to configurable maximum revisions with critique
- **Web research integration**: Tavily-powered search for factual content
- **Structured outputs**: Pydantic models ensure consistent query generation
- **Checkpointing**: SQLite memory persistence for thread state
- **LangSmith tracing**: Full observability of agent execution

## Requirements

Install the required dependencies:

```bash
pip install -q langchain langchain_openai langchain_core langgraph tavily-python pydantic langsmith langgraph-checkpoint-sqlite
```

Set up your environment variables:

```bash
export OPENAI_API_KEY="your_openai_key"
export TAVILY_API_KEY="your_tavily_key"
export LANGSMITH_API_KEY="your_langsmith_key"
export LANGCHAIN_PROJECT="essay-writer"
export LANGSMITH_TRACING="true"
export LANGCHAIN_TRACING_V2="true"
```

## Quick Start

```python
from essay_writer import create_essay_writer

# Initialize the essay writing graph
graph = create_essay_writer()

# Generate essay with 2 maximum revisions
config = {'configurable': {'thread_id': 'essay-1'}}
prompt = {
    'task': 'Explain the impact of quantum computing on cryptography',
    'max_revisions': 2,
    'revision_number': 1
}

# Stream the essay generation process
for event in graph.stream(prompt, config):
    print(event)
```

## LangSmith Observability

**LangSmith integration is pre-configured with project name `"essay-writer"` for full trace visibility.**

All runs are automatically traced to [LangSmith](https://smith.langchain.com/) with:

- **Project**: `essay-writer`
- **Full execution traces**: Every node, edge, and LLM call
- **Input/output inspection**: View prompts, research content, drafts, critiques
- **Performance metrics**: Token usage, latency per stage
- **Debugging**: Step through iterations, inspect state changes

**Access traces at**: https://smith.langchain.com/o/{your-org}/projects/p/essay-writer

**Key trace features**:
- Filter by `thread_id` to debug specific essays
- Compare draft quality across revisions
- Analyze research query effectiveness
- Monitor cost per essay generation

```bash
# View traces in terminal
export LANGSMITH_TRACING="true"
python essay_writer.py  # All runs logged automatically
```

## Architecture

```
Task → Planner → Research Plan → Writer (Draft 1)
                    ↓
              Reflection → Research Critique → Writer (Draft 2)
                    ↓
              [Max Revisions] → Final Essay
```

### Workflow Stages

1. **Planner**: Creates high-level essay outline
2. **Research Plan**: Generates 3 targeted search queries
3. **Writer**: Produces essay draft using research + outline
4. **Reflection**: Provides detailed critique and recommendations
5. **Research Critique**: Additional research based on reflection
6. **Repeat**: Up to `max_revisions` iterations

## AgentState Schema

```python
class AgentState(TypedDict):
    task: str                    # Original essay topic
    plan: str                    # High-level outline
    draft: str                   # Current essay draft
    critique: str                # Teacher-style feedback
    content: List[str]           # Researched content
    revision_number: int         # Current iteration
    max_revisions: int           # Maximum iterations
```

## Customization

### Adjust Revision Count

```python
prompt = {
    'task': 'Your essay topic here',
    'max_revisions': 3,      # Increase for more iterations
    'revision_number': 1
}
```

### Custom Model

```python
model = ChatOpenAI(model='gpt-4o', temperature=0.1)  # Higher quality model
```

### Modify Prompts

Edit the prompt constants (`PLAN_PROMPT`, `WRITER_PROMPT`, etc.) to adjust tone, length, or style requirements.

## Core Components

### Research Queries (Pydantic)

```python
class Queries(BaseModel):
    queries: List[str]  # Max 3 queries per research step
```

### Tavily Integration

```python
tavily = TavilyClient()
response = tavily.search(query=q, max_results=2)
```

### Checkpointing

```python
from langgraph.checkpoint.memory import MemorySaver
memory = MemorySaver()  # Or SqliteSaver.from_conn_string(':memory:')
graph = builder.compile(checkpointer=memory)
```

## Usage Examples

### Single Essay

```python
config = {'configurable': {'thread_id': 'essay-1'}}
result = graph.invoke(prompt, config)
print(result['draft'])  # Final essay
```

### Streaming Progress

```python
for event in graph.stream(prompt, config):
    if 'draft' in event:
        print("New draft:", event['draft'][:200], "...")
    elif 'critique' in event:
        print("Critique:", event['critique'][:200], "...")
```

### Resume Interrupted Session

```python
# Same thread_id resumes from last checkpoint
config = {'configurable': {'thread_id': 'essay-1'}}
graph.stream(prompt, config)
```

## Output Format

The final `draft` field contains the complete essay. Intermediate states include:

- `plan`: Detailed outline
- `content`: Raw research results
- `critique`: Improvement suggestions per iteration

## Error Handling

- **Research failures**: System continues without additional content
- **Max revisions exceeded**: Returns best available draft
- **Model errors**: Retries handled by LangGraph

## Production Deployment

```bash
# Export graph configuration
python -m langgraph export essay_writer.json

# Deploy with persistence
graph = builder.compile(
    checkpointer=SqliteSaver.from_conn_string('essay_writer.db')
)
```

## Relevant docs:

- [LangGraph StateGraph](https://docs.langchain.com/oss/langgraph/)
- [LangSmith Tracing](https://docs.smith.langchain.com/tracing)
- [Tavily Integration](https://docs.langchain.com/tools/tavily_search)
- [Structured Output](https://docs.langchain.com/how_to/structured_output/)
- [Checkpointing](https://docs.langchain.com/oss/langgraph/persistence/)
