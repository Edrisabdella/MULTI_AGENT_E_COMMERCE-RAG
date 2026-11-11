"""
agent_orchestrator.py
=====================
Enterprise Multi-Agent Customer Support System
Built with Strands Agents SDK + Amazon Bedrock AgentCore

Architecture implemented:

  Customer Request
        │
  OrchestratorAgent  (Claude Haiku 4.5 - fast routing, manages WorkflowState)
        │
   ┌────┼────────────────────┬────────────────────────┐
   │    │                    │                        │
InventoryAgent   PolicyAgent   RefundAgent  CommunicationAgent
(DynamoDB)    (Multi-Agent RAG)  (DynamoDB)   (composes response)
                    │
         ┌──────────┼──────────┐
    ReturnsPolicyRetriever  ShippingPolicyRetriever  WarrantyPolicyRetriever
        (KB: returns)           (KB: shipping)           (KB: warranty)
         └──────────── all run in PARALLEL ────────────┘

Shared state flows through DynamoDB WorkflowStateTable.
OrchestratorAgent creates state at start, each routing tool reads and
updates it after the worker responds.

Commands:
  python src/agent_orchestrator.py test            # 3 scenarios, local run, traced to X-Ray
  python src/agent_orchestrator.py chat            # interactive terminal chat
  python src/agent_orchestrator.py deploy          # Tasks 3-6 deployment pipeline (uses the AgentCore CLI)
  python src/agent_orchestrator.py invoke "<msg>"  # call the deployed AgentCore Runtime
  python src/agent_orchestrator.py serve           # HTTP server (what AgentCore Runtime runs)

Deployment uses the AgentCore CLI (`agentcore`, npm package @aws/agentcore,
https://github.com/aws/agentcore-cli) through the pre-written helper
src/agentcore_cli.py - see the README for the prerequisites (Node.js 20+, uv).
"""

import boto3
import json
import time
import os
import sys
import uuid
import logging
import threading
from datetime import datetime, timezone
from concurrent.futures import ThreadPoolExecutor, as_completed
from typing import Optional

# Ensure the parent directory is on sys.path so config.py and
# bedrock_kb_retrieval.py are importable regardless of where this
# script is invoked from (e.g. python src/agent_orchestrator.py)
sys.path.insert(0, os.path.dirname(os.path.dirname(os.path.abspath(__file__))))
sys.path.insert(0, os.path.dirname(os.path.abspath(__file__)))

# Strands Agents SDK - see: https://github.com/strands-agents/sdk-python
from strands import Agent
from strands.models import BedrockModel
from boto3.dynamodb.conditions import Key
from botocore.exceptions import ClientError

import config
from bedrock_kb_retrieval import retrieve_from_knowledge_base, format_kb_results

# Configure logging for debugging
logging.basicConfig(
    level=logging.WARNING,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)


# ─────────────────────────────────────────────────────
# OUTPUT UTILITIES
# ─────────────────────────────────────────────────────
# Terminal trace UI, ANSI colour constants, and agent metadata
# are defined in agent_utils.py - keeping this file focused on
# agent architecture.
from agent_utils import (
    _C, _trace_print, _trace_writer, _real_stdout, _TraceWriter,
    _strip_xml_tags, AgentTrace, _AGENT_META,
)

# ─────────────────────────────────────────────────────
# OBSERVABILITY
# ─────────────────────────────────────────────────────
# `tool` is the Strands @tool decorator wrapped so that every tool call is
# recorded as an X-Ray subsegment (the orchestrator's route_to_* tools become
# the worker-agent nodes on the X-Ray Service Map) and logged at INFO level.
# Use it exactly like `strands.tool`:  @tool  above each tool function.
from agent_observability import (
    tool, tracer, setup_logging, flush_logs, print_trace_hint,
    apply_observability_config, wait_for_runtime_ready,
)


# ─────────────────────────────────────────────────────
# AWS CLIENTS
# ─────────────────────────────────────────────────────
bedrock_agent_client = boto3.client('bedrock-agent', region_name=config.AWS_REGION)
bedrock_runtime      = boto3.client('bedrock-runtime', region_name=config.AWS_REGION)
agentcore_client     = boto3.client('bedrock-agentcore', region_name=config.AWS_REGION)
agentcore_control    = boto3.client('bedrock-agentcore-control', region_name=config.AWS_REGION)
dynamodb             = boto3.resource('dynamodb', region_name=config.AWS_REGION)
logs_client          = boto3.client('logs', region_name=config.AWS_REGION)


# ═══════════════════════════════════════════════════════
#  WORKFLOW STATE - SHARED DynamoDB STATE OBJECT
#
#  WorkflowState stores the accumulated context for one customer session:
#    - What the InventoryAgent found (order status, eligibility, customer tier)
#    - What the PolicyAgent found (relevant policy text)
#    - What the RefundAgent decided (approval/denial, reference number)
#    - The CommunicationAgent's final draft
#
#  The `version` field enables optimistic locking: every write is a
#  conditional DynamoDB update that fails if someone else updated first.
#  If the condition fails, the update is retried after a fresh read.
# ═══════════════════════════════════════════════════════

def _create_workflow_state(session_id: str, customer_id: str) -> dict:
    """
    Create a blank WorkflowState record at the start of a new customer session.

    Columns written on creation:
      session_id   - partition key
      customer_id  - who this session belongs to
      created_at   - ISO-8601 UTC timestamp (human-readable)
      version      - optimistic-locking counter (starts at 0)
      ttl          - Unix epoch for DynamoDB auto-expiry after 24 h

    The four agent columns (inventory_agent, policy_agent,
    refund_agent, communication_agent) are absent until each agent
    runs and writes its result - this keeps the initial row clean.
    """
    state = {
        'session_id':  session_id,
        'customer_id': customer_id,
        'created_at':  time.strftime('%Y-%m-%dT%H:%M:%SZ', time.gmtime()),
        'version':     0,
        'ttl':         int(time.time()) + (24 * 3600),
    }
    table = dynamodb.Table(config.WORKFLOW_STATE_TABLE)
    table.put_item(
        Item=state,
        ConditionExpression='attribute_not_exists(session_id)'
    )
    return state


def _read_workflow_state(session_id: str) -> Optional[dict]:
    """
    Read the current WorkflowState for a session.
    """
    table = dynamodb.Table(config.WORKFLOW_STATE_TABLE)
    response = table.get_item(Key={'session_id': session_id}, ConsistentRead=True)
    return response.get('Item')


# Trace singleton - created after _read_workflow_state so AgentTrace.summary()
# can read DynamoDB WorkflowState. The read_state_fn avoids a circular import.
trace = AgentTrace(read_state_fn=_read_workflow_state)


def _update_workflow_state(session_id: str, updates: dict,
                           expected_version: int, max_retries: int = 3) -> dict:
    """
    Update WorkflowState with optimistic locking.
    """
    from boto3.dynamodb.conditions import Attr

    table = dynamodb.Table(config.WORKFLOW_STATE_TABLE)

    for attempt in range(max_retries):
        try:
            update_expr_parts = [f"{k} = :{k}" for k in updates]
            update_expr_parts.append("version = :new_version")
            update_expr = "SET " + ", ".join(update_expr_parts)

            expr_values = {f":{k}": v for k, v in updates.items()}
            expr_values[':new_version']      = expected_version + 1
            expr_values[':expected_version'] = expected_version

            table.update_item(
                Key={'session_id': session_id},
                UpdateExpression=update_expr,
                ConditionExpression='version = :expected_version',
                ExpressionAttributeValues=expr_values
            )
            return _read_workflow_state(session_id)

        except dynamodb.meta.client.exceptions.ConditionalCheckFailedException:
            if attempt == max_retries - 1:
                raise RuntimeError(
                    f"WorkflowState update failed after {max_retries} retries "
                    f"(session: {session_id}). Too many concurrent writes."
                )
            logger.warning(
                f"WorkflowState version conflict on attempt {attempt+1}, retrying..."
            )
            current = _read_workflow_state(session_id)
            if current:
                expected_version = int(current['version'])
            time.sleep(0.1 * (attempt + 1))

    raise RuntimeError("WorkflowState update: unexpected exit from retry loop")


