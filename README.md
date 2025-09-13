# 📚 Source Code Documentation

## Overview

This document provides comprehensive documentation for the Market Research Agent source code, including detailed explanations of each module, class, and function.

## 📁 Project Structure

```
MarketResearchAgent/
├── agent.py                 # Research agent implementation
├── supervisor_agent.py      # Multi-agent coordination
├── final_report.py         # Report generation system
├── reserach_brief.py       # Research brief creation
├── tools.py                # Search and utility tools
├── state.py                # State management
├── schema.py               # Data schemas
├── prompts.py              # AI prompts and templates
├── utils_agent.py          # Agent utilities
├── utils_display.py        # Display utilities
├── streamlit_app.py        # Streamlit web UI
├── run_streamlit.py        # UI launcher
├── run_agent.py            # CLI launcher
└── docs/                   # Documentation
    └── SOURCE_CODE_DOCUMENTATION.md
```

---

## 🔧 Core Modules

### 1. `agent.py` - Research Agent Implementation

**Purpose**: Implements the core research agent that conducts web research using Tavily search API.

#### Key Components:

##### Configuration
```python
# Tools and model setup
tools = [tavily_search, think_tool]
tools_by_name = {tool.name: tool for tool in tools}

# Model initialization
model = init_chat_model(model="gemini-2.5-flash", model_provider="google-genai")
model_with_tools = model.bind_tools(tools)
compress_model = init_chat_model(model="gemini-2.5-flash", model_provider="google-genai", max_tokens=32000)
```

##### Functions:

**`llm_call(state: ResearcherState)`**
- **Purpose**: Analyzes current state and decides on next actions
- **Parameters**: 
  - `state`: Current researcher state with messages and metadata
- **Returns**: Updated state with model's response
- **Behavior**: 
  - Uses research agent prompt to guide decision-making
  - Determines whether to call tools or provide final answer

**`tool_node(state: ResearcherState)`**
- **Purpose**: Executes all tool calls from the previous LLM response
- **Parameters**: 
  - `state`: Current researcher state
- **Returns**: Updated state with tool execution results
- **Behavior**:
  - Extracts tool calls from the last message
  - Executes each tool call using the appropriate tool
  - Creates ToolMessage outputs for each result

**`compress_research(state: ResearcherState) -> dict`**
- **Purpose**: Compresses research findings into a concise summary
- **Parameters**: 
  - `state`: Current researcher state with all research messages
- **Returns**: Dictionary with compressed research and raw notes
- **Behavior**:
  - Uses compression model to summarize findings
  - Extracts raw notes from tool and AI messages
  - Creates structured output for supervisor consumption

**`should_continue(state: ResearcherState) -> Literal["tool_node", "compress_research"]`**
- **Purpose**: Determines whether to continue research or provide final answer
- **Parameters**: 
  - `state`: Current researcher state
- **Returns**: Next step in the workflow
- **Logic**: 
  - If LLM made tool calls → continue to tool execution
  - Otherwise → compress research and end

##### Graph Construction:
```python
# Build the agent workflow
agent_builder = StateGraph(ResearcherState, output_schema=ResearcherOutputState)

# Add nodes
agent_builder.add_node("llm_call", llm_call)
agent_builder.add_node("tool_node", tool_node)
agent_builder.add_node("compress_research", compress_research)

# Add edges
agent_builder.add_edge(START, "llm_call")
agent_builder.add_conditional_edges("llm_call", should_continue, {...})
agent_builder.add_edge("tool_node", "llm_call")
agent_builder.add_edge("compress_research", END)

# Compile
researcher_agent = agent_builder.compile()
```

---

### 2. `supervisor_agent.py` - Multi-Agent Coordination

**Purpose**: Orchestrates the entire research workflow by coordinating multiple research agents.

#### Key Components:

##### Configuration
```python
supervisor_tools = [ConductResearch, ResearchComplete, think_tool]
supervisor_model = init_chat_model(model="gemini-2.5-flash", model_provider="google-genai")
supervisor_model_with_tools = supervisor_model.bind_tools(supervisor_tools)

# System constants
max_researcher_iterations = 6
max_concurrent_researchers = 2
```

##### Functions:

**`supervisor(state: SupervisorState) -> Command[Literal["supervisor_tools"]]`**
- **Purpose**: Coordinates research activities and makes strategic decisions
- **Parameters**: 
  - `state`: Current supervisor state with messages and progress
- **Returns**: Command to proceed to supervisor_tools node
- **Behavior**:
  - Analyzes research brief and current progress
  - Decides what research topics need investigation
  - Determines whether to conduct parallel research
  - Uses lead researcher prompt for guidance

**`supervisor_tools(state: SupervisorState) -> Command[Literal["supervisor", "__end__"]]`**
- **Purpose**: Executes supervisor decisions and manages research execution
- **Parameters**: 
  - `state`: Current supervisor state
- **Returns**: Command to continue supervision or end process
- **Behavior**:
  - Executes think_tool calls for strategic reflection
  - Launches parallel research agents for different topics
  - Aggregates research results from sub-agents
  - Determines when research is complete
  - Handles error conditions gracefully

##### Graph Construction:
```python
# Build supervisor graph
supervisor_builder = StateGraph(SupervisorState)
supervisor_builder.add_node("supervisor", supervisor)
supervisor_builder.add_node("supervisor_tools", supervisor_tools)
supervisor_builder.add_edge(START, "supervisor")

supervisor_agent = supervisor_builder.compile()
```

---

### 3. `final_report.py` - Report Generation System

**Purpose**: Synthesizes all research findings into comprehensive final reports.

#### Key Components:

##### Configuration
```python
writer_model = init_chat_model(model="gemini-2.5-flash", model_provider="google-genai")
```

##### Functions:

**`final_report_generation(state: AgentState)`**
- **Purpose**: Generates the final comprehensive research report
- **Parameters**: 
  - `state`: Agent state with all research findings
- **Returns**: Updated state with final report
- **Behavior**:
  - Combines all research notes into findings string
  - Uses final report generation prompt
  - Generates executive-ready report
  - Updates state with final report content

##### Graph Construction:
```python
# Build the overall workflow
deep_researcher_builder = StateGraph(AgentState, input_schema=AgentInputState)

# Add workflow nodes
deep_researcher_builder.add_node("clarify_with_user", clarify_with_user)
deep_researcher_builder.add_node("write_research_brief", write_research_brief)
deep_researcher_builder.add_node("supervisor_subgraph", supervisor_agent)
deep_researcher_builder.add_node("final_report_generation", final_report_generation)

# Add workflow edges
deep_researcher_builder.add_edge(START, "clarify_with_user")
deep_researcher_builder.add_edge("write_research_brief", "supervisor_subgraph")
deep_researcher_builder.add_edge("supervisor_subgraph", "final_report_generation")
deep_researcher_builder.add_edge("final_report_generation", END)

# Compile with checkpointing
checkpointer = InMemorySaver()
full_agent = deep_researcher_builder.compile(checkpointer=checkpointer)
```

---

### 4. `reserach_brief.py` - Research Brief Creation

**Purpose**: Transforms user queries into structured research briefs and handles clarification.

#### Key Components:

##### Configuration
```python
model = init_chat_model(model="gemini-2.5-flash", model_provider="google-genai")
```

##### Functions:

**`clarify_with_user(state: AgentState) -> Command[Literal["write_research_brief", "__end__"]]`**
- **Purpose**: Determines if user's request contains sufficient information
- **Parameters**: 
  - `state`: Agent state with user messages
- **Returns**: Command to proceed or end with clarification
- **Behavior**:
  - Uses structured output to make deterministic decisions
  - Routes to research brief generation if sufficient info
  - Ends with clarification question if more info needed
  - Avoids hallucination through structured output

