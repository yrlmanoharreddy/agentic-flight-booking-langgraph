# Agentic Flight Booking (LangGraph)

A stateful, multi-step flight booking agent built with LangGraph and ChatGroq, where every step of the booking flow reads from and writes to one shared, typed state object instead of passing ad-hoc arguments between functions.

## Problem

Chaining LLM calls with plain function calls quickly turns into an untraceable mess of parameters once a workflow has more than one or two steps. This project models flight booking as an explicit graph of nodes over a single `TypedDict` state (`FlightState`), so every step's input and output is visible in one place and the whole trace can be inspected after a run.

## Architecture

```mermaid
flowchart LR
Start([START]) --> SearchFlights[search_flights]
SearchFlights --> CheckFare[check_fare_and_route]
CheckFare --> BookingConfirmation[booking_confirmation]
BookingConfirmation --> End([END])
```

Each node receives the current `FlightState`, calls `ChatGroq` when it needs to generate or validate data, and returns an updated state. LangGraph's `StateGraph` wires the nodes together and compiles them into a single runnable graph.

## Tech Stack

The workflow is implemented in Python using LangGraph's `StateGraph`, `START`, and `END` primitives, `langchain-core` message types (`HumanMessage`, `AIMessage`), and `langchain-groq` for LLM access. State is modeled with `typing_extensions.TypedDict` and LangGraph's `add_messages` reducer for the running chat history.

## How It Works

The graph has three nodes. `search_flights` prompts the LLM to generate realistic sample flights between the requested origin and destination as JSON, falling back to a small hardcoded flight list if the model does not return valid JSON. `check_fare_and_route` validates the route and computes fare details from the selected flight. `booking_confirmation` produces the final booking confirmation. All three nodes read and write the same `FlightState`, so the full history of the booking is available at the end of the run.

## Key Features

The project demonstrates typed, shared state across every step of an agentic workflow, a defensive JSON-parsing fallback so the graph stays functional even when the LLM response is not perfectly structured, and a linear LangGraph pipeline that is easy to extend with additional nodes such as payment or seat selection.

## API Flow

An initial `FlightState` containing the user's request (for example, "Book a flight from Saint Louis to Atlanta") enters the graph at `START`. It flows through `search_flights`, `check_fare_and_route`, and `booking_confirmation` in sequence, with each node mutating the shared state, before reaching `END` with a complete booking record.

## Setup

```bash
git clone https://github.com/yrlmanoharreddy/agentic-flight-booking-langgraph.git
cd agentic-flight-booking-langgraph
uv sync
export GROQ_API_KEY="your-api-key-here"
jupyter notebook notebook/5-flight-reservation-sys.ipynb
```

## Testing

This is currently an exploratory notebook-driven workflow rather than a package with automated tests. The JSON-parsing fallback in `search_flights` acts as a lightweight safety net against malformed LLM output, but adding `pytest` coverage around each node function is the clear next step.

## Deployment

The workflow currently runs as a notebook and is not packaged as a service. Wrapping the compiled LangGraph graph in a small FastAPI endpoint would be the natural way to expose it as a callable booking API.

## Engineering Decisions

A single shared `TypedDict` state was chosen over passing individual arguments between steps specifically so the entire booking trace could be printed and inspected at the end of a run, which matters when debugging why an agent produced a particular booking outcome. The fallback flight list in `search_flights` was a deliberate choice to keep the graph runnable during development even when the LLM's JSON output is inconsistent.

## Status

The three-node graph is implemented and runnable end to end against a real Groq-backed LLM. It is intentionally scoped to the booking decision flow rather than a production reservation system with payment or persistence.