# ═══════════════════════════════════════════════════════
#  TASK 2 - MULTI-AGENT ORCHESTRATION
# ═══════════════════════════════════════════════════════


# ───────────────────────────────────────────────────────
#  2.A - INVENTORY AGENT
# ───────────────────────────────────────────────────────

def build_inventory_agent() -> Agent:
    """Build a factual-only worker that reads customer and order records."""
    model = BedrockModel(model_id=config.WORKER_MODEL_ID, temperature=0.1)
    system_prompt = """You are NovaMart's Inventory Agent. You are a factual data-gatherer only.
Use your DynamoDB tools to retrieve order and customer records accurately. Never
infer or decide return/refund eligibility, never change records, and clearly state
when a record is missing. Do not invent facts. Include identifiers and relevant dates."""

    @tool
    def check_order_status(customer_id: str, order_id: str) -> dict:
        """Fetch one order using its composite key.

        Args: customer_id: Customer partition key. order_id: Order sort key.
        Returns: The complete order record, or a structured not-found result.
        """
        try:
            item = dynamodb.Table(config.ORDERS_TABLE).get_item(
                Key={'customer_id': customer_id, 'order_id': order_id},
                ConsistentRead=True
            ).get('Item')
            return {'found': True, 'order': item} if item else {
                'found': False, 'customer_id': customer_id, 'order_id': order_id,
                'message': 'No matching order was found.'}
        except Exception as exc:
            logger.exception('Order lookup failed')
            return {'found': False, 'error': str(exc), 'customer_id': customer_id, 'order_id': order_id}

    @tool
    def get_customer_tier(customer_id: str) -> dict:
        """Retrieve a customer profile and tier from DynamoDB.

        Args: customer_id: Customer partition key.
        Returns: Customer profile including tier, or a not-found/error result.
        """
        try:
            item = dynamodb.Table(config.CUSTOMERS_TABLE).get_item(
                Key={'customer_id': customer_id}, ConsistentRead=True
            ).get('Item')
            return {'found': True, 'customer': item} if item else {
                'found': False, 'customer_id': customer_id, 'message': 'Customer not found.'}
        except Exception as exc:
            logger.exception('Customer lookup failed')
            return {'found': False, 'error': str(exc), 'customer_id': customer_id}

    @tool
    def list_customer_orders(customer_id: str) -> dict:
        """List all orders belonging to a customer.

        Args: customer_id: Customer partition key.
        Returns: Orders and a count, or a structured error result.
        """
        try:
            response = dynamodb.Table(config.ORDERS_TABLE).query(
                KeyConditionExpression=Key('customer_id').eq(customer_id),
                ConsistentRead=True
            )
            items = response.get('Items', [])
            while response.get('LastEvaluatedKey'):
                response = dynamodb.Table(config.ORDERS_TABLE).query(
                    KeyConditionExpression=Key('customer_id').eq(customer_id),
                    ConsistentRead=True, ExclusiveStartKey=response['LastEvaluatedKey']
                )
                items.extend(response.get('Items', []))
            return {'customer_id': customer_id, 'count': len(items), 'orders': items}
        except Exception as exc:
            logger.exception('Customer order listing failed')
            return {'customer_id': customer_id, 'orders': [], 'error': str(exc)}

    return Agent(model=model, system_prompt=system_prompt,
                 tools=[check_order_status, get_customer_tier, list_customer_orders])


def build_refund_agent() -> Agent:
    """Build the policy-aware return/refund decision worker."""
    model = BedrockModel(model_id=config.WORKER_MODEL_ID, temperature=0.1)
    system_prompt = """You are NovaMart's Refund Agent. ALWAYS call get_inventory_context first.
Use only the retrieved order/customer facts. Standard customers have a 30-day
return window; Premium customers have a 60-day window. Evaluate the age from the
order_date available in the record, require a delivered order, and do not approve
ineligible, missing, or ambiguous orders. Explain the decision and its basis.
Only call initiate_refund after confirming eligibility. Never claim a refund was
issued unless the tool confirms the return request was recorded."""

    @tool
    def get_inventory_context(session_id: str) -> dict:
        """Read the inventory worker's findings from shared WorkflowState.

        Args: session_id: Current workflow session identifier.
        Returns: Inventory findings, or an explicit missing-state result.
        """
        state = _read_workflow_state(session_id) or {}
        return state.get('inventory_agent') or {'available': False, 'message': 'Inventory context is not available.'}

    @tool
    def initiate_refund(customer_id: str, order_id: str, reason: str) -> dict:
        """Record an eligible return request in the order's DynamoDB item.

        Args: customer_id: Customer partition key. order_id: Order sort key.
              reason: Customer-provided return reason.
        Returns: Return reference and next steps, or a denial/error result.
        """
        table = dynamodb.Table(config.ORDERS_TABLE)
        try:
            order = table.get_item(Key={'customer_id': customer_id, 'order_id': order_id},
                                   ConsistentRead=True).get('Item')
            if not order:
                return {'approved': False, 'message': 'Order not found; no return was initiated.'}
            customer = dynamodb.Table(config.CUSTOMERS_TABLE).get_item(
                Key={'customer_id': customer_id}, ConsistentRead=True).get('Item') or {}
            tier = str(customer.get('tier', 'Standard')).strip().lower()
            window = 60 if tier == 'premium' else 30
            try:
                order_date = datetime.strptime(str(order['order_date'])[:10], '%Y-%m-%d').date()
                age_days = (datetime.now(timezone.utc).date() - order_date).days
            except (KeyError, ValueError, TypeError):
                return {'approved': False, 'message': 'Order date is missing or invalid; manual review is required.'}
            if str(order.get('status', '')).lower() != 'delivered':
                return {'approved': False, 'message': 'Only delivered orders can be returned.'}
            if age_days < 0 or age_days > window:
                return {'approved': False, 'tier': customer.get('tier', 'Standard'),
                        'age_days': age_days, 'return_window_days': window,
                        'message': f'The order is outside the {window}-day return window.'}
            reference = f'RET-{uuid.uuid4().hex[:10].upper()}'
            table.update_item(
                Key={'customer_id': customer_id, 'order_id': order_id},
                UpdateExpression='SET #status = :status, return_reference = :ref, return_reason = :reason, return_requested_at = :ts',
                ConditionExpression='#status = :delivered AND attribute_not_exists(return_reference)',
                ExpressionAttributeNames={'#status': 'status'},
                ExpressionAttributeValues={':status': 'return_requested', ':delivered': 'delivered',
                                           ':ref': reference, ':reason': reason,
                                           ':ts': datetime.now(timezone.utc).isoformat()}
            )
            return {'approved': True, 'return_reference': reference, 'tier': customer.get('tier', 'Standard'),
                    'age_days': age_days, 'return_window_days': window,
                    'instructions': 'Pack the item securely and follow the return-label instructions provided by NovaMart.'}
        except dynamodb.meta.client.exceptions.ConditionalCheckFailedException:
            return {'approved': False, 'message': 'A return has already been requested or the order status changed.'}
        except Exception as exc:
            logger.exception('Return initiation failed')
            return {'approved': False, 'error': str(exc), 'message': 'The return could not be recorded.'}

    return Agent(model=model, system_prompt=system_prompt, tools=[get_inventory_context, initiate_refund])


