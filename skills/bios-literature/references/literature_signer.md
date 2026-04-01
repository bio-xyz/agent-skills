#!/usr/bin/env node
/**
 * Node helper for x402 payment signing and SIWX challenge signing.
 * Called as a subprocess by the Python client.
 *
 * Usage:
 *   node literature_signer.mjs payment <payment-required.json>
 *   node literature_signer.mjs siwx <siwx-challenge.json>
 *
 * Env:
 *   WALLET_PRIVATE_KEY — hex private key (0x...)
 *   or CDP_API_KEY_ID + CDP_API_KEY_SECRET + CDP_WALLET_ADDRESS
 */

import fs from 'fs';
import { x402Client } from '@x402/core/client';
import { registerExactEvmScheme } from '@x402/evm/exact/client';
import { toClientEvmSigner } from '@x402/evm';
import { encodePaymentSignatureHeader } from '@x402/core/http';
import { createSiweMessage } from 'viem/siwe';
import { privateKeyToAccount } from 'viem/accounts';
import { createPublicClient, http } from 'viem';
import { base } from 'viem/chains';
import { CdpClient } from '@coinbase/cdp-sdk';

const [mode, inputPath] = process.argv.slice(2);

if (mode === 'payment') {
  if (!inputPath) {
    console.error('usage: literature_signer.mjs payment <payment-required.json>');
    process.exit(1);
  }
  const paymentRequired = JSON.parse(fs.readFileSync(inputPath, 'utf8'));
  const signer = await getEvmSigner();
  const client = new x402Client();
  registerExactEvmScheme(client, { signer });
  const paymentPayload = await client.createPaymentPayload(paymentRequired);
  process.stdout.write(encodePaymentSignatureHeader(paymentPayload));
} else if (mode === 'siwx') {
  if (!inputPath) {
    console.error('usage: literature_signer.mjs siwx <siwx-challenge.json>');
    process.exit(1);
  }
  const challenge = JSON.parse(fs.readFileSync(inputPath, 'utf8'));
  const account = getAccount();
  const message = createSiweMessage({
    address: account.address,
    domain: challenge.domain,
    uri: challenge.uri,
    nonce: challenge.nonce,
    chainId: challenge.chainId,
    statement: challenge.statement,
    version: challenge.version,
    issuedAt: new Date(challenge.issuedAt),
    expirationTime: new Date(challenge.expirationTime),
  });
  const signature = await account.signMessage({ message });
  process.stdout.write(Buffer.from(JSON.stringify({ message, signature })).toString('base64'));
} else {
  if (!mode) {
    console.error('usage: literature_signer.mjs <payment|siwx> <input.json>');
    process.exit(1);
  }
  // Legacy: first arg is a payment-required JSON path
  const paymentRequired = JSON.parse(fs.readFileSync(mode, 'utf8'));
  const signer = await getEvmSigner();
  const client = new x402Client();
  registerExactEvmScheme(client, { signer });
  const paymentPayload = await client.createPaymentPayload(paymentRequired);
  process.stdout.write(encodePaymentSignatureHeader(paymentPayload));
}

function getAccount() {
  const pk = process.env.WALLET_PRIVATE_KEY;
  if (pk) return privateKeyToAccount(pk);
  throw new Error('SIWX signing requires WALLET_PRIVATE_KEY.');
}

async function getEvmSigner() {
  const pk = process.env.WALLET_PRIVATE_KEY;
  if (pk) {
    const account = privateKeyToAccount(pk);
    const publicClient = createPublicClient({ chain: base, transport: http() });
    return toClientEvmSigner(account, publicClient);
  }

  const keyId = process.env.CDP_API_KEY_ID;
  const keySecret = process.env.CDP_API_KEY_SECRET;
  const walletSecret = process.env.CDP_WALLET_SECRET;
  const walletAddress = process.env.CDP_WALLET_ADDRESS;
  if (keyId && keySecret && walletAddress) {
    const cdp = new CdpClient({
      apiKeyId: keyId,
      apiKeySecret: keySecret,
      ...(walletSecret ? { walletSecret } : {}),
    });
    const account = await cdp.evm.getAccount({ address: walletAddress });
    return toClientEvmSigner(account, createPublicClient({ chain: base, transport: http() }));
  }

  throw new Error('No supported wallet env found. Set WALLET_PRIVATE_KEY or CDP_API_KEY_ID/CDP_API_KEY_SECRET/CDP_WALLET_ADDRESS.');
}
