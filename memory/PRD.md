# FozPay — Crypto Payment Gateway (PRD)

## Origin
Full copy of GitHub repo `genrymaksd-del/WER` (FozPay) deployed into `/app`.
Stack: FastAPI + React + MongoDB. HD (BIP-44) hot wallet, EVM + TRON + BTC/LTC/SOL.
Live integrations: Alchemy (EVM RPC/scan), TronGrid (TRON), 1inch (real on-chain swaps), Binance public prices.

## Personas
- Admin/superadmin: manages users, platform fees, hot wallet, withdrawals.
- Merchant/user: receives crypto (invoices/deposits), exchanges, withdraws.

## Core Requirements (static)
- Crypto payment gateway: merchant invoices, cabinet balances, deposits, withdrawals, swap.
- Single BIP-39 mnemonic → per-user deposit addresses; funds swept to a platform hot wallet.
- Public registration disabled; admin creates/manages users.
- Hot-wallet mnemonic encrypted at rest (Fernet); key `MNEMONIC_ENC_KEY` only in backend/.env.

## Implemented (2026-06 — this deployment session)
- **Deploy**: repo copied to `/app`, deps installed, `.env` wired with user keys
  (ALCHEMY_KEY, TRONGRID_KEY, ONEINCH_KEY) + generated JWT_SECRET/MNEMONIC_ENC_KEY,
  ADMIN_EMAIL/ADMIN_PASSWORD, FRONTEND_URL. New HD wallet generated. Admin: admin@fozpay.io / FozPay#Admin2026.
- **Deposit crediting fix**: invoice/pay-in `amount_to_pay = amount` (no fee added on top).
  A request for 3 USDT now expects exactly 3 USDT (was 3.5 → "Partially"). Platform fee is
  deducted AFTER arrival in `_confirm_payment`/`_credit_direct_deposit`.
- **Auto-convert on withdrawal (from USDT)**: `withdraw()` — if requested currency balance is
  insufficient but USDT covers the USD-equivalent (+2% buffer), debit USDT and mark the tx
  `convert_from=USDT`; `withdrawal_worker` runs a REAL 1inch swap USDT→iso on the hot wallet,
  then sends the requested currency. EVM-only; non-EVM falls back to "insufficient funds".
- **Per-currency+network withdrawal fee**: `withdrawal_fee_by_key` ("ISO:network_id" →
  {cabinet, api}) with global fallback. `withdrawal_matrix()` drives the admin UI (16 cells).
  Admin Settings → Платформа renders an editable table; `withdrawal_fee_for()` applied in withdraw().
- **Admin user balances in USDT**: `GET /api/admin/users` returns `balance_usdt` + per-iso `balances`.
  Shown per user in Settings → Користувачі.
- **Explorer hash on all tx**: `catalog.explorer_tx_url()` + `recovery.last_incoming_tx()` fetch
  the real incoming deposit hash (Alchemy/TronGrid/mempool); withdrawals store the payout hash.
  `explorer_url` stored on every transaction and rendered as a link in the Wallet tx table.

## Testing
- 14/14 backend tests pass (`backend/tests/test_fozpay_new_features.py`). Frontend flows verified.

## MOCKED / untestable without funds
- REAL on-chain 1inch swap execution, EVM withdrawal payout, and live deposit detection require
  the hot wallet to hold crypto + native gas. Only API logic/branching/validation/DB records were
  verified end-to-end. TRON/BTC payouts remain operator-handled (auto payout is EVM only).

## Backlog / Next
- P1: Real automatic TRON (USDT-TRC20) sweeps/withdrawals.
- P1: When a deposit txid backfills after confirmation, also rewrite explorer_url.
- P2: Admin gas-balance alert when hot-wallet BNB/ETH drops below threshold.
- P2: Admin log of auto-conversions (USDT→target) with hashes.