def build_policy_agent() -> Agent:
    """Build three single-domain retrievers and a parallel-RAG coordinator."""
    coordinator_model = BedrockModel(model_id=config.WORKER_MODEL_ID, temperature=0.2)

    @tool
    def retrieve_returns_policy(query: str) -> str:
        """Retrieve passages from the Returns Knowledge Base for a query."""
        return format_kb_results(retrieve_from_knowledge_base(config.RETURNS_KB_ID, query))
    returns_retriever = Agent(model=BedrockModel(model_id=config.WORKER_MODEL_ID, temperature=0.0),
        system_prompt='You retrieve only NovaMart returns policy passages. Call your retrieval tool and report its evidence without adding policy.',
        tools=[retrieve_returns_policy])

    @tool
    def retrieve_shipping_policy(query: str) -> str:
        """Retrieve passages from the Shipping Knowledge Base for a query."""
        return format_kb_results(retrieve_from_knowledge_base(config.SHIPPING_KB_ID, query))
    shipping_retriever = Agent(model=BedrockModel(model_id=config.WORKER_MODEL_ID, temperature=0.0),
        system_prompt='You retrieve only NovaMart shipping policy passages. Call your retrieval tool and report its evidence without adding policy.',
        tools=[retrieve_shipping_policy])

    @tool
    def retrieve_warranty_policy(query: str) -> str:
        """Retrieve passages from the Warranty Knowledge Base for a query."""
        return format_kb_results(retrieve_from_knowledge_base(config.WARRANTY_KB_ID, query))
    warranty_retriever = Agent(model=BedrockModel(model_id=config.WORKER_MODEL_ID, temperature=0.0),
        system_prompt='You retrieve only NovaMart warranty policy passages. Call your retrieval tool and report its evidence without adding policy.',
        tools=[retrieve_warranty_policy])

    @tool
    def search_all_policies(query: str) -> str:
        """Search Returns, Shipping and Warranty KB agents concurrently.

        Args: query: Natural-language customer policy question.
        Returns: Labeled combined evidence from all three policy domains.
        """
        retrievers = {'Returns': returns_retriever, 'Shipping': shipping_retriever,
                      'Warranty': warranty_retriever}
        trace.kb_start({'Returns': config.RETURNS_KB_ID, 'Shipping': config.SHIPPING_KB_ID,
                        'Warranty': config.WARRANTY_KB_ID})
        results = {}
        def _run_retriever(domain: str, agent, question: str) -> tuple:
            """Invoke one domain agent and return its label and textual evidence."""
            try:
                return domain, str(agent(question))
            except Exception as exc:
                logger.exception('%s policy retrieval failed', domain)
                return domain, f'Retrieval error: {exc}'
        with ThreadPoolExecutor(max_workers=3) as executor:
            futures = {executor.submit(_run_retriever, domain, agent, query): domain
                       for domain, agent in retrievers.items()}
            for future in as_completed(futures):
                domain, result = future.result()
                results[domain] = result
        trace.kb_done(len(retrievers))
        for domain in ('Returns', 'Shipping', 'Warranty'):
            trace.kb_result(domain, results.get(domain, '[No results]'))
        return '\n\n'.join(f'=== {domain} Policy ===\n{results.get(domain, "[No results]")}'
                            for domain in ('Returns', 'Shipping', 'Warranty'))

    return Agent(model=coordinator_model,
        system_prompt="""You are NovaMart's Policy Agent. ALWAYS call search_all_policies first.
Answer only from retrieved policy evidence. Synthesize relevant Returns, Shipping,
and Warranty passages, cite the domain/source when available, distinguish missing
information from policy facts, and never invent terms or customer/order data.""",
        tools=[search_all_policies])


def build_communication_agent() -> Agent:
    """Build the final customer-facing response composer."""
    model = BedrockModel(model_id=config.WORKER_MODEL_ID, temperature=0.3)
    system_prompt = """You are NovaMart's customer communication specialist. Always read the
complete WorkflowState using get_full_workflow_context before drafting. Address the
customer's actual request, accurately include relevant findings and decisions, explain
next steps, and use a warm, concise, professional and empathetic tone. Never invent
order details, policy terms, approvals, refunds, or reference numbers. If evidence is
missing or an action failed, say so clearly."""

    @tool
    def get_full_workflow_context(session_id: str) -> dict:
        """Read the complete accumulated WorkflowState for a customer session.

        Args: session_id: Current workflow session identifier.
        Returns: Full state record, or a clear missing-state result.
        """
        state = _read_workflow_state(session_id)
        return state if state else {'found': False, 'session_id': session_id,
                                    'message': 'Workflow state was not found.'}

    return Agent(model=model, system_prompt=system_prompt, tools=[get_full_workflow_context])


def build_orchestrator_agent(inventory_agent: Agent, refund_agent: Agent,
                             policy_agent: Agent, communication_agent: Agent) -> Agent:
    """Build the Haiku orchestrator and its five WorkflowState routing tools."""
    model = BedrockModel(model_id=config.ORCHESTRATOR_MODEL_ID, temperature=0.0)
    system_prompt = """You are NovaMart's request orchestrator, not a customer-facing responder.
For EVERY request, call initialize_session first. Classify intent and follow these
rules exactly: (1) order status/history, return and refund requests: Inventory first;
for return/refund then Refund. (2) return-window, shipping-rate and warranty-term
questions: Policy. (3) account/tier questions: Inventory only, never Policy. (4) pure
math/calculation: skip Inventory, Policy and Refund. (5) every request must finish by
calling route_to_communication_agent as the VERY LAST tool call. You must never write
the final customer response yourself. Pass the original request and identifiers to
workers. Do not skip initialization or the communication step."""

    def _state_or_error(session_id: str) -> dict:
        state = _read_workflow_state(session_id)
        if not state:
            raise RuntimeError(f'WorkflowState not initialized for session {session_id}')
        return state

    @tool
    def route_to_inventory_agent(session_id: str, customer_id: str, request: str) -> str:
        """Run InventoryAgent, then persist its findings with optimistic locking.
        Args: session_id: Active session. customer_id: Customer identifier. request: Original request.
        Returns: Inventory agent findings.
        """
        state = _state_or_error(session_id); version = int(state['version'])
        trace.step_start('inventory_agent')
        result = str(inventory_agent(f"Customer ID: {customer_id}\nCustomer request: {request}\nRetrieve all relevant order/customer facts. Do not decide eligibility."))
        _update_workflow_state(session_id, {'inventory_agent': result}, version)
        trace.step_done('inventory_agent', version)
        return result

    @tool
    def route_to_policy_agent(session_id: str, request: str) -> str:
        """Run PolicyAgent parallel retrieval and save its synthesized findings.
        Args: session_id: Active session. request: Customer policy question.
        Returns: Grounded policy findings.
        """
        state = _state_or_error(session_id); version = int(state['version'])
        trace.step_start('policy_agent')
        result = str(policy_agent(request))
        _update_workflow_state(session_id, {'policy_agent': result}, version)
        trace.step_done('policy_agent', version)
        return result

    @tool
    def route_to_refund_agent(session_id: str, customer_id: str, request: str) -> str:
        """Run RefundAgent after inventory, then persist its decision.
        Args: session_id: Active session. customer_id: Customer identifier. request: Return request.
        Returns: Eligibility decision and any confirmed return reference.
        """
        state = _state_or_error(session_id); version = int(state['version'])
        if not state.get('inventory_agent'):
            raise RuntimeError('RefundAgent cannot run before InventoryAgent findings are saved.')
        trace.step_start('refund_agent')
        result = str(refund_agent(f"Session ID: {session_id}\nCustomer ID: {customer_id}\nCustomer request: {request}\nCheck inventory context first, apply the tier window, and initiate only an eligible return."))
        _update_workflow_state(session_id, {'refund_agent': result}, version)
        trace.step_done('refund_agent', version)
        return result

    @tool
    def route_to_communication_agent(session_id: str, customer_id: str,
                                     original_request: str) -> str:
        """Run CommunicationAgent as the final step and save its customer reply.
        Args: session_id: Active session. customer_id: Customer identifier. original_request: Original message.
        Returns: Final customer-facing response.
        """
        state = _state_or_error(session_id); version = int(state['version'])
        trace.step_start('communication_agent')
        result = str(communication_agent(
            f"Session ID: {session_id}\nCustomer ID: {customer_id}\nOriginal customer request: {original_request}\nUse all relevant WorkflowState findings to compose the final response."))
        _update_workflow_state(session_id, {'communication_agent': result}, version)
        trace.step_done('communication_agent', version)
        return result

    @tool
    def initialize_session(session_id: str, customer_id: str) -> str:
        """Create the shared workflow record before processing a request.
        Args: session_id: Unique session identifier. customer_id: Customer identifier.
        Returns: Initialization confirmation and current workflow version.
        """
        try:
            state = _create_workflow_state(session_id, customer_id)
        except dynamodb.meta.client.exceptions.ConditionalCheckFailedException:
            state = _read_workflow_state(session_id)
            if not state:
                raise
            if state.get('customer_id') != customer_id:
                raise ValueError('Session ID is already associated with a different customer.')
        return f"Session {session_id} initialized (WorkflowState version {state['version']})."

    return Agent(model=model, system_prompt=system_prompt,
                 tools=[initialize_session, route_to_inventory_agent, route_to_policy_agent,
                        route_to_refund_agent, route_to_communication_agent])


