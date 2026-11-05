# Agent Notes (`ethereumjs/protocols`)

Short entrypoint for coding agents working in **`ethereumjs/protocols`**.

## Purpose of This Repository

This repository hosts open protocol specifications designed within the **EthereumJS** ecosystem (such as [`devp2p/ETHY-0-SPEC.md`](./devp2p/ETHY-0-SPEC.md)), paired with reference implementations in the sibling [`ethereumjs-monorepo`](../ethereumjs-monorepo) workspace (`packages/devp2p`, `packages/ethy`, `packages/beacon-light`, `packages/statemanager`, `packages/mpt`).

## Rules & Habits

- **Spec Quality (`spec-writing.mdc`):** Specifications here are written for both human protocol engineers and AI coding agents. Every message code, binary layout, cryptographic invariant, and state-machine rule must be explicit, unambiguous, and accompanied by concrete wire schemas and worked examples.
- **Cross-Repo Alignment:** Keep message codes, RLP tuple field orders, binary record sizes (e.g., 104-byte `BinaryAccount` / 136-byte slab rows), and discovery tags synchronized with [`ethereumjs-monorepo`](../ethereumjs-monorepo).
- **Git (`git.mdc`):** All changes go through Pull Requests against `main`. Never commit, push, open a PR, or merge without an explicit ask from the human in the current turn.