**`write_research_brief(state: AgentState)`**
- **Purpose**: Transforms conversation history into comprehensive research brief
- **Parameters**: 
  - `state`: Agent state with conversation history
- **Returns**: Updated state with research brief
- **Behavior**:
  - Uses structured output for consistent format
  - Generates detailed research brief from conversation
  - Updates state with brief and supervisor messages

---

### 5. `tools.py` - Search and Utility Tools

**Purpose**: Provides tools for web search, strategic thinking, and research coordination.

#### Key Components:

##### Research Tools:

**`tavily_search(query: str, max_results: int = 3, topic: str = "general") -> str`**
- **Purpose**: Fetches results from Tavily search API with content summarization
- **Parameters**:
  - `query`: Search query string
  - `max_results`: Maximum number of results (default: 3)
  - `topic`: Topic filter ("general", "news", "finance")
- **Returns**: Formatted search results with summaries
- **Behavior**:
  - Executes search using Tavily API
  - Deduplicates results by URL
  - Processes results with summarization
  - Formats output for consumption

**`think_tool(reflection: str) -> str`**
- **Purpose**: Tool for strategic reflection on research progress
- **Parameters**:
  - `reflection`: Detailed reflection on research progress
- **Returns**: Confirmation of recorded reflection
- **Usage**: Used after each search to analyze results and plan next steps

##### Supervisor Tools:

**`ConductResearch` (Pydantic Model)**
- **Purpose**: Tool for delegating research tasks to specialized sub-agents
- **Fields**:
  - `research_topic`: Detailed topic description (at least a paragraph)

**`ResearchComplete` (Pydantic Model)**
- **Purpose**: Tool for indicating research process completion
- **Fields**: None (empty model)

##### Specialized Tools:

**`IndustryResearch`, `UseCaseGeneration`, `ResourceCollection`, `FinalProposal`**
- **Purpose**: Specialized tools for different research phases
- **Usage**: Available for future expansion of research capabilities

**`search_datasets(query: str, platforms: list[str]) -> str`**
- **Purpose**: Search for datasets on specific platforms
- **Parameters**:
  - `query`: Search query for datasets
  - `platforms`: List of platforms ("kaggle", "huggingface", "github")
- **Returns**: Formatted dataset search results
- **Behavior**:
  - Constructs platform-specific search queries
  - Uses Tavily search for each platform
  - Processes and formats results

---

### 6. `state.py` - State Management

**Purpose**: Defines state objects used throughout the multi-agent workflow.

#### Key Classes:

**`ResearcherState(TypedDict)`**
- **Purpose**: State for individual research agents
- **Fields**:
  - `researcher_messages`: Message history with add_messages annotation
  - `tool_call_iterations`: Iteration counter for limiting tool calls
  - `research_topic`: Current research topic being investigated
  - `compressed_research`: Compressed summary of findings
  - `raw_notes`: Raw research notes for detailed analysis

**`ResearcherOutputState(TypedDict)`**
- **Purpose**: Output state for research agents
- **Fields**:
  - `compressed_research`: Final compressed research findings
  - `raw_notes`: All raw notes from research process
  - `researcher_messages`: Complete message history

**`SupervisorState(TypedDict)`**
- **Purpose**: State for multi-agent research supervisor
- **Fields**:
  - `supervisor_messages`: Messages for coordination and decision-making
  - `research_brief`: Detailed research brief guiding research direction
  - `notes`: Processed and structured notes for report generation
  - `research_iterations`: Counter tracking research iterations
  - `raw_notes`: Raw unprocessed research notes from sub-agents

**`AgentState(MessagesState)`**
- **Purpose**: Main state for full multi-agent research system
- **Fields**:
  - `research_brief`: Research brief from user conversation
  - `supervisor_messages`: Messages exchanged with supervisor
  - `raw_notes`: Raw research notes collected during research
  - `notes`: Processed notes ready for report generation
  - `final_report`: Final formatted research report