# ═══════════════════════════════════════════════════════
#  AGENT GRAPH HELPERS
# ═══════════════════════════════════════════════════════

def _apply_guardrail(agents: list) -> None:
    """
    Attach the Bedrock Guardrail (Task 3) to every agent's BedrockModel.
    Guardrails are enforced per model invocation, so once GUARDRAIL_ID /
    GUARDRAIL_VERSION are known (in .env locally, as runtime environment
    variables when deployed) every agent in the graph runs behind the
    guardrail - no change to the agents themselves is needed.
    """
    guardrail_id      = config.GUARDRAIL_ID
    guardrail_version = config.GUARDRAIL_VERSION
    if not guardrail_id or not guardrail_version:
        return
    for agent in agents:
        model = getattr(agent, 'model', None)
        if model is not None and hasattr(model, 'update_config'):
            model.update_config(guardrail_id=guardrail_id,
                                guardrail_version=guardrail_version)


def build_agent_graph(verbose: bool = False) -> Agent:
    """Build all five agents, apply the guardrail, return the orchestrator."""
    def _ok(label):
        if verbose:
            print(f"  {_C.GRY}          {_C.OK}[OK]{_C.RESET}{_C.GRY}  {label}{_C.RESET}", flush=True)

    inventory_agent     = build_inventory_agent();     _ok('InventoryAgent')
    refund_agent        = build_refund_agent();        _ok('RefundAgent')
    policy_agent        = build_policy_agent();        _ok('PolicyAgent')
    communication_agent = build_communication_agent(); _ok('CommunicationAgent')
    orchestrator = build_orchestrator_agent(
        inventory_agent, refund_agent, policy_agent, communication_agent
    )
    _ok('Orchestrator')
    _apply_guardrail([inventory_agent, refund_agent, policy_agent,
                      communication_agent, orchestrator])
    if verbose and config.GUARDRAIL_ID:
        print(f"  {_C.GRY}          Guardrail {config.GUARDRAIL_ID} "
              f"(v{config.GUARDRAIL_VERSION}) attached to all agents{_C.RESET}")
    return orchestrator


# ═══════════════════════════════════════════════════════
#  DEPLOYMENT TOOLING - AgentCore CLI
#
#  The runtime is deployed with the AgentCore CLI (`agentcore`, npm package
#  @aws/agentcore - https://github.com/aws/agentcore-cli) through the
#  pre-written helper src/agentcore_cli.py:
#
#    agentcore_cli.stage_runtime_code()   copies this file, its helper modules
#                                         and config.py to build/runtime/ with a
#                                         pyproject.toml of the runtime deps
#    agentcore_cli.configure_runtime()    writes the runtime settings (network
#                                         mode, protocol, execution role, env
#                                         vars) to agentcore/agentcore.json
#    agentcore_cli.deploy()               runs `agentcore deploy -y`: the CLI
#                                         downloads arm64 / Python 3.12 wheels
#                                         with uv, zips them with the code
#                                         (direct code deployment) and creates
#                                         or updates the runtime via CDK
#    agentcore_cli.deployed_runtime_arn() reads the ARN the CLI recorded
#
#  Inside the runtime this same file is the entry point: it is started with
#  no command-line argument and serves HTTP (see run_serve). The marker file
#  written next to it by stage_runtime_code() tells __main__ to do so.
# ═══════════════════════════════════════════════════════

_RUNTIME_MARKER = '.agentcore-runtime'         # written by agentcore_cli.stage_runtime_code()
_SRC_DIR        = os.path.dirname(os.path.abspath(__file__))


# ═══════════════════════════════════════════════════════
#  TASK 3 - AGENTCORE DEPLOYMENT + GUARDRAILS
# ═══════════════════════════════════════════════════════

def create_guardrail() -> tuple[str, str]:
    """
    Create a Bedrock Guardrail for enterprise safety enforcement.

    Blocks harmful content, PII exposure, off-topic subjects, and profanity.
    Returns (guardrail_id, guardrail_version).
    """
    bedrock_client = boto3.client('bedrock', region_name=config.AWS_REGION)

    # Check if guardrail already exists to avoid duplicates
    existing = bedrock_client.list_guardrails()
    for g in existing.get('guardrails', []):
        if g['name'] == config.GUARDRAIL_NAME:
            guardrail_id = g['id']
            versions = bedrock_client.list_guardrails(guardrailIdentifier=g['arn'])
            summaries = versions.get('guardrails', [])
            numbered = [v for v in summaries if str(v.get('version', '')).isdigit()]
            if numbered:
                latest = max(numbered, key=lambda item: int(item['version']))
                guardrail_version = str(latest['version'])
                print(f"Guardrail already exists: {guardrail_id} (version: {guardrail_version})")
                return guardrail_id, guardrail_version
            published = bedrock_client.create_guardrail_version(
                guardrailIdentifier=guardrail_id,
                description='Published production version for NovaMart customer support.'
            )
            guardrail_version = str(published['version'])
            print(f"Existing guardrail published: {guardrail_id} (version: {guardrail_version})")
            return guardrail_id, guardrail_version

    content_filters = [
        {'type': kind, 'inputStrength': strength, 'outputStrength': strength}
        for kind, strength in [('SEXUAL', 'HIGH'), ('VIOLENCE', 'HIGH'), ('HATE', 'HIGH'),
                               ('INSULTS', 'MEDIUM'), ('MISCONDUCT', 'MEDIUM')]
    ]
    pii_entities = [
        {'type': 'CREDIT_DEBIT_CARD_NUMBER', 'action': 'BLOCK'},
        {'type': 'US_SOCIAL_SECURITY_NUMBER', 'action': 'BLOCK'},
        {'type': 'EMAIL', 'action': 'ANONYMIZE'},
        {'type': 'PHONE', 'action': 'ANONYMIZE'},
    ]
    topic_definitions = {
        'competitor products': 'Requests to recommend, compare, promote, or provide product details for products sold by NovaMart competitors.',
        'pricing negotiations': 'Requests to haggle, bargain, negotiate, or change a price advertised by NovaMart. This does not include arithmetic, totals, or discount calculations using a price and discount rate already specified by the customer.',
        'legal threats': 'Threats or demands to sue, initiate legal proceedings, or use legal action against NovaMart.',
    }
    topics = [{'name': name.title(), 'definition': topic_definitions.get(name, name), 'type': 'DENY'}
              for name in config.GUARDRAIL_BLOCKED_TOPICS]
    response = bedrock_client.create_guardrail(
        name=config.GUARDRAIL_NAME,
        description='NovaMart enterprise customer-support safety guardrail.',
        contentPolicyConfig={'filtersConfig': content_filters},
        sensitiveInformationPolicyConfig={'piiEntitiesConfig': pii_entities},
        topicPolicyConfig={'topicsConfig': topics, 'tierConfig': {'tierName': 'STANDARD'}},
        wordPolicyConfig={'managedWordListsConfig': [{'type': 'PROFANITY'}]},
        blockedInputMessaging='I’m sorry, but I can’t help with that request. Please ask a NovaMart support question.',
        blockedOutputsMessaging='I’m sorry, but I can’t provide that response. Please contact NovaMart support for further assistance.',
        crossRegionConfig={'guardrailProfileIdentifier': 'us.guardrail.v1:0'},
    )
    guardrail_id = response['guardrailId']
    version_response = bedrock_client.create_guardrail_version(
        guardrailIdentifier=guardrail_id,
        description='Published production version for NovaMart customer support.'
    )
    guardrail_version = str(version_response['version'])
    print(f'Guardrail created and published: {guardrail_id} (version {guardrail_version})')
    return guardrail_id, guardrail_version


