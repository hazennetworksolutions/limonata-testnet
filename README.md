<div align="center">

# 🍋 Limonata Testnet — Full Node & Validator Setup Guide

**A complete guide to running a Limonata testnet full node and registering as a validator**
*Binary installation, genesis sync, systemd service, state sync, validator creation, and the foundation grant flow — step by step.*

[![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04+-E95420?style=flat-square&logo=ubuntu&logoColor=white)](https://ubuntu.com)
[![Limonata](https://img.shields.io/badge/Limonata-Testnet-FFD100?style=flat-square)](https://limonata.xyz)
[![EVM Chain ID](https://img.shields.io/badge/EVM%20Chain%20ID-10777-blue?style=flat-square)](https://limonata.xyz)
[![Cosmos Chain ID](https://img.shields.io/badge/Cosmos%20Chain%20ID-limonata__10777--1-6B7CFF?style=flat-square)](https://limonata.xyz)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow?style=flat-square)](LICENSE)

[hazennetworksolutions.com](https://hazennetworksolutions.com)

</div>

---

> **Author:** HazenNetworkSolutions
> **Network:** Limonata Testnet (Cosmos chain-id: `limonata_10777-1` / EVM chain-id: `10777`)
> **Node binary:** `limonatad` (alias of cosmos/evm `evmd`)
> **Last Updated:** July 2026

---

## 📚 Guides

| Type | Language | Link |
|------|----------|------|
| Full Node & Validator Setup | 🇬🇧 English | [guide-en.md](guide-en.md) |
| Full Node & Validator Setup | 🇹🇷 Türkçe | [guide-tr.md](guide-tr.md) |

---

## 📋 Overview

Limonata is an independent EVM Layer 1 built on the Cosmos SDK + cosmos/evm, with CometBFT single-slot BFT finality (~2s blocks), near-zero network fees, and a full standard EVM toolkit (MetaMask, Hardhat, Foundry, Viem connect unchanged). The native coin is `LIMO` (base denom `aLIMO`, 18 decimals) — a pure network-utility coin used for gas and staking only.

This repository contains step-by-step guides for setting up a Limonata testnet full node and registering as a validator. On Limonata, **validating is access, not capital**: there is no public token sale, so operators self-bond a small faucet amount, build a track record, and can then apply for a locked, non-transferable grant from the foundation to grow their voting power.

- Official Site: [limonata.xyz](https://limonata.xyz)
- Official Validator Guide: [limonata.xyz/VALIDATOR.md](https://limonata.xyz/VALIDATOR.md)
- Chain Source: [github.com/Limonata-Blockchain/limonata](https://github.com/Limonata-Blockchain/limonata)
- Explorer: [explorer.limonata.xyz](https://explorer.limonata.xyz)
- Faucet: [faucet.limonata.xyz](https://faucet.limonata.xyz)
- Proving Grounds (live leaderboard): [grounds.limonata.xyz](https://grounds.limonata.xyz)
- Discord: [discord.gg/vzbJ5u5Kex](https://discord.gg/vzbJ5u5Kex)

---

## About the Author

This guide was prepared by **HazenNetworkSolutions**.
🌐 [hazennetworksolutions.com](https://hazennetworksolutions.com)
