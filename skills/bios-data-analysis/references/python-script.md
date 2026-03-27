#!/usr/bin/env python3
"""
BIOS Data Analysis client.

Supports two wallet backends with auto-detection:
1. WALLET_PRIVATE_KEY
2. CDP_API_KEY_ID + CDP_API_KEY_SECRET + CDP_WALLET_ADDRESS

This script uses the x402 v2 flow for BIOS data analysis:
- optionally upload input files
- request analysis job
- parse payment requirements from JSON body or PAYMENT-REQUIRED header
- create PAYMENT-SIGNATURE via a small Node helper using @x402/core/@x402/evm
- authenticate poll requests via SIWX (SIWE wallet signatures)
- retry the request
- optionally poll until completion
- download artifacts from completed tasks

Requirements in this environment:
- python3
- node + npm
- eth_account (pip install eth-account)

Usage examples:
python3 analysis.py --dry-run "Analyze gene expression patterns"
python3 analysis.py "Analyze gene expression patterns" --file /path/to/data.csv
python3 analysis.py --no-poll "Summarize public COVID-19 case trends"
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
SIGNER_JS = SCRIPT_DIR / "analysis_signer.mjs"
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
        pkg.write_text(json.dumps({"name": "bios-data-analysis-node-deps", "private": True}, indent=2))

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


def http_request(method: str, url: str, body: dict | bytes | None = None, headers: dict | None = None, content_type: str | None = None) -> tuple[int, dict, str]:
    data = None
    req_headers = {"User-Agent": "bios-data-analysis-script/1.0"}
    if headers:
        req_headers.update(headers)
    if isinstance(body, bytes):
        data = body
    elif body is not None:
        data = json.dumps(body).encode()
        req_headers.setdefault("Content-Type", "application/json")
    if content_type:
        req_headers["Content-Type"] = content_type
    req = urllib.request.Request(url, data=data, headers=req_headers, method=method)
    try:
        with urllib.request.urlopen(req, timeout=60) as resp:
            return resp.getcode(), dict(resp.info()), resp.read().decode()
    except urllib.error.HTTPError as e:
        return e.code, dict(e.headers), e.read().decode()


def upload_file(file_path: str) -> str:
    """Upload a file and return its fileId."""
    import mimetypes
    boundary = "----PythonFormBoundary"
    filename = os.path.basename(file_path)
    mime_type = mimetypes.guess_type(file_path)[0] or "application/octet-stream"

    with open(file_path, "rb") as f:
        file_data = f.read()

    body_parts = []
    body_parts.append(f"--{boundary}\r\n".encode())
    body_parts.append(f'Content-Disposition: form-data; name="file"; filename="{filename}"\r\n'.encode())
    body_parts.append(f"Content-Type: {mime_type}\r\n\r\n".encode())
    body_parts.append(file_data)
    body_parts.append(f"\r\n--{boundary}--\r\n".encode())
    body_bytes = b"".join(body_parts)

    status, _headers, text = http_request(
        "POST",
        f"{BASE_URL}/api/files/upload",
        body=body_bytes,
        content_type=f"multipart/form-data; boundary={boundary}",
    )
    if status != 200:
        raise RuntimeError(f"File upload failed: {status}: {text[:500]}")
    result = json.loads(text)
    return result["fileId"]


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


def start_analysis(task_description: str, file_ids: list[str] | None = None) -> dict:
    wallet_type = check_wallet_env()
    body: dict = {"taskDescription": task_description}
    if file_ids:
        body["fileIds"] = file_ids

    status, headers, text = http_request("POST", f"{BASE_URL}/api/data-analysis/start", body=body)
    if status == 200:
        return json.loads(text)

    payment_required = parse_payment_required(status, headers, text)
    payment_sig = create_payment_signature(payment_required)

    status2, headers2, text2 = http_request(
        "POST",
        f"{BASE_URL}/api/data-analysis/start",
        body=body,
        headers={"PAYMENT-SIGNATURE": payment_sig},
    )
    if status2 != 200:
        raise RuntimeError(f"Paid retry failed: {status2}: {text2[:1000]}")

    result = json.loads(text2)
    result["_wallet_backend"] = wallet_type
    return result


def fetch_status(task_id: str, payment_sig: str | None = None) -> tuple[int, dict]:
    url = f"{BASE_URL}/api/data-analysis/{task_id}"
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
    if status == 429:
        return 429, {}
    raise RuntimeError(f"Unexpected poll status {status}: {text[:1000]}")


def download_artifact(task_id: str, artifact_id: str) -> dict:
    """Download an artifact and return { url, expiresIn }."""
    url = f"{BASE_URL}/api/data-analysis/{task_id}/artifacts/{artifact_id}"

    status, _headers, text = http_request("GET", url)
    if status == 401:
        body = json.loads(text)
        challenge = body.get("siwx")
        if not challenge:
            raise RuntimeError(f"Got 401 but no SIWX challenge: {text[:500]}")
        siwx_header = sign_siwx_challenge(challenge)
        status, _headers, text = http_request("GET", url, headers={"X-SIWX": siwx_header})

    if status != 200:
        raise RuntimeError(f"Artifact download failed: {status}: {text[:500]}")
    return json.loads(text)


def poll_until_done(task_id: str, interval: int, timeout_s: int) -> dict:
    started = time.time()
    while time.time() - started < timeout_s:
        status, data = fetch_status(task_id)

        if status == 429:
            elapsed = int(time.time() - started)
            print(f"[{elapsed}s] rate limited, waiting {interval}s", file=sys.stderr)
            time.sleep(interval)
            continue

        state = data.get("status")
        if status == 200 and state == "completed":
            return data
        if status == 200 and state == "failed":
            print(f"Analysis failed. Data: {json.dumps(data.get('data'), indent=2)}", file=sys.stderr)
            raise RuntimeError("Analysis failed upstream")
        if status == 200 and state in {"queued", "pending"}:
            elapsed = int(time.time() - started)
            print(f"[{elapsed}s] status={state}", file=sys.stderr)
            time.sleep(interval)
            continue
        if status == 402:
            payment_sig = create_payment_signature(data)
            status2, data2 = fetch_status(task_id, payment_sig)
            if status2 == 200:
                return data2
            raise RuntimeError(f"Re-authorized poll failed: {status2}: {json.dumps(data2)[:1000]}")
        raise RuntimeError(f"Unexpected poll response: status={status}, body={json.dumps(data)[:1000]}")
    raise RuntimeError("Polling timeout exceeded")


def dry_run(task_description: str) -> None:
    body: dict = {"taskDescription": task_description}
    status, headers, text = http_request("POST", f"{BASE_URL}/api/data-analysis/start", body=body)
    print(f"Status: {status}")
    try:
        parsed = parse_payment_required(status, headers, text)
        print(json.dumps(parsed, indent=2))
    except Exception:
        print(text)


def main() -> int:
    parser = argparse.ArgumentParser(description="BIOS Data Analysis via x402 v2")
    parser.add_argument("query", help="Task description for the analysis")
    parser.add_argument("--file", action="append", dest="files", help="Local file to upload (can be repeated)")
    parser.add_argument("--dry-run", action="store_true")
    parser.add_argument("--no-poll", action="store_true", help="Submit the paid request and stop after receiving taskId")
    parser.add_argument("--poll-interval", type=int, default=60)
    parser.add_argument("--timeout", type=int, default=9000)
    parser.add_argument("--download-artifacts", action="store_true", help="Download artifact URLs after completion")
    args = parser.parse_args()

    try:
        if args.dry_run:
            dry_run(args.query)
            return 0

        # Step 1: Upload files
        file_ids: list[str] = []
        if args.files:
            for fp in args.files:
                print(f"Uploading {fp} ...", file=sys.stderr)
                fid = upload_file(fp)
                print(f"  fileId: {fid}", file=sys.stderr)
                file_ids.append(fid)

        # Steps 2–3: Start analysis with payment
        result = start_analysis(args.query, file_ids or None)
        print(json.dumps(result, indent=2))
        task_id = result.get("taskId")
        if not task_id:
            raise RuntimeError("No taskId returned")

        if args.no_poll:
            return 0

        # Step 4: Poll until done
        completed = poll_until_done(task_id, args.poll_interval, args.timeout)
        print(json.dumps(completed, indent=2))

        # Step 5: Download artifacts
        artifacts = completed.get("artifacts") or []
        if args.download_artifacts and artifacts:
            for art in artifacts:
                art_id = art.get("id")
                if not art_id:
                    continue
                print(f"Downloading artifact {art_id} ({art.get('name', 'unknown')}) ...", file=sys.stderr)
                dl = download_artifact(task_id, art_id)
                print(f"  url: {dl['url']} (expires in {dl['expiresIn']}s)", file=sys.stderr)

        return 0
    except Exception as e:
        print(f"Error: {e}", file=sys.stderr)
        return 1


if __name__ == "__main__":
    raise SystemExit(main())