def deploy_to_agentcore_runtime(
    orchestrator_agent: Agent,
    guardrail_id: str,
    guardrail_version: str
) -> str:
    """
    Deploy the multi-agent system to Amazon Bedrock AgentCore Runtime with
    the AgentCore CLI (src/agentcore_cli.py wraps it).

    AgentCore does not serialize Python objects, so `orchestrator_agent` is
    not uploaded directly. Instead the pre-written staging step copies this
    file, which doubles as the HTTP entry point (see run_serve), together
    with its helper modules and config.py to build/runtime/. `agentcore
    deploy` then packages that directory with arm64 dependencies and creates
    or updates the runtime ("direct code deployment"). Re-running is safe:
    an unchanged runtime is left alone, a changed one is updated in place.

    The guardrail is attached by environment variables: inside the runtime
    build_agent_graph() reads GUARDRAIL_ID / GUARDRAIL_VERSION and applies
    them to every agent's model (see _apply_guardrail), exactly as `test`
    and `chat` do locally.

    Returns:
        The AgentCore Runtime ARN
    """
    import agentcore_cli

    runtime_name = config.AGENTCORE_RUNTIME_NAME
    print(f"  AWS Account: {config.ACCOUNT_ID}  |  Region: {config.AWS_REGION}")
    print(f"  Runtime: {runtime_name}  |  CLI project: agentcore/agentcore.json "
          f"(stack {agentcore_cli.stack_name()})")
    previous_arn = agentcore_cli.deployed_runtime_arn()
    if previous_arn:
        print(f"  Runtime already deployed - updating it: {previous_arn}")

    # Stage the code the CLI packages (src modules + config.py + pyproject.toml).
    agentcore_cli.stage_runtime_code()

    # 1. Build the runtime environment variables dict `runtime_env` with:
    #      AWS_REGION, PROJECT_NAME (config.AWS_REGION / config.PROJECT_NAME),
    #      RETURNS_KB_ID, SHIPPING_KB_ID, WARRANTY_KB_ID (from config),
    #      AGENT_LOG_GROUP (config.AGENT_LOG_GROUP), and the guardrail
    #      (GUARDRAIL_ID = guardrail_id, GUARDRAIL_VERSION = guardrail_version)
    # 2. Write the runtime settings to agentcore/agentcore.json with
    #      agentcore_cli.configure_runtime(env_vars=runtime_env,
    #                                      network_mode='PUBLIC',
    #                                      protocol='HTTP',
    #                                      execution_role_arn=config.AGENTCORE_ROLE_ARN)
    # 3. Deploy:  agentcore_cli.deploy()        (runs `agentcore deploy -y`)
    # 4. Read the ARN the CLI recorded:
    #      runtime_arn = agentcore_cli.deployed_runtime_arn()
    runtime_env = {
        'AWS_REGION': config.AWS_REGION,
        'PROJECT_NAME': config.PROJECT_NAME,
        'RETURNS_KB_ID': config.RETURNS_KB_ID,
        'SHIPPING_KB_ID': config.SHIPPING_KB_ID,
        'WARRANTY_KB_ID': config.WARRANTY_KB_ID,
        'AGENT_LOG_GROUP': config.AGENT_LOG_GROUP,
        'GUARDRAIL_ID': guardrail_id,
        'GUARDRAIL_VERSION': str(guardrail_version),
    }
    missing = [key for key in ('RETURNS_KB_ID', 'SHIPPING_KB_ID', 'WARRANTY_KB_ID') if not runtime_env[key]]
    if missing:
        raise RuntimeError('Cannot deploy runtime; missing Knowledge Base IDs: ' + ', '.join(missing))
    agentcore_cli.configure_runtime(env_vars=runtime_env, network_mode='PUBLIC',
                                    protocol='HTTP', execution_role_arn=config.AGENTCORE_ROLE_ARN)
    agentcore_cli.deploy()
    runtime_arn = agentcore_cli.deployed_runtime_arn()

    if not runtime_arn:
        raise RuntimeError('AgentCore CLI deployment completed but no runtime ARN was recorded.')

    # Wait for the runtime to become READY and return its ARN.
    print(f"  Runtime deployed: {runtime_arn}")
    print("  Waiting for runtime status READY", end='', flush=True)
    wait_for_runtime_ready(agentcore_control, runtime_arn.split('/')[-1])
    print(' ready.')
    return runtime_arn


# ═══════════════════════════════════════════════════════
#  TASK 4 - MEMORY
# ═══════════════════════════════════════════════════════

def configure_memory(runtime_arn: str) -> str:
    """
    Create an AgentCore Memory resource for session-scoped conversational
    context. Uses the SESSION_SUMMARY (summaryMemoryStrategy) strategy with
    7-day event retention.

    Returns:
        The memory resource ARN
    """
    memory_name = config.MEMORY_NAME
    existing = agentcore_control.list_memories()
    for m in existing.get('memories', []):
        if m['id'].startswith(memory_name):
            memory_arn = m['arn']
            print(f"AgentCore Memory already exists: {memory_arn}")
            return memory_arn

    # Use agentcore_control.create_memory() with:
    #   - name (memory_name) and a description
    #   - eventExpiryDuration = 7   (days)
    #   - memoryStrategies = [{'summaryMemoryStrategy': {
    #         'name': 'SessionSummary',
    #         'namespaces': ['/summaries/{actorId}/{sessionId}']}}]
    #   - clientToken (e.g. str(uuid.uuid4())) for idempotency
    # Store the API response in `response`.
    response = agentcore_control.create_memory(
        name=memory_name,
        description='NovaMart customer-support session summaries with seven-day event retention.',
        eventExpiryDuration=7,
        memoryStrategies=[{'summaryMemoryStrategy': {
            'name': 'SessionSummary',
            'namespaces': ['/summaries/{actorId}/{sessionId}']
        }}],
        clientToken=str(uuid.uuid4()),
    )

    # Wait until the memory resource is ACTIVE and return its ARN.
    memory = response['memory']
    print(f"  Memory created: {memory['arn']}  (status: {memory['status']})")
    print("  Waiting for memory status ACTIVE", end='', flush=True)
    deadline = time.time() + 300
    while memory['status'] != 'ACTIVE' and time.time() < deadline:
        time.sleep(10)
        print('.', end='', flush=True)
        memory = agentcore_control.get_memory(memoryId=memory['id'])['memory']
        if memory['status'] == 'FAILED':
            raise RuntimeError(f"Memory creation failed: {memory.get('failureReason')}")
    print(' ready.' if memory['status'] == 'ACTIVE' else f" status {memory['status']}")
    return memory['arn']


# ═══════════════════════════════════════════════════════
#  TASK 6 - OBSERVABILITY
# ═══════════════════════════════════════════════════════

def configure_observability(runtime_arn: str) -> None:
    """
    Configure observability for the deployed agent:
    - Agent logs → CloudWatch Logs at INFO level (config.AGENT_LOG_GROUP)
    - Execution traces → AWS X-Ray at 100% sampling

    The loggingConfiguration built here is applied by
    apply_observability_config() (agent_observability.py):
      cloudWatchConfig -> log group created; runtime env AGENT_LOG_GROUP /
                          AGENT_LOG_LEVEL so the deployed agent ships its logs there
      xRayConfig       -> CloudWatch Transaction Search enabled with the given
                          sampling percentage; runtime env AGENT_TRACING_ENABLED /
                          AGENT_TRACE_SAMPLING_RATE
    """
    # logging_configuration = {
    #     'cloudWatchConfig': {'logGroupName': config.AGENT_LOG_GROUP,
    #                          'logLevel': 'INFO', 'enabled': True},
    #     'xRayConfig':       {'enabled': True, 'samplingRate': 1.0},
    # }
    # Then apply it:  summary = apply_observability_config(runtime_arn, logging_configuration)
    # Wrap the call in try/except - on success print the CloudWatch log group
    # and the X-Ray sampling rate; on exception print
    #   "[Note] Observability configuration failed: <e>"

    logging_configuration = {
        'cloudWatchConfig': {
            'logGroupName': config.AGENT_LOG_GROUP,
            'logLevel': 'INFO',
            'enabled': True,
        },
        'xRayConfig': {'enabled': True, 'samplingRate': 1.0},
    }
    try:
        summary = apply_observability_config(runtime_arn, logging_configuration)
        print(f"  CloudWatch logging enabled: {summary.get('log_group', config.AGENT_LOG_GROUP)}")
        print('  X-Ray tracing enabled: 100% sampling')
    except Exception as exc:
        logger.exception('Observability configuration failed')
        print(f'[Note] Observability configuration failed: {exc}')


