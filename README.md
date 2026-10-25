# Stateful Trajectory Analysis of LLM-Driven Network Attacks

A post-hoc analysis pipeline that goes beyond pass/fail metrics to measure *how well* an LLM red-team agent attacks a network. It replays an Incalmo action log to reconstruct an oracle attack graph and optimal path, aligns the agent's actual trajectory against that path, and labels every deviation with a failure category (irrelevant, redundant, dead-end, and others). It outputs annotated traces, summary metrics such as path efficiency and waste ratio, and visualizations that show where the agent drifted from optimal behavior.

**18-731 Network Security, Carnegie Mellon University, Spring 2026**  
Christian Wong, Karthik Chakkalakal, Cora Marcet

## About this repository

This repository is a fork of [Incalmo](https://github.com/bsinger98/Incalmo), an autonomous LLM-assisted red-teaming system from CMU CyLab ([paper](https://arxiv.org/abs/2501.16466)). We did not modify Incalmo itself, apart from a small update to the Haiku model name.

Our code lives in [`optimal_path_analysis/`](optimal_path_analysis/). See [that folder's README](optimal_path_analysis/README.md) for full details.

## How it works

The pipeline runs on a standard Incalmo `action_log.jsonl` and has three modules:

- **Oracle Solver** replays every event in the log to rebuild the network as the agent experienced it (hosts, subnets, credentials, CVEs, infection chains), then runs BFS to find the shortest action sequence that exfiltrates all critical data.
- **Alignment Engine** compares the actual trace to the optimal path with a set-based parallel walk, tagging each step as PRODUCTIVE, DEVIATION or FAILED_EXECUTION. Productive steps taken out of order (e.g. batching phases across hosts) are flagged rather than counted as waste.
- **Taxonomy Classifier** resolves each non-productive step into IRRELEVANT, FAILED_EXECUTION, REDUNDANT (re-doing completed work, our "frontier decay" signal), DEAD_END or SUBOPTIMAL_ORDERING, using the oracle graph and a running memory of completed actions.

Outputs: `analysis_report.json`, `aligned_trace.json`, `classified_trace.json`, `summary_report.json`, plus a `trajectory_timeline.png` and an optional side-by-side `attack_trajectory.mp4`.

## Results

We ran the pipeline on two traces from the MHBench EquifaxLarge environment (~50 hosts, 48 goal databases):

| Run | Steps | Goals reached | Path efficiency | Wasted steps |
|---|---|---|---|---|
| Claude Sonnet 4 (sample log from Incalmo) | 156 | 48/48 | 0.942 | 9 (3 irrelevant, 3 redundant, 3 suboptimal ordering) |
| Claude Haiku 4.5 (our run) | 69 | 0/48 | undefined | 69 (33 failed execution, 22 redundant, 14 suboptimal exploration) |

Sonnet's 9 wasted steps all fell in one early block (steps 2–10): it picked the wrong web server, tried lateral moves along invalid edges, and re-scanned a subnet it had already mapped, then ran the remaining 146 steps without deviation. Haiku never exploited the Struts CVE and stalled at lateral movement, so no attack graph could be reconstructed; path efficiency is undefined because the solver found no goals, and the classifier fell back to graph-free labels.

**Synthetic validation:** we injected 10 known failures into the Sonnet trace, and the pipeline flagged all 10 as non-productive. Some were assigned a different failure category than the one we injected, so this validates detection, not exact categorization.

## Limitations

- Results come from only two runs, in one environment, with one trial per model, so they are suggestive rather than conclusive.
- DEAD_END almost never fires on EquifaxLarge, because every host can re-pivot to a credential-bearing host; it should be more informative on more partitioned networks.
- The checks for FindInformationOnAHost and EscalatePrivilege are permissive: they accept any host with an inbound edge rather than only hosts actually compromised at that point in the trace.

## Running it

```bash
cd optimal_path_analysis
python main.py <path/to/action_log.jsonl>
```

See the [folder README](optimal_path_analysis/README.md) for options, outputs and setup.

## Contributions

- **Karthik Chakkalakal:** majority of the pipeline code (the optimal path solver, the trace alignment engine, the `main.py` script tying the pipeline together, and later fixes to the taxonomy labels), plus the Haiku 4.5 run and the synthetic injection outputs.
- **Christian Wong** contributed the initial taxonomy classifier and log parser.
- **Cora Marcet** wrote the first summary report generator (`generate_report.py`, which builds the summary from the classified trace) and the graph visualization scripts.

---




# Original Incalmo README

# Incalmo: An Autonomous LLM-assisted System for Red Teaming Multi-Host Networks

<div align="center">

[![MIT License](https://img.shields.io/badge/License-MIT-green.svg)](https://choosealicense.com/licenses/mit/)
![GitHub issues](https://img.shields.io/github/issues/bsinger98/Incalmo?style=flat-square)
![GitHub pull requests](https://img.shields.io/github/issues-pr/bsinger98/Incalmo?style=flat-square)
![GitHub commit activity](https://img.shields.io/github/commit-activity/m/bsinger98/Incalmo?style=flat-square)
![GitHub contributors](https://img.shields.io/github/contributors/bsinger98/Incalmo?style=flat-square)
![GitHub stars](https://img.shields.io/github/stars/bsinger98/Incalmo?style=flat-square)
![GitHub forks](https://img.shields.io/github/forks/bsinger98/Incalmo?style=flat-square)

Incalmo is an autonomous AI-driven network penetration testing tool that automatically conducts intelligent red-teaming activities with the aim to enhance and assist operator abilities when performing complex network attack tasks.

**Research Paper**: [On the Feasibility of Using LLMs to Execute Multistage Network Attacks](https://arxiv.org/abs/2501.16466)

**Website**: Visit our [website](https://www.incalmo.ai/) for more information! 
</div>

---


## Table Of Contents

- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Usage](#usage)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Contributing](#contributing)
- [License](#license)
- [Acknowledgements](#acknowledgements)
- [Support](#support)

## Prerequisites

- **[Docker Desktop](https://www.docker.com/)** - Required for containerized environment
- **[Python 3.13+](https://www.python.org/downloads/)** - Required for local development
- **[uv](https://github.com/astral-sh/uv)** - Fast Python package installer (recommended) or pip
- **LLM API Keys** - At least one of the following:
  - [OpenAI API Key](https://platform.openai.com/api-keys) - For GPT models
  - [Anthropic API Key](https://console.anthropic.com/) - For Claude models
  - [Google API Key](https://makersuite.google.com/app/apikey) - For Gemini models
  - [DeepSeek API Key](https://platform.deepseek.com/) - For DeepSeek models
- **[Node.js](https://nodejs.org/en)** (Optional) - Only needed for [UI Interface](#ui-interface-optional)
## Installation

#### 1. Setup configuration

Create a configuration file by copying the example:

```bash
cp config/config_example.json config/config.json
```

Then edit `config/config.json` as needed.

#### 2. Set API Keys

Create an environment file by copying the example:

```bash
cp .env.example .env
```

Then add LLM API keys to `.env`.

#### 3. Start the Development Environment

Navigate to the docker directory and start the containers:

```bash
cd docker
docker compose up
```

#### 4. Run Incalmo

In a new terminal window, attach to the running container and execute Incalmo:

   ```bash
   cd docker
   docker compose exec attacker /bin/bash
   uv run main.py
   ```

### UI Interface (optional)

If you want to use the web-based interface for Incalmo:

#### 1. Start Backend

Follow Steps 1 through 3 in the [Setup Instructions](#setup-instructions).

#### 2. Install Node.js dependencies

Install Node dependencies:

   ```bash
   cd incalmo/frontend/incalmo-ui
   npm install
   ```

#### 3. Start the React Server

Once dependencies are installed, run the react server:

   ```bash
   npm start
   ```

This will lauch the frontend at [http://localhost:3000](http://localhost:3000)

## Usage

Note: A "strategy" is the logic behind an attack. See the [strategies/](incalmo/core/strategies) folder for examples of llm based attacks and state machine based attacks.

To use an your choice of an LLM-based attack:

- Follow the [setup](#installation) and then:

- Specify in ```config/config.json``` what LLMs to use (list is available in the [registry](incalmo/core/strategies/llm/langchain_registry.py)). Use the ```config_example.json``` as a template

- Run ```main.py``` as described in the setup

- Observe the status of the attack through the logs in the output directory. Your attack will be timestamped and used to name the logs folder

This is the most stable way to test attack.

Incalmo also supports creating non-LLM state-machine strategies. To use a custom (non-LLM/manual) strategy for attacks:

-  Follow the [setup](#installation) and then:

- Specify in ```config/config.json``` what manual strategy to use (list is available in [strategies/state_machine](incalmo/core/strategies/state_machine)). Use the ```config_example_state_machine.json``` as a template and fill in the strategy name as the class name.

- Run ```main.py``` as described in the setup

- Observe the status of the attack through the logs in the output directory. Your attack will be timestamped and used to name the logs folder

- To create your own basic strategy, create a file in [strategies/state_machine](incalmo/core/strategies/state_machine) as follows:

```
class YourStrategyName(IncalmoStrategy):
    async def step(self) -> bool:
    # Your attack/strategy logic
```
It may be easier to look at examples of existing strategies to understand formatting and strategy abilities

To use the UI:

- Instead of running ```main.py```, launch the frontend and use the UI to start and stop attacks and observe logs. This is the cleanest/easiest way to observe the attacks in real time 
    
## Tech Stack

**Backend:** Python 3.13, Flask, Celery, SQLite \
**LLM Integration:** LangChain, OpenAI, Anthropic, Google Gemini, DeepSeek \
**Frontend:** React, TypeScript, Node.js \
**Containerization:** Docker, Docker Compose \
**Package Management:** uv 

## Project Structure

```
Incalmo/
├── .env.example               # Template for environment configuration
├── CITATION.cff               # Research paper citation metadata
├── LICENSE                    # MIT License
├── main.py                    # CLI entry point - runs Incalmo strategy
├── README.md                  # Project Guide
├── config/                    # Configuration management
├── docker/                    # Docker containerization
│   ├── attacker/              # Attacker container configuration
│   └── equifax/               # Target environment (Equifax breach simulation)
│       ├── database/          # Database server container
│       └── webserver/         # Web server container
├── incalmo/                   # Core application code
│   ├── incalmo_runner.py      # Main strategy execution runner
│   ├── server.py              # Flask server entry point
│   ├── api/                   # Client API for C2 server communication
│   ├── c2server/              # Command & Control server
│   │   ├── agents/            # Agent implementations
│   │   ├── celery/            # Async task queue
│   │   ├── payloads/          # Exploit and deployment payloads
│   │   └── routes/            # Flask blueprints for API endpoints
│   ├── core/                  # Core attack framework
│   │   ├── actions/           # Action classes 
│   │   │   ├── EmptyServiceActions/   # Placeholder actions
│   │   │   ├── HighLevel/         # High-level actions
│   │   │   │   └── llm_agents/        # LLM-agent action implementations
│   │   │   └── LowLevel/          # Low-level commands
│   │   │       └── privledge_escalation/      # Privilege escalation exploits
│   │   ├── models/            # Core domain models
│   │   │   ├── events/        # Event system for state updates
│   │   │   └── network/       # Network infrastructure models
│   │   ├── services/          # Core logic services
│   │   └── strategies/        # Attack strategies
│   │       ├── llm/               # LLM-based strategies
│   │       ├── state_machine/     # Rule-based strategies
│   │       ├── testers/           # Strategy testing utilities
│   │       └── util/              # Strategy utilities
│   ├── frontend/              # Web interface
│   │   └── incalmo-ui/        # React-based UI
│   └── models/                # Shared data models (Pydantic)
└── output/                    # Execution logs and results

```

## Contributing

Contributions are always welcome! Please raise issues or make PR's if you have ideas on how to improve this project


## License

This project is licensed under the [MIT](LICENSE) License
## Acknowledgements

To cite this project or paper, check out the [Citation](CITATION.cff) specifications


## Support

For support, email hello@incalmo.ai

