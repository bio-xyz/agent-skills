#!/usr/bin/env python3
"""
BIOS Deep Research client.

Supports two wallet backends with auto-detection:
1. WALLET_PRIVATE_KEY
2. CDP_API_KEY_ID + CDP_API_KEY_SECRET + CDP_WALLET_ADDRESS

This script uses the currently working x402 v2 flow for BIOS:
- request research job
- parse payment requirements from JSON body or PAYMENT-REQUIRED header
- create PAYMENT-SIGNATURE via a small Node helper using @x402/core/@x402/evm
- authenticate poll requests via SIWX (SIWE wallet signatures)
- retry the request
- optionally poll until completion

Requirements in this environment:
- python3
- node + npm
- eth_account (pip install eth-account)

Usage examples:
python3 research.py --dry-run "What is NAD+?"
python3 research.py --mode steering --no-poll "What are the most common causes of freshwater fish die-offs in lakes and rivers?"
"""

from __future__ import annotations

import argparse
import base64
import json
import os
import shutil
import subprocess
import sys
import tempfile
import time
import urllib.error
import urllib.request
from pathlib import Path

from eth_account import Account
from eth_account.messages import encode_defunct

BASE_URL = os.environ.get("BIOS_X402_BASE_URL", "https://x402.ai.bio.xyz")
SCRIPT_DIR = Path(__file__).resolve().parent
NODE_DEPS_DIR = SCRIPT_DIR / ".node-deps"
SIGNER_JS = SCRIPT_DIR / "research_signer.mjs"
PAYMENT_REQUIRED_HEADERS = ["PAYMENT-REQUIRED", "payment-required"]


def _get_wallet_account() -> Account:
    pk = os.environ.get("WALLET_PRIVATE_KEY")
    if pk:
        return Account.from_key(pk)
    raise RuntimeError(
        "SIWX signing requires WALLET_PRIVATE_KEY. "
        "CDP wallet SIWX signing is not yet supported in this script."
    )


def _build_siwe_message(challenge: dict, wallet_address: str) -> str:
    return (
        f"{challenge['domain']} wants you to sign in with your Ethereum account:\n"
        f"{wallet_address}\n"
        f"\n"
        f"{challenge['statement']}\n"
        f"\n"
        f"URI: {challenge['uri']}\n"
        f"Version: {challenge['version']}\n"
        f"Chain ID: {challenge['chainId']}\n"
        f"Nonce: {challenge['nonce']}\n"
        f"Issued At: {challenge['issuedAt']}\n"
        f"Expiration Time: {challenge['expirationTime']}"
    )


def sign_siwx_challenge(challenge: dict) -> str:
    """Sign an SIWX challenge and return the base64-encoded X-SIWX header value."""
    account = _get_wallet_account()
    message = _build_siwe_message(challenge, account.address)
    signed = account.sign_message(encode_defunct(text=message))
    payload = json.dumps({"message": message, "signature": "0x" + signed.signature.hex()})
    return base64.b64encode(payload.encode()).decode()


def load_cdp_creds_from_defaults() -> None:
"""Populate env vars from known credential files if they aren't already set."""
if os.environ.get("CDP_API_KEY_ID") and os.environ.get("CDP_API_KEY_SECRET") and os.environ.get("CDP_WALLET_ADDRESS"):
return

cred_candidates = [
Path.home() / ".openclaw/credentials/cdp_api_key.json",
Path("/data/.clawdbot/credentials/cdp_api_key.json"),
]
wallet_candidates = [
Path.home() / ".openclaw/credentials/agentkit_wallet.json",
Path("/data/.clawdbot/credentials/agentkit_wallet.json"),
]

for p in cred_candidates:
if p.exists():
data = json.loads(p.read_text())
os.environ.setdefault("CDP_API_KEY_ID", data.get("apiKeyId", ""))
os.environ.setdefault("CDP_API_KEY_SECRET", data.get("apiKeySecret", ""))
if data.get("walletSecret"):
os.environ.setdefault("CDP_WALLET_SECRET", data.get("walletSecret", ""))
break