# ═══════════════════════════════════════════════════════
#  AGENTCORE GATEWAY DEPLOYMENT
#
#  Production equivalent of in-process @tool functions.
#  Registers Lambda-backed tools on a managed MCP endpoint so tools
#  can be independently deployed, versioned, and discovered at runtime.
#
#  Deployment pattern:
#    Local dev  → LambdaGateway + gateway.register_target(...)
#    Production → deploy_agentcore_gateway() using real AWS API
#
#  Requires Lambda tool functions to be deployed separately.
#  Set ORDERS_FUNCTION, POLICY_FUNCTION, CUSTOMERS_FUNCTION in .env
#  to the deployed Lambda function names.
# ═══════════════════════════════════════════════════════

# Lambda function names for gateway tool backends (set in .env after deploying)
_ORDERS_FUNCTION = os.environ.get('ORDERS_FUNCTION', '')
_POLICY_FUNCTION = os.environ.get('POLICY_FUNCTION', '')
_CUSTOMERS_FUNCTION = os.environ.get('CUSTOMERS_FUNCTION', '')


def _gw_get_function_arn(function_name: str) -> str:
    """Resolve a Lambda function name to its full ARN."""
    lambda_client = boto3.client('lambda', region_name=config.AWS_REGION)
    resp = lambda_client.get_function(FunctionName=function_name)
    return resp['Configuration']['FunctionArn']


def _gw_stack_uuid() -> str:
    """Return the short UUID from the project CloudFormation stack ID.
    Gives the gateway a stable name so re-runs never hit ConflictException."""
    cf = boto3.client('cloudformation', region_name=config.AWS_REGION)
    stacks = cf.describe_stacks(StackName=config.PROJECT_NAME)
    stack_id = stacks['Stacks'][0]['StackId']
    full_uuid = stack_id.split('/')[-1]
    return full_uuid.split('-')[0]


def _gw_wait_for_ready(agentcore_ctrl, gateway_id: str, timeout: int = 120) -> str:
    """Poll until the gateway reaches READY status. Returns the gateway URL."""
    deadline = time.time() + timeout
    first    = True
    while time.time() < deadline:
        gw     = agentcore_ctrl.get_gateway(gatewayIdentifier=gateway_id)
        status = gw['status']
        if status == 'READY':
            if not first:
                print(' ready.')
            return gw.get('gatewayUrl', '')
        if 'FAILED' in status:
            print(f' failed: {status}')
            raise RuntimeError(f"Gateway {gateway_id} entered status {status}")
        if first:
            print('    Gateway provisioning (async — normal AWS behaviour)',
                  end='', flush=True)
            first = False
        print('.', end='', flush=True)
        time.sleep(5)
    raise TimeoutError(f"Gateway {gateway_id} not READY after {timeout}s")


def _gw_get_or_create(agentcore_ctrl, name: str, role_arn: str,
                       instructions: str) -> tuple[str, str]:
    """Create an AgentCore Gateway, or reuse it if it already exists."""
    try:
        gw = agentcore_ctrl.create_gateway(
            name=name,
            roleArn=role_arn,
            protocolType='MCP',
            authorizerType='NONE',
            protocolConfiguration={'mcp': {'instructions': instructions,
                                            'searchType': 'SEMANTIC'}},
        )
        gw_id  = gw['gatewayId']
        print(f'    Gateway ID  : {gw_id}')
        print(f'    Status      : {gw["status"]}')
        gw_url = _gw_wait_for_ready(agentcore_ctrl, gw_id)
        print(f'    Gateway URL : {gw_url}')
        return gw_id, gw_url
    except agentcore_ctrl.exceptions.ConflictException:
        print(f"    Gateway '{name}' already exists — reusing it.")
        gateways = agentcore_ctrl.list_gateways().get('items', [])
        existing = next((g for g in gateways if g['name'] == name), None)
        if not existing:
            raise RuntimeError(f"Gateway '{name}' not found after ConflictException")
        gw_id  = existing['gatewayId']
        print(f'    Gateway ID  : {gw_id}')
        gw_url = _gw_wait_for_ready(agentcore_ctrl, gw_id)
        print(f'    Gateway URL : {gw_url}')
        return gw_id, gw_url


def _gw_create_target(agentcore_ctrl, gateway_id: str, t: dict,
                       lambda_arn: str) -> None:
    """Register one Lambda target on the gateway. Skips if it already exists."""
    payload = dict(
        gatewayIdentifier=gateway_id,
        name=t['name'],
        description=t['description'],
        targetConfiguration={
            'mcp': {
                'lambda': {
                    'lambdaArn': lambda_arn,
                    'toolSchema': {
                        'inlinePayload': [{
                            'name':        t['tool_name'],
                            'description': t['tool_description'],
                            'inputSchema': {
                                'type': 'object',
                                'properties': {
                                    t['param_name']: {
                                        'type':        'string',
                                        'description': t['param_desc'],
                                    }
                                },
                                'required': [t['param_name']],
                            },
                        }]
                    },
                }
            }
        },
        credentialProviderConfigurations=[
            {'credentialProviderType': 'GATEWAY_IAM_ROLE'}
        ],
    )
    try:
        resp = agentcore_ctrl.create_gateway_target(**payload)
        print(f"    [{resp['status']:12s}] {t['name']} → target {resp['targetId']}")
    except agentcore_ctrl.exceptions.ConflictException:
        print(f"    [already exists] {t['name']} — skipped")


def deploy_agentcore_gateway() -> dict:
    """
    Create an AgentCore Gateway and register the NovaMart tool Lambda targets.

    Optional extension to the in-process @tool functions. Resolve configured
    Lambda functions first; if none exist, skip gateway creation. Otherwise
    create/reuse the gateway and submit its targets. Connecting agents to this
    MCP endpoint requires separate integration; this starter uses in-process tools.

    Requires Lambda tool functions to be deployed via a separate stack.
    Set ORDERS_FUNCTION, POLICY_FUNCTION, CUSTOMERS_FUNCTION in .env.

    Returns:
        A SKIPPED result with a reason, or gateway details and target count.
    """
    targets = [
        {
            'name':             'orders-api',
            'description':      'Look up order details, status, and return eligibility for a customer',
            'function':         _ORDERS_FUNCTION,
            'tool_name':        'check_order_status',
            'tool_description': 'Check order status and return eligibility for a specific order',
            'param_name':       'order_id',
            'param_desc':       'Order ID (e.g. ORD-27176)',
        },
        {
            'name':             'policy-api',
            'description':      'Retrieve return, shipping, and warranty policy text from knowledge bases',
            'function':         _POLICY_FUNCTION,
            'tool_name':        'search_policies',
            'tool_description': 'Search all policy knowledge bases for a customer query',
            'param_name':       'query',
            'param_desc':       'Customer question about returns, shipping, or warranty',
        },
        {
            'name':             'customers-api',
            'description':      'Look up customer tier (Standard or Premium) and account details',
            'function':         _CUSTOMERS_FUNCTION,
            'tool_name':        'get_customer_tier',
            'tool_description': 'Get customer tier and account information by customer ID',
            'param_name':       'customer_id',
            'param_desc':       'Customer ID (e.g. CUST-001)',
        },
    ]

    # Resolve optional Lambda targets before creating any gateway resources.
    available = []
    for target in targets:
        if not target['function']:
            continue
        try:
            available.append((target, _gw_get_function_arn(target['function'])))
        except ClientError as exc:
            if exc.response['Error']['Code'] != 'ResourceNotFoundException':
                raise
            print(f"    [Skipped] {target['name']}: Lambda function not found")

    if not available:
        return {'status': 'SKIPPED', 'reason': 'No configured Lambda tool functions are available.'}

    agentcore_ctrl = boto3.client('bedrock-agentcore-control',
                                   region_name=config.AWS_REGION)

    try:
        gw_uuid = _gw_stack_uuid()
    except Exception:
        gw_uuid = config.PROJECT_NAME

    gw_name = f"novamart-support-{gw_uuid}"
    print(f"  Calling create_gateway (name: {gw_name})...")
    gateway_id, gateway_url = _gw_get_or_create(
        agentcore_ctrl, gw_name, config.AGENTCORE_ROLE_ARN,
        "NovaMart customer support gateway. Provides order lookup, "
        "policy search, and customer tier tools.",
    )

    print(f"\n  Registering {len(available)} Gateway targets...")
    for target, lambda_arn in available:
        _gw_create_target(agentcore_ctrl, gateway_id, target, lambda_arn)

    return {'gateway_id': gateway_id, 'gateway_url': gateway_url,
            'status': 'TARGETS_SUBMITTED', 'target_count': len(available)}



