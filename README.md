# 🧬🌍 GenWorlds — Collaborative AI Agent Framework

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](./LICENSE) [![Discord](https://dcbadge.vercel.app/api/server/VpfmXEMN66?compact=true&style=flat)](https://discord.gg/VpfmXEMN66)

Version: 0.0.18

GenWorlds is an event-based framework for building autonomous, goal-driven multi-agent systems. Agents interact asynchronously through a WebSocket-backed world, can manipulate shared objects, store memories, and call LLM-powered "thoughts" to fill event parameters.

Important: GenWorlds is under active development and uses LLM APIs — expect breaking changes and potential API usage costs.

Contents
- Quick links
- Install
- Environment & configuration
- Core concepts
- Quickstart (copy-paste examples)
- Running simulations & socket server
- Examples & tutorials
- Development, tests & docs
- Contributing, support & license
- Troubleshooting & tips

Quick links
- Docs site (detailed): https://genworlds.com/docs/get-started/intro
- Tutorials: docs/get-started/quickstart and docs/tutorials/
- Examples & use_cases: use_cases/
- Contributing: CONTRIBUTING.md
- License: MIT (LICENSE)

Requirements
- Python 3.10 or 3.11
- Typical dependencies (also see pyproject.toml / requirements.txt): fastapi, uvicorn, langchain, openai, qdrant-client, websockets, pydantic, tiktoken

Install

Stable release (PyPI)
pip install genworlds

From source (for development / latest)
git clone <repo>
cd genworlds
# using pip
pip install -r requirements.txt
# or editable install
pip install -e .

Or use Poetry (if you prefer)
poetry install

Environment variables
- Copy .env.example to .env and fill values
  - OPENAI_API_KEY — required for LLM-based agents / thoughts
  - LOGGING_LEVEL — DEBUG|INFO|WARNING|ERROR|CRITICAL

Example .env
OPENAI_API_KEY=sk-...
LOGGING_LEVEL=INFO

Core concepts (brief)
- World: the runtime environment that holds agents, objects and delivers events over a WebSocket.
- Agent: an autonomous, goal-driven entity that receives events, reasons (thoughts) and emits actions/events.
- Object: deterministic entities that expose Actions — useful for deterministic logic (e.g., calculators, stores).
- Event: the payload exchanged between participants that represents state changes or requests.
- Action: logic bound to an object that reacts to trigger events and usually emits another event.
- Thought: (LLM call) nondeterministic reasoning step to fill event parameters or make decisions.

Quickstart — minimal runnable examples

1) Launch a simple world and WebSocket server
```python
from genworlds.worlds.concrete.base.world import BaseWorld

hello_world = BaseWorld(
    name="Hello World",
    description="A world for testing the library."
)

# Launch the world (this starts the FastAPI WebSocket server)
hello_world.launch()
# By default uvicorn runs on http://127.0.0.1:7456
```

2) Add a basic assistant (LLM) agent
```python
import os
from dotenv import load_dotenv
from genworlds.agents.concrete.basic_assistant.utils import generate_basic_assistant
from genworlds.worlds.concrete.base.actions import UserSpeaksWithAgentEvent

load_dotenv()
OPENAI_API_KEY = os.getenv("OPENAI_API_KEY")

description = "Agent that helps the user to answer questions."

agent1 = generate_basic_assistant(
    agent_name="agent1",
    description=description,
    openai_api_key=OPENAI_API_KEY
)

# Add wakeup events so agent listens for a UserSpeaksWithAgentEvent
agent1.add_wakeup_event(event_class=UserSpeaksWithAgentEvent)

# Attach to the world
hello_world.add_agent(agent1)
```

3) Create a deterministic object with an action (Simple Calculator)
```python
from genworlds.events.abstracts.event import AbstractEvent
from genworlds.events.abstracts.action import AbstractAction
from genworlds.objects.abstracts.object import AbstractObject

class AgentWantsToAddTwoNumbers(AbstractEvent):
    event_type = "agent_wants_to_add_two_numbers_event"
    description = "Sends two float numbers to be added in a deterministic way."
    number1: float
    number2: float

class SendAddedNumbersToAgent(AbstractEvent):
    event_type = "send_added_numbers_to_agent_event"
    description = "Sends the deterministic sum back to the caller."
    sum: float

class AddTwoNumbers(AbstractAction):
    trigger_event_class = AgentWantsToAddTwoNumbers
    description = "Adds two floats deterministically."

    def __init__(self, host_object: AbstractObject):
        self.host_object = host_object

    def __call__(self, event: AgentWantsToAddTwoNumbers):
        ev = SendAddedNumbersToAgent(
            sender_id=self.host_object.id,
            target_id=event.sender_id,
            sum=event.number1 + event.number2
        )
        self.host_object.send_event(ev)

class SimpleCalculator(AbstractObject):
    def __init__(self, id: str):
        actions = [AddTwoNumbers(host_object=self)]
        super().__init__(
            name="Simple Calculator",
            id=id,
            description="Object used to do simple calculations.",
            actions=actions
        )

simple_calculator = SimpleCalculator(id="simple_calculator")
hello_world.add_object(simple_calculator)
```