**`AgentInputState(MessagesState)`**
- **Purpose**: Input state containing only user messages
- **Fields**: Inherits from MessagesState (messages only)

---

### 7. `schema.py` - Data Schemas

**Purpose**: Defines Pydantic models for structured output and data validation.

#### Key Classes:

**`ClarifyWithUser(BaseModel)`**
- **Purpose**: Schema for user clarification decisions
- **Fields**:
  - `need_clarification`: Boolean indicating if clarification is needed
  - `question`: Question to ask user for clarification
  - `verification`: Verification message for proceeding with research

**`ResearchQuestion(BaseModel)`**
- **Purpose**: Schema for research brief generation
- **Fields**:
  - `research_brief`: Comprehensive research question/objective

**`Summary(BaseModel)`**
- **Purpose**: Schema for webpage content summarization
- **Fields**:
  - `summary`: Concise summary of webpage content
  - `key_excerpts`: Important quotes and excerpts from content

---

### 8. `prompts.py` - AI Prompts and Templates

**Purpose**: Contains all AI prompts and templates used throughout the system.

#### Key Prompts:

**`research_agent_prompt`**
- **Purpose**: Guides research agent behavior and decision-making
- **Content**: 
  - Task definition and available tools
  - Instructions for research methodology
  - Hard limits for tool call budgets
  - Guidelines for strategic thinking

**`lead_researcher_prompt`**
- **Purpose**: Guides supervisor agent coordination
- **Content**:
  - Research coordination instructions
  - Parallel research management
  - Quality control guidelines

**`final_report_generation_prompt`**
- **Purpose**: Guides final report synthesis
- **Content**:
  - Report structure requirements
  - Formatting guidelines
  - Content organization instructions

**`clarify_with_user_instructions`**
- **Purpose**: Guides clarification decision-making
- **Content**:
  - Criteria for determining if clarification is needed
  - Guidelines for generating clarification questions

**`transform_messages_into_research_topic_prompt`**
- **Purpose**: Guides research brief generation
- **Content**:
  - Instructions for extracting research objectives
  - Guidelines for comprehensive brief creation

---

### 9. `utils_agent.py` - Agent Utilities

**Purpose**: Provides utility functions for agent operations and search functionality.

#### Key Functions:

**`get_today_str() -> str`**
- **Purpose**: Get current date in human-readable format
- **Returns**: Formatted date string

**`get_current_dir() -> Path`**
- **Purpose**: Get current directory (Jupyter notebook compatible)
- **Returns**: Path object representing current directory

**`tavily_search_multiple(search_queries: List[str], ...) -> List[dict]`**
- **Purpose**: Perform search using Tavily API for multiple queries
- **Parameters**:
  - `search_queries`: List of search queries
  - `max_results`: Maximum results per query
  - `topic`: Topic filter
  - `include_raw_content`: Whether to include raw content
- **Returns**: List of search result dictionaries

**`deduplicate_search_results(results: List[dict]) -> List[dict]`**
- **Purpose**: Remove duplicate search results by URL
- **Parameters**: List of search results
- **Returns**: Deduplicated list of results

**`process_search_results(results: List[dict]) -> List[dict]`**
- **Purpose**: Process and summarize search results
- **Parameters**: List of search results
- **Returns**: Processed results with summaries

**`format_search_output(results: List[dict]) -> str`**
- **Purpose**: Format search results for display
- **Parameters**: List of processed results
- **Returns**: Formatted string output

**`get_notes_from_tool_calls(messages: List[BaseMessage]) -> List[str]`**
- **Purpose**: Extract notes from tool call messages
- **Parameters**: List of messages
- **Returns**: List of extracted notes

---

### 10. `utils_display.py` - Display Utilities

**Purpose**: Provides utilities for formatting and displaying messages in the CLI.

#### Key Functions:

**`format_message_content(message)`**
- **Purpose**: Convert message content to displayable string
- **Parameters**: Message object
- **Returns**: Formatted string content
- **Behavior**:
  - Handles different message content types
  - Formats tool calls with proper indentation
  - Processes complex content structures

**`format_messages(messages)`**
- **Purpose**: Format and display a list of messages with Rich formatting
- **Parameters**: List of messages
- **Behavior**:
  - Uses Rich library for beautiful console output
  - Different styling for different message types
  - Color-coded panels for easy reading

---

### 11. `streamlit_app.py` - Streamlit Web UI

**Purpose**: Provides a modern web interface for the market research agent.

#### Key Components:

##### Configuration
```python
st.set_page_config(
    page_title="Market Research & AI Use Case Generator",
    page_icon="🔍",
    layout="wide"
)
```

##### Functions:

**`extract_clickable_links(text: str) -> str`**
- **Purpose**: Convert URLs in text to clickable markdown links
- **Parameters**: Text containing URLs
- **Returns**: Text with clickable HTML links

**`run_research(query: str)`**
- **Purpose**: Run the research process with progress tracking
- **Parameters**: Research query string
- **Returns**: Research results
- **Behavior**:
  - Updates session state for progress tracking
  - Runs full agent workflow
  - Handles errors gracefully

**`main()`**
- **Purpose**: Main Streamlit application function
- **Behavior**:
  - Renders header and layout
  - Manages session state
  - Handles user interactions
  - Displays results and progress

##### UI Layout:
- **Header**: Branded header with gradient background
- **Left Column (1/4)**: Input panel with text area and buttons
- **Right Column (3/4)**: Output panel with scrollable report display

---

### 12. `run_streamlit.py` - UI Launcher

**Purpose**: Launcher script for the Streamlit application.

#### Key Functions:

**`check_dependencies()`**
- **Purpose**: Check if required dependencies are installed
- **Returns**: Boolean indicating if dependencies are available

**`launch_streamlit()`**
- **Purpose**: Launch the Streamlit application
- **Behavior**:
  - Checks dependencies
  - Launches Streamlit with proper configuration
  - Handles user interruption gracefully

---

### 13. `run_agent.py` - CLI Launcher

**Purpose**: Command-line interface launcher for the research agent.

#### Key Components:

**`main()`**
- **Purpose**: Main CLI application function
- **Behavior**:
  - Sets up console output
  - Executes market research process
  - Displays results with Rich formatting
  - Handles async execution

---

## 🔄 Data Flow

### 1. User Input Processing
```
User Query → clarify_with_user → write_research_brief → research_brief
```

### 2. Research Coordination
```
research_brief → supervisor → supervisor_tools → parallel research agents
```

### 3. Research Execution
```
research_topic → researcher_agent → tavily_search → think_tool → compress_research
```

### 4. Report Generation
```
compressed_research → final_report_generation → final_report
```

## 🛠️ Configuration

### Environment Variables
- `TAVILY_API_KEY`: Required for web search functionality
- `GOOGLE_API_KEY`: Required for Google Gemini models

### System Constants
- `max_researcher_iterations`: Maximum research iterations (default: 6)
- `max_concurrent_researchers`: Maximum parallel researchers (default: 2)

## 🔧 Error Handling

### Agent Level
- Graceful handling of API failures
- Progress preservation during errors
- User feedback for error conditions

### System Level
- Checkpointing for state recovery
- Exception handling in supervisor tools
- Fallback mechanisms for critical failures

## 📊 Performance Considerations

### Optimization Strategies
- Parallel research execution
- Result caching and deduplication
- Efficient state management
- Async processing for I/O operations

### Resource Management
- Tool call budgets to prevent infinite loops
- Memory management for large research outputs
- API rate limiting considerations

---

This documentation provides a comprehensive overview of the source code structure, functionality, and implementation details. Each module is designed to work together seamlessly to provide a robust market research and AI use case generation system.