# ═══════════════════════════════════════════════════════
#  RUNTIME INVOCATION
# ═══════════════════════════════════════════════════════

def invoke_agent(session_id: str, customer_id: str, user_message: str) -> dict:
    """
    Invoke the deployed agent via AgentCore Runtime (see run_serve).

    AgentCore requires runtimeSessionId to be at least 33 characters, so the
    short project session id is embedded in a longer, unique runtime session id.
    """
    if not config.AGENTCORE_RUNTIME_ARN:
        raise RuntimeError("AGENTCORE_RUNTIME_ARN is not set - run the deploy command first")

    runtime_session_id = f"{session_id}-{uuid.uuid4().hex}"     # >= 33 chars
    payload = json.dumps({
        'prompt':      user_message,
        'session_id':  session_id,
        'customer_id': customer_id,
    })
    response = agentcore_client.invoke_agent_runtime(
        agentRuntimeArn=config.AGENTCORE_RUNTIME_ARN,
        runtimeSessionId=runtime_session_id,
        contentType='application/json',
        accept='application/json',
        payload=payload,
    )
    body = response['response'].read()
    try:
        return json.loads(body)
    except (TypeError, ValueError):
        return {'result': body.decode('utf-8', errors='replace') if isinstance(body, bytes) else str(body)}


# ═══════════════════════════════════════════════════════
#  DEPLOYMENT ENTRY POINT
# ═══════════════════════════════════════════════════════

def _save_env_values(values: dict) -> None:
    """Persist deployment outputs to .env without discarding existing settings."""
    env_path = os.path.join(os.path.dirname(os.path.dirname(os.path.abspath(__file__))), '.env')
    if not os.path.exists(env_path):
        example = os.path.join(os.path.dirname(env_path), '.env.example')
        if os.path.exists(example):
            with open(example, encoding='utf-8') as source:
                lines = source.readlines()
        else:
            lines = []
    else:
        with open(env_path, encoding='utf-8') as source:
            lines = source.readlines()
    pending = {str(key): str(value) for key, value in values.items()}
    output = []
    for line in lines:
        stripped = line.strip()
        if stripped and not stripped.startswith('#') and '=' in stripped:
            key = stripped.split('=', 1)[0].strip()
            if key in pending:
                output.append(f'{key}={pending.pop(key)}\n')
                continue
        output.append(line)
    for key, value in pending.items():
        output.append(f'{key}={value}\n')
    with open(env_path, 'w', encoding='utf-8') as target:
        target.writelines(output)


def deploy_all():
    """Full deployment pipeline. Run after completing all tasks."""
    print("\n" + "="*60)
    print("  Deploying Enterprise Multi-Agent System")
    print("="*60 + "\n")

    # Fail fast if the AgentCore CLI (used by Steps 3 and 5) is missing.
    import agentcore_cli
    print(f"AgentCore CLI: {agentcore_cli.cli_version()} ({agentcore_cli.cli_path()})\n")

    print("Step 1/6: Building agent graph...")
    inventory_agent     = build_inventory_agent()
    refund_agent        = build_refund_agent()
    policy_agent        = build_policy_agent()
    communication_agent = build_communication_agent()
    orchestrator = build_orchestrator_agent(
        inventory_agent, refund_agent, policy_agent, communication_agent
    )
    print("  All 5 agents initialized\n")

    print("Step 2/6: Creating Bedrock Guardrail...")
    guardrail_id, guardrail_version = create_guardrail()
    print()

    print("Step 3/6: Deploying to AgentCore Runtime...")
    runtime_arn = deploy_to_agentcore_runtime(orchestrator, guardrail_id, guardrail_version)
    _save_env_values({
        'AGENTCORE_RUNTIME_ARN': runtime_arn,
        'GUARDRAIL_ID': guardrail_id,
        'GUARDRAIL_VERSION': guardrail_version,
    })
    print('  Saved runtime ARN and published Guardrail configuration to .env')
    print()

    print("Step 4/6: Configuring Memory...")
    memory_arn = configure_memory(runtime_arn)
    print()

    print("Step 5/6: Configuring Observability...")
    configure_observability(runtime_arn)
    print()

    print("Step 6/6: Deploying AgentCore Gateway...")
    try:
        gw = deploy_agentcore_gateway()
        if gw['status'] == 'SKIPPED':
            print(f"  [Skipped] Gateway: {gw['reason']}")
        else:
            print(f"  Gateway URL : {gw['gateway_url']}")
            print("  Lambda targets submitted; connect an MCP client separately to use them.")
    except Exception as e:
        print(f"  [Note] Optional Gateway deployment failed: {e}")
        print(f"  (Deploy Lambda tool functions and set ORDERS_FUNCTION etc. in .env to enable)")
    print()

    print("="*60)
    print("  Deployment Complete!")
    print("="*60)
    print(f"\n  Add these to your .env file:")
    print(f"  AGENTCORE_RUNTIME_ARN={runtime_arn}")
    print(f"  GUARDRAIL_ID={guardrail_id}")
    print(f"  GUARDRAIL_VERSION={guardrail_version}\n")
    print(f"  Then try the deployed runtime:")
    print(f"  python src/agent_orchestrator.py invoke \"What is the return policy for premium customers?\"")
    print(f"  or with the CLI:  agentcore invoke \"What is the return policy for premium customers?\"")
    print(f"  (agentcore status / agentcore logs show the deployed runtime and its logs)\n")
    return runtime_arn, guardrail_id


# ═══════════════════════════════════════════════════════
#  LOCAL TEST SCENARIOS
# ═══════════════════════════════════════════════════════

# Order IDs match infrastructure/seed_data.py.
TEST_CASES = [
    ("CUST-001", "I want to return my wireless headphones from order ORD-27176"),
    ("CUST-002", "What is the return policy for premium customers?"),
    ("CUST-003", "How much would 5 items at $29.99 be with a 10% discount?"),
]

# Test customers shown by the chat command. Data matches seed_data.py.
TEST_CUSTOMERS = [
    ("CUST-001", "Alice Johnson", "Premium",  "ORD-27176", "Wireless Headphones Pro"),
    ("CUST-002", "Bob Smith",     "Standard", "ORD-28001", "Mechanical Keyboard K2"),
    ("CUST-003", "Carol Davis",   "Premium",  "ORD-29001", "Laptop UltraBook 14"),
    ("CUST-004", "David Lee",     "Standard", "ORD-30001", "Phone Case Slim"),
]


def run_test_scenarios() -> None:
    """Run the three scenarios locally; every request is traced to X-Ray."""
    print("Running local agent test...")
    setup_logging(to_cloudwatch=True)
    orchestrator = build_agent_graph()

    for customer_id, query in TEST_CASES:
        session_id = str(uuid.uuid4())[:8]
        print(f"\n{'─'*60}")
        print(f"Session: {session_id} | Customer: {customer_id}")
        print(f"Query: {query}")
        prompt = f"[Session ID: {session_id}] [Customer ID: {customer_id}] {query}"
        with tracer.trace_request(session_id, customer_id, query):
            response = orchestrator(prompt)
        print(f"Response: {response}")
        print_trace_hint()
    flush_logs()