4) Interact via a TestUser (send event to agent)
```python
from genworlds.utils.test_user import TestUser
from genworlds.worlds.concrete.base.actions import UserSpeaksWithAgentEvent

test_user = TestUser()

message = UserSpeaksWithAgentEvent(
    sender_id=test_user.id,
    message="Please add 2.45 and 6.78 and send the result.",
    target_id="agent1"
).json()

test_user.socket_client.send_message(message)
```

Running simulations & socket server

- Start the WebSocket server programmatically
  - Use the world.launch() method which starts the FastAPI + Uvicorn server in the main thread (quickstart examples use this).
  - Or use genworlds.simulation.sockets.server.start(host, port, ...) to start standalone.

- Start the server in a background thread
```python
from genworlds.simulation.sockets.server import start_thread
start_thread(host="127.0.0.1", port=7456)
```

- Launch a Simulation (collections of agents and objects)
```python
from genworlds.simulation.simulation import Simulation
from genworlds.simulation.utils.launch_simulation import launch_simulation

# assume `world`, `objects`, `agents` are prepared (see use_cases/world_setup.py)
simulation = Simulation(
    name=world.name,
    description=world.description,
    world=world,
    objects=objects,  # list of (object_instance, world_properties)
    agents=agents,    # list of (agent_instance, world_properties)
)
# Launch using helper (starts socket server + simulation)
launch_simulation(simulation)
```

Example use cases & configuration
- Roundtable / podcast-style world: use_cases/roundtable contains YAML-driven world setup (world_setup.py) and a CLI config example (roundtable_cli_config_example.json).
- RAG & custom agents: see use_cases/foundational_rag and use_cases/custom_qna_agent.
- Use world_definitions YAMLs to define worlds, objects, agents and memory backends. The world_setup loader demonstrates dynamic construction from YAML.

Docs & tutorials
- Full docs: https://genworlds.com/docs/get-started/intro
- Quickstart & tutorials: docs/get-started/quickstart.md and docs/tutorials/* (simple collaboration, foundational RAG, first custom agent)
- Local docs site:
  - cd docs
  - yarn
  - yarn start
  - yarn build

Development & testing
- Recommended: create a virtualenv (Python 3.10/3.11)
- Install dev deps: poetry install or pip install -r requirements.txt
- Tests: pytest is available (see pyproject.toml dev dependencies)
- Linting/formatting: black (configured in pyproject.toml / dev tools)

Contributing & Code of Conduct
- We welcome contributions — see CONTRIBUTING.md for guidelines.
- This project is evolving; breaking changes can occur. When contributing, please open issues/PRs against the repo and follow the contribution workflow in CONTRIBUTING.md.

Support & community
- Discord: https://discord.gg/VpfmXEMN66
- If you want enterprise support from Yeager.ai, the project includes a contact form (mentioned in original docs).

License
- MIT License — see LICENSE.

Troubleshooting & tips
- Missing OPENAI_API_KEY: LLM-based agents will fail without a valid key. Use .env or pass openai_api_key when constructing agents.
- WebSocket port conflicts: default port is 7456. If you see "Address already in use", pick another port or stop the process using the port.
- Long-running processes: world.launch and Simulation.launch block the calling thread. Use start_thread helper functions or run them in separate threads if you need an interactive REPL or UI.
- Logging: set LOGGING_LEVEL in .env to control verbosity.

Where to go next
- Read docs/get-started/intro.md and follow docs/get-started/quickstart.md for step-by-step guided tutorials.
- Explore use_cases/ for concrete, real-world examples.
- If you plan to build a UI, the websocket-based world server makes it easy to connect frontends like the GenWorlds-Community UI.

Maintainers & contact
- See repository for current maintainers and contributors. For enterprise / paid support, follow contact guidance in the docs.

Acknowledgements
- Inspired by "Generative Agents: Interactive Simulacra of Human Behavior" (Stanford) and Auto-GPT; built to work with OpenAI, LangChain, and Qdrant.

Enjoy building collaborative agent systems with GenWorlds — if you run into issues, open an issue or join the Discord community.