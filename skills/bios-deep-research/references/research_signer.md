#!/usr/bin/env node
import fs from 'fs';
import { x402Client, x402HTTPClient } from '@x402/core/client';
import { ExactEvmScheme, toClientEvmSigner } from '@x402/evm';
import { privateKeyToAccount } from 'viem/accounts';
import { createPublicClient, http } from 'viem';
import { base } from 'viem/chains';
import { CdpClient } from '@coinbase/cdp-sdk';

const [paymentPath] = process.argv.slice(2);
if (!paymentPath) {
console.error('usage: research_signer.mjs <payment-required.json>');
process.exit(1);
}

const paymentRequired = JSON.parse(fs.readFileSync(paymentPath, 'utf8'));

async function getSigner() {
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
return new ExactEvmScheme(account);
}

throw new Error('No supported wallet env found. Set WALLET_PRIVATE_KEY or CDP_API_KEY_ID/CDP_API_KEY_SECRET/CDP_WALLET_ADDRESS.');
}

const scheme = await getSigner();
const client = new x402Client().register('eip155:*', scheme);
const httpClient = new x402HTTPClient(client);
const paymentPayload = await client.createPaymentPayload(paymentRequired);
const headers = httpClient.encodePaymentSignatureHeader(paymentPayload);
process.stdout.write(headers['PAYMENT-SIGNATURE'] || headers['payment-signature'] || '');