for p in wallet_candidates:
if p.exists():
data = json.loads(p.read_text())
if data.get("address"):
os.environ.setdefault("CDP_WALLET_ADDRESS", data["address"])
break


def ensure_node_deps() -> None:
if shutil.which("node") is None or shutil.which("npm") is None:
raise RuntimeError("node and npm are required")

pkg = NODE_DEPS_DIR / "package.json"
if not pkg.exists():
NODE_DEPS_DIR.mkdir(parents=True, exist_ok=True)
pkg.write_text(json.dumps({"name": "bios-deep-research-node-deps", "private": True}, indent=2))

modules = NODE_DEPS_DIR / "node_modules"
if not modules.exists():
subprocess.run(
["npm", "install", "@coinbase/cdp-sdk", "@x402/core", "@x402/evm", "viem"],
cwd=NODE_DEPS_DIR,
check=True,
stdout=subprocess.DEVNULL,
stderr=subprocess.DEVNULL,
)

scripts_node_modules = SCRIPT_DIR / "node_modules"
if not scripts_node_modules.exists() and modules.exists():
try:
scripts_node_modules.symlink_to(modules, target_is_directory=True)
except FileExistsError:
pass


def check_wallet_env() -> str:
load_cdp_creds_from_defaults()
if os.environ.get("WALLET_PRIVATE_KEY"):
return "private_key"
if os.environ.get("CDP_API_KEY_ID") and os.environ.get("CDP_API_KEY_SECRET") and os.environ.get("CDP_WALLET_ADDRESS"):
return "cdp"
raise RuntimeError(
"No wallet configured. Set WALLET_PRIVATE_KEY or CDP_API_KEY_ID/CDP_API_KEY_SECRET/CDP_WALLET_ADDRESS."
)


def http_request(method: str, url: str, body: dict | None = None, headers: dict | None = None) -> tuple[int, dict, str]:
data = None
req_headers = {"User-Agent": "bios-deep-research-script/1.0"}
if headers:
req_headers.update(headers)
if body is not None:
data = json.dumps(body).encode()
req_headers.setdefault("Content-Type", "application/json")
req = urllib.request.Request(url, data=data, headers=req_headers, method=method)
try:
with urllib.request.urlopen(req, timeout=60) as resp:[11:41 AM]return resp.getcode(), dict(resp.info()), resp.read().decode()
except urllib.error.HTTPError as e:
return e.code, dict(e.headers), e.read().decode()


def parse_payment_required(status: int, headers: dict, body_text: str) -> dict:
if status != 402:
raise RuntimeError(f"Expected 402, got {status}: {body_text[:500]}")

for k in PAYMENT_REQUIRED_HEADERS:
if k in headers and headers[k]:
try:
return json.loads(base64.b64decode(headers[k]).decode())
except Exception:
pass

try:
return json.loads(body_text)
except Exception as e:
raise RuntimeError(f"Could not parse payment requirements: {e}; body={body_text[:500]}")


def create_payment_signature(payment_required: dict) -> str:
ensure_node_deps()
with tempfile.NamedTemporaryFile("w", suffix=".json", delete=False) as f:
json.dump(payment_required, f)
tmp_path = f.name
try:
proc = subprocess.run(
["node", str(SIGNER_JS), "payment", tmp_path],
cwd=NODE_DEPS_DIR,
check=True,
capture_output=True,
text=True,
env=os.environ.copy(),
)
sig = proc.stdout.strip()
if not sig:
raise RuntimeError(f"No PAYMENT-SIGNATURE returned. stderr={proc.stderr}")
return sig
finally:
try:
os.unlink(tmp_path)
except OSError:
pass


def start_research(query: str, mode: str, conversation_id: str | None = None) -> dict:
wallet_type = check_wallet_env()
body = {"message": query, "researchMode": mode}
if conversation_id:
body["conversationId"] = conversation_id