def run_chat() -> None:
    """Interactive terminal chat - educational mode."""
    W = _C.W

    # ── Welcome banner ────────────────────────────────────────────────
    print()
    print(f"  {_C.GRY}{'=' * W}{_C.RESET}")
    print(f"  {_C.ORCH}{_C.BOLD}{'NovaMart -- Multi-Agent Customer Support':^{W}}{_C.RESET}")
    print(f"  {_C.GRY}{'Strands Agents SDK  +  Amazon Bedrock AgentCore':^{W}}{_C.RESET}")
    print(f"  {_C.GRY}{'=' * W}{_C.RESET}")

    # ── Test customers ────────────────────────────────────────────────
    print()
    print(f"  {_C.GRY}{'─' * W}{_C.RESET}")
    print(f"  {_C.BOLD}Test Customers{_C.RESET}")
    print(f"  {_C.GRY}{'─' * W}{_C.RESET}")
    print(f"  {_C.GRY}{'ID':<10}  {'Name':<18}  {'Tier':<10}  {'Order':<12}  Product{_C.RESET}")
    print(f"  {_C.GRY}{'─'*8}  {'─'*16}  {'─'*8}  {'─'*10}  {'─'*20}{_C.RESET}")
    for cid, name, tier, order, product in TEST_CUSTOMERS:
        tier_col = _C.INV if tier == 'Premium' else _C.GRY
        print(f"  {_C.BOLD}{cid}{_C.RESET}  {name:<18}  "
              f"{tier_col}{tier:<10}{_C.RESET}  {order}  {product}")
    print(f"  {_C.GRY}{'─' * W}{_C.RESET}")
    print()

    customer_id = (
        input(f"  Enter Customer ID (default: CUST-001): ").strip()
        or "CUST-001"
    )
    session_id  = str(uuid.uuid4())[:8]
    print()
    print(f"  {_C.GRY}Session  : {_C.RESET}{_C.BOLD}{session_id}{_C.RESET}")
    print(f"  {_C.GRY}Customer : {_C.RESET}{_C.BOLD}{customer_id}{_C.RESET}")
    print(f"  {_C.GRY}Type a question and press Enter.  Type 'quit' to exit.{_C.RESET}")
    print()

    # ── Build agents and show initialization order.
    print(f"  {_C.GRY}[SYSTEM]  Initializing agent graph...{_C.RESET}")
    setup_logging(to_cloudwatch=True)
    orchestrator = build_agent_graph(verbose=True)
    print(f"  {_C.GRY}[SYSTEM]  All 5 agents ready.{_C.RESET}")
    print()

    # ── Conversation loop ─────────────────────────────────────────────
    while True:
        try:
            user_input = input(
                f"  {_C.BOLD}You >{_C.RESET} "
            ).strip()
        except (EOFError, KeyboardInterrupt):
            print(f"\n  {_C.GRY}Session ended.{_C.RESET}")
            break

        if not user_input:
            continue
        if user_input.lower() in ('quit', 'exit', 'q'):
            print(f"  {_C.GRY}Session ended.{_C.RESET}")
            break

        prompt  = (f"[Session ID: {session_id}] "
                   f"[Customer ID: {customer_id}] {user_input}")
        t0_turn = time.time()

        # ── Install proxy, run orchestrator (traced), restore stdout ───
        trace.new_turn()
        sys.stdout = _trace_writer
        try:
            with tracer.trace_request(session_id, customer_id, user_input):
                response = orchestrator(prompt)
        finally:
            sys.stdout = _real_stdout   # always restore, even on exception

        elapsed = time.time() - t0_turn

        # ── Resolve the final customer-facing text ────────────────────
        final_state = _read_workflow_state(session_id) or {}
        comm_result = final_state.get('communication_agent', '')
        text = _strip_xml_tags(comm_result or str(response))

        # ── DynamoDB workflow state summary ───────────────────────────
        trace.summary(session_id, elapsed)

        # ── Final customer-facing response ────────────────────────────
        print()
        print(f"  {_C.GRY}{'=' * W}{_C.RESET}")
        print(f"  {_C.COM}{_C.BOLD}AGENT RESPONSE{_C.RESET}")
        print(f"  {_C.GRY}{'=' * W}{_C.RESET}")
        for line in text.splitlines():
            print(f"  {line}")
        print(f"  {_C.GRY}{'=' * W}{_C.RESET}")
        if tracer.last_trace_id:
            print(f"  {_C.GRY}X-Ray trace : {tracer.last_trace_id}"
                  f"{'' if tracer.last_published else '  (not published)'}{_C.RESET}")
        print()
    flush_logs()


def run_invoke(message: str, customer_id: str = "CUST-001") -> None:
    """Send one message to the deployed AgentCore Runtime and print the reply."""
    session_id = str(uuid.uuid4())[:8]
    print(f"Invoking {config.AGENTCORE_RUNTIME_ARN}")
    print(f"Session: {session_id} | Customer: {customer_id}")
    print(f"Query: {message}\n")
    result = invoke_agent(session_id, customer_id, message)
    print(f"Response: {result.get('result', result)}")
    if result.get('trace_id'):
        print(f"X-Ray trace: {result['trace_id']}")


def run_serve() -> None:
    """
    HTTP entry point executed inside Amazon Bedrock AgentCore Runtime.

    BedrockAgentCoreApp (bedrock-agentcore SDK) exposes the contract the
    runtime expects - POST /invocations and GET /ping on port 8080 - and hands
    each request payload to the function decorated with @app.entrypoint.

    Request payload (see invoke_agent):
        {"prompt": "<customer message>", "customer_id": "CUST-001", "session_id": "abc12345"}
    Response:
        {"result": "<final customer-facing text>", "session_id": ..., "trace_id": ...}

    The five-agent graph is built once (first request) and reused. Guardrail,
    tracing and logging are applied exactly as in the local test/chat modes,
    from the runtime's environment variables.
    """
    from bedrock_agentcore import BedrockAgentCoreApp

    os.environ.setdefault('AGENT_RUNTIME_MODE', 'agentcore-runtime')
    if os.environ.get('AGENT_LOG_GROUP') and 'AGENT_LOG_TO_CLOUDWATCH' not in os.environ:
        os.environ['AGENT_LOG_TO_CLOUDWATCH'] = 'true'

    app   = BedrockAgentCoreApp()
    lock  = threading.Lock()
    graph = {}

    def _orchestrator():
        with lock:
            if 'agent' not in graph:
                setup_logging()
                graph['agent'] = build_agent_graph()
        return graph['agent']

    @app.entrypoint
    def invoke(payload, context=None):
        payload     = payload or {}
        prompt      = payload.get('prompt') or payload.get('message') or ''
        customer_id = payload.get('customer_id') or 'CUST-001'
        session_id  = payload.get('session_id') or (
            getattr(context, 'session_id', None) or uuid.uuid4().hex)[:8]
        if not prompt:
            return {'error': "payload must include 'prompt'"}

        enriched = f"[Session ID: {session_id}] [Customer ID: {customer_id}] {prompt}"
        with tracer.trace_request(session_id, customer_id, prompt):
            response = _orchestrator()(enriched)

        state = _read_workflow_state(session_id) or {}
        text  = _strip_xml_tags(state.get('communication_agent', '') or str(response))
        flush_logs()
        return {'result': text, 'session_id': session_id, 'customer_id': customer_id,
                'trace_id': tracer.last_trace_id}

    app.run()


if __name__ == '__main__':
    command = sys.argv[1] if len(sys.argv) > 1 else ''

    # Inside the AgentCore Runtime package (marker file next to this script)
    # the entry point is started without arguments -> serve HTTP.
    if not command and os.path.exists(os.path.join(_SRC_DIR, _RUNTIME_MARKER)):
        command = 'serve'

    if command == 'deploy':
        deploy_all()

    elif command == 'serve':
        run_serve()

    elif command == 'test':
        run_test_scenarios()

    elif command == 'chat':
        run_chat()

    elif command == 'invoke':
        if len(sys.argv) < 3:
            print('Usage: python src/agent_orchestrator.py invoke "<message>" [CUSTOMER_ID]')
            sys.exit(1)
        run_invoke(sys.argv[2], sys.argv[3] if len(sys.argv) > 3 else "CUST-001")

    else:
        print("Usage:")
        print("  python src/agent_orchestrator.py deploy           # Deploy to AgentCore (Tasks 3-6)")
        print("  python src/agent_orchestrator.py test             # Run the 3 test scenarios locally")
        print("  python src/agent_orchestrator.py chat             # Interactive terminal chat")
        print("  python src/agent_orchestrator.py invoke \"<msg>\"   # Call the deployed runtime")
        print("  python src/agent_orchestrator.py serve            # HTTP server (used inside AgentCore Runtime)")
