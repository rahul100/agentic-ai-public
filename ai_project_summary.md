# Undertsand the project easily


Hello! I'm here to help you dive deep into this agentic AI project. It's an excellent demo that showcases a multi-agent system, complete with planning, tool use, and a web interface.

Let's break down the entire repository, file by file, and then I'll give you the exact steps to get it running on your MacBook.

🚀 Project Analysis: What Does This Code Do?
At a high level, this project builds a web application where a user can submit a research topic. This topic is then picked up by a team of AI agents who:

Plan: An initial "Planning Agent" breaks the complex research task into a series of smaller, concrete steps.

Execute: A "Dispatcher" agent (called the executor_agent_step) reads this plan and assigns each step to a specialized "worker" agent.

Research: A research_agent uses external tools—like Tavily for web searches, arXiv for academic papers, and Wikipedia for general knowledge—to gather information.

Write: A writer_agent takes all the research findings and synthesizes them into a formatted report.

Edit: An editor_agent reviews and polishes the report.

Serve: All of this is managed by a FastAPI backend, which saves the progress to a PostgreSQL database and serves a simple HTML/JavaScript frontend that polls for live updates.

Codebase Explanation (File by File)
Here is a breakdown of what each file in the project is responsible for.

main.py (The Backend Server)
This is the core of your application. It's the web server, task manager, and database connector all in one.

Technology: It uses FastAPI, a modern, high-performance Python web framework.

Database: It uses SQLAlchemy to connect to a PostgreSQL database. It defines a Task table to store the user's prompt, the task's status (e.g., running, done, error), and the final result.

Endpoints (API):

@app.get("/"): Serves the index.html file as the main web page.

@app.post("/generate_report"): This is the most important endpoint.

It receives a user's prompt.

It creates a new Task in the database.

It immediately calls planner_agent() to get the list of steps.

It kicks off the actual work (run_agent_workflow) in a separate background thread using threading.Thread. This is crucial so the user gets an immediate response ("Task started!") instead of waiting for the full report.

@app.get("/task_progress/{task_id}"): The frontend calls this endpoint every few seconds to get the latest status of each step, which it uses to update the UI live.

@app.get("/task_status/{task_id}"): The frontend calls this to see if the entire job is finished and to fetch the final report.

run_agent_workflow(...): This is the background function that does the real work. It iterates through the plan (e.g., "Step 1: Research..._," "Step 2: Write...") and calls executor_agent_step for each one, collecting the results as it goes. When finished, it updates the database task with status="done" and saves the final report.

src/planning_agent.py (The "Manager" Agents)
This file defines the high-level logic for planning and executing the workflow.

planner_agent(...): This is the "General Contractor."

It's given the user's topic.

It uses an LLM (like openai:o4-mini) with a specific prompt that tells it to create a step-by-step plan as a Python list.

It has strict instructions (guardrails) to always use Tavily for a broad search first, then use arXiv for a targeted search.

It includes robust parsing to handle the LLM's output, which might not be perfectly formatted.

executor_agent_step(...): This is the "Dispatcher" or "Foreman."

It receives one step from the planner (e.g., "Research agent: Find recent news on..._").

Its job is to route this task to the correct worker agent.

It builds an "enriched prompt" that includes the original user request and the history of all previous steps. This gives the worker agents context.

It uses simple if/elif logic on the step's title (e.g., if "research" in step, call research_agent) to make the call.

src/agents.py (The "Worker" Agents)
These are the specialized agents that perform the individual tasks. They are all powered by LLM calls using the aisuite library.

research_agent(...):

This agent's prompt tells it to act as a research assistant.

It is given access to tools (arxiv_search_tool, tavily_search_tool, wikipedia_search_tool).

When you call this agent, the LLM (gpt-4.1-mini) will analyze the request and decide which of those tools to call (and with what arguments) to best answer the prompt. This is the core of "tool use."

writer_agent(...):

This agent's prompt instructs it to be an expert academic writer.

It is given a strict Markdown structure to follow (Title, Abstract, Introduction, etc.).

It does not have tools. Its only job is to synthesize the large amount of text (the research findings) given to it.

editor_agent(...):

This agent's prompt tells it to review and refine an existing draft, checking for flow, clarity, and accuracy.

src/research_tools.py (The "Tools")
This file contains the actual Python functions that the research_agent can call. For an LLM to use a function, that function must exist as real Python code.

tavily_search_tool(...): A wrapper for the Tavily AI search API, a search engine built specifically for AI agents. It requires a TAVILY_API_KEY.