status, headers, text = http_request("POST", f"{BASE_URL}/api/deep-research/start", body=body)
if status == 200:
return json.loads(text)

payment_required = parse_payment_required(status, headers, text)
payment_sig = create_payment_signature(payment_required)

status2, headers2, text2 = http_request(
"POST",
f"{BASE_URL}/api/deep-research/start",
body=body,
headers={"PAYMENT-SIGNATURE": payment_sig},
)
if status2 != 200:
raise RuntimeError(f"Paid retry failed: {status2}: {text2[:1000]}")

result = json.loads(text2)
result["_wallet_backend"] = wallet_type
return result


def fetch_status(conversation_id: str, payment_sig: str | None = None) -> tuple[int, dict]:
    url = f"{BASE_URL}/api/deep-research/{conversation_id}"
    req_headers: dict[str, str] = {}
    if payment_sig:
        req_headers["PAYMENT-SIGNATURE"] = payment_sig

    status, resp_headers, text = http_request("GET", url, headers=req_headers or None)

    if status == 401:
        body = json.loads(text)
        challenge = body.get("siwx")
        if not challenge:
            raise RuntimeError(f"Got 401 but no SIWX challenge in response: {text[:500]}")
        siwx_header = sign_siwx_challenge(challenge)
        req_headers["X-SIWX"] = siwx_header
        status, resp_headers, text = http_request("GET", url, headers=req_headers)

    if status in (200, 402):
        return status, json.loads(text)
    raise RuntimeError(f"Unexpected poll status {status}: {text[:1000]}")


def poll_until_done(conversation_id: str, interval: int, timeout_s: int) -> dict:
    started = time.time()
    while time.time() - started < timeout_s:
        status, data = fetch_status(conversation_id)
        state = data.get("status")
        if status == 200 and state == "completed":
            return data
        if status == 200 and state in {"queued", "processing", "running", "pending"}:
            elapsed = int(time.time() - started)
            print(f"[{elapsed}s] status={state}", file=sys.stderr)
            time.sleep(interval)
            continue
        if status == 402:
            payment_sig = create_payment_signature(data)
            status2, data2 = fetch_status(conversation_id, payment_sig)
            if status2 == 200:
                return data2
            raise RuntimeError(f"Re-authorized poll failed: {status2}: {json.dumps(data2)[:1000]}")
        raise RuntimeError(f"Unexpected poll response: status={status}, body={json.dumps(data)[:1000]}")
    raise RuntimeError("Polling timeout exceeded")


def dry_run(query: str, mode: str) -> None:
status, headers, text = http_request("POST", f"{BASE_URL}/api/deep-research/start", body={"message": query, "researchMode": mode})
print(f"Status: {status}")
try:
parsed = parse_payment_required(status, headers, text)
print(json.dumps(parsed, indent=2))
except Exception:
print(text)


def main() -> int:
parser = argparse.ArgumentParser(description="BIOS Deep Research via x402 v2")
parser.add_argument("query", help="Research query")
parser.add_argument("--mode", choices=["steering", "smart", "fully-autonomous"], default="steering")
parser.add_argument("--conversation-id")
parser.add_argument("--dry-run", action="store_true")[11:41 AM]parser.add_argument("--no-poll", action="store_true", help="Submit the paid request and stop after receiving conversationId")
parser.add_argument("--poll-interval", type=int, default=10)
parser.add_argument("--timeout", type=int, default=1800)
args = parser.parse_args()

try:
if args.dry_run:
dry_run(args.query, args.mode)
return 0

result = start_research(args.query, args.mode, args.conversation_id)
print(json.dumps(result, indent=2))
conv_id = result.get("conversationId")
if not conv_id:
raise RuntimeError("No conversationId returned")

if args.no_poll:
return 0

completed = poll_until_done(conv_id, args.poll_interval, args.timeout)
print(json.dumps(completed, indent=2))
return 0
except Exception as e:
print(f"Error: {e}", file=sys.stderr)
return 1


if __name__ == "__main__":
raise SystemExit(main())