arxiv_search_tool(...):

A function that queries the arXiv API for academic papers.

It goes a step further: it attempts to download the PDF of the paper, extract its full text using libraries like PyMuPDF (fitz) or pdfminer.six, and return that text. This is far more powerful than just reading the abstract.

wikipedia_search_tool(...): A simple wrapper for the wikipedia Python library to get article summaries.

templates/index.html (The Frontend)
This is the user interface you see in your browser.

HTML/CSS: It's a single page styled with Bootstrap for a clean look. It has a text area for the prompt and empty <div> containers to hold the progress and final report.

JavaScript:

submitPrompt(): Triggered when you click "Submit." It sends the prompt to the /generate_report backend endpoint.

pollInterval = setInterval(fetchProgress, 2000): This is the key to the "live" feel. Every 2 seconds, it calls fetchProgress().

fetchProgress(): Calls the /task_progress/... endpoint. It gets the list of steps and their statuses and dynamically renders them onto the page.

fetchTaskStatus(): Checks if the job is done. If so, it stops the polling, fetches the final report from /task_status/..., and uses the Marked.js library to render the report's Markdown as clean HTML.

Dockerfile & docker/entrypoint.sh
These files are for containerization using Docker.

Dockerfile: A blueprint for building a Docker image. It starts from a Python 3.11 image, installs system-level needs (like postgresql and gcc), installs all the Python packages from requirements.txt, and copies the application code.

docker/entrypoint.sh: A clever script that runs when the container starts. It starts a PostgreSQL server inside the container, creates the necessary user and database, and then starts the FastAPI server. This makes the entire application (app + database) self-contained in a single container for an easy demo.

🛠️ How to Run Locally on Your MacBook
Here are the detailed steps to run this project directly on your Mac without using Docker.

Step 1: Prerequisites
Before you start, you'll need a few things installed. The easiest way to get them on a Mac is with Homebrew.

Xcode Command Line Tools: This is essential for compiling some Python packages. Open your Terminal and run:

Bash

xcode-select --install
Homebrew: If you don't have it, install it from brew.sh.

Git:

Bash

brew install git
Python 3.11: The project is based on 3.11.

Bash

brew install python@3.11
PostgreSQL: This is the database.

Bash

brew install postgresql
Step 2: Get the Code
Clone the repository and move into the directory.

Bash

git clone https://github.com/https-deeplearning-ai/agentic-ai-public.git
cd agentic-ai-public
Step 3: Set Up the Python Environment
It's best practice to use a virtual environment to avoid conflicts.

Bash

# Create a virtual environment using Python 3.11
python3.11 -m venv venv

# Activate it (you must do this every time you open a new terminal)
source venv/bin/activate

# Install all the required Python packages
pip install -r requirements.txt
Step 4: Set Up the PostgreSQL Database
The app needs a running database, a user, and a password that match its configuration.

Start the PostgreSQL service:

Bash

brew services start postgresql
Create the database: (The default is appdb)

Bash

createdb appdb
Create the user and password: (Defaults are app and local)

Bash

createuser app
psql -c "ALTER USER app WITH PASSWORD 'local';"
Step 5: Get API Keys & Create .env File
The agents need API keys to function.

Tavily API Key:

Go to tavily.com, sign up, and get your free API key.

OpenAI API Key:

Go to platform.openai.com and get your API key. (The aisuite library used here is a wrapper for the OpenAI API).

Create the .env file: This file securely stores your keys where the app can find them.

Bash

touch .env
Edit the .env file (e.g., nano .env) and add the following, pasting in your actual keys.

Ini, TOML

# Database URL
DATABASE_URL="postgresql://app:local@localhost:5432/appdb"

# API Keys
OPENAI_API_KEY="sk-YOUR_OPENAI_API_KEY_HERE"
TAVILY_API_KEY="tvly-YOUR_TAVILY_API_KEY_HERE"
Step 6: Run the Application!
You're all set. With your virtual environment still active (you should see (venv) in your terminal prompt), run the FastAPI server using uvicorn:

Bash

uvicorn main:app --host 0.0.0.0 --port 8000
You'll see output indicating the server is running.

Step 7: Access Your Agent
Open your web browser (like Chrome or Safari) and go to:

http://localhost:8000

You should see the web interface. Type in a research prompt and watch your agents get to work!

I hope this detailed breakdown was helpful! This is a fantastic project for understanding the core components of modern agentic systems.
