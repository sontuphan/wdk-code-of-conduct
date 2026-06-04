# Proposal for test consistency and coverage in wallet modules

## Table of Contents

- [Abstract](#abstract)
- [Status quo](#status-quo)
  - [Repository audit](#repository-audit)
    - [Unit coverage snapshot](#unit-coverage-snapshot)
    - [Test-case volume](#test-case-volume)
    - [Integration-test conventions](#integration-test-conventions)
    - [Package scripts](#package-scripts)
  - [Concrete examples](#concrete-examples)
    - [Example 1: verify unit tests should not depend on sign](#example-1-verify-unit-tests-should-not-depend-on-sign)
    - [Example 2: provider failover tests should cover behavior, not only construction](#example-2-provider-failover-tests-should-cover-behavior-not-only-construction)
- [Objectives](#objectives)
- [Proposed approach](#proposed-approach)
  - [Use wdk-wallet-evm tests as the reference suite](#use-wdk-wallet-evm-tests-as-the-reference-suite)
  - [Keep unit tests focused on a single unit](#keep-unit-tests-focused-on-a-single-unit)
  - [Standardize integration tests when they exist](#standardize-integration-tests-when-they-exist)
  - [Add blockchain-specific tests only when required by the module](#add-blockchain-specific-tests-only-when-required-by-the-module)
  - [Keep coverage above the OKR threshold](#keep-coverage-above-the-okr-threshold)
  - [Add linting for test files](#add-linting-for-test-files)
- [Execution plan](#execution-plan)
- [Appendix](#appendix)
  - [A. Using StandardJS with Jest](#a-using-standardjs-with-jest)
  - [B. Unit test checklist](#b-unit-test-checklist)
  - [C. Integration test checklist](#c-integration-test-checklist)

## Abstract

Wallet modules expose a shared public api, but their test suites differ in structure, coverage quality and style enforcement. The current state makes it harder to compare modules, identify missing shared behavior and keep test reviews focused on correctness rather than formatting.

This proposal uses `wdk-wallet-evm` as the reference suite for common wallet behavior, maps the other published `wdk-wallet-*` modules against that reference, and keeps blockchain-specific tests limited to behavior that is unique to each module. It also proposes tracking module coverage against the 80% OKR target and extending the existing StandardJS setup to test files so test style stays consistent across contributors.

## Status quo

There are three different problems to solve.

The first problem is test consistency. All `wdk-wallet-*` modules expose the same kind of wallet api, but their test suites are not structured in the same way. Some modules follow the `wdk-wallet-evm` shape with account, read-only account, manager and integration tests. Other modules have only a subset of those tests, or, as in `wdk-wallet-spark`, keep integration tests at the top level of `tests` as `*.integration.test.js` files.

The second problem is coverage quality. A module can have many tests or high line coverage and still miss the behavior that matters. This is especially visible in failover and signature verification tests: some suites test that a failover provider can be constructed, but not that requests actually move from the failing provider to the next provider; some tests use `sign` to prepare data for `verify`, which means the test covers two units instead of one.

The third problem is test style drift. Test files are currently ignored by the StandardJS lint configuration in the wallet modules, so formatting and style decisions are left to each contributor. Over time, this creates divergent test styles across modules and makes reviews spend effort on issues that should be enforced automatically.

### Repository audit

The following table summarizes the current test-suite shape across the `wdk-wallet-*` modules.

> The numbers are intended to highlight structural differences between modules and should be validated again before implementation.

| Module                        | Test files | Test cases | Integration files | Main observation                                                   |
| ----------------------------- | ---------: | ---------: | ----------------: | ------------------------------------------------------------------ |
| `wdk-wallet-btc`              |          5 |         63 |                 1 | Has a richer helper setup and module integration tests.            |
| `wdk-wallet-evm`              |          4 |         73 |                 1 | Reference shape for common wallet behavior.                        |
| `wdk-wallet-evm-7702-gasless` |          3 |         54 |                 0 | Has unit coverage.                                                 |
| `wdk-wallet-evm-erc-4337`     |          1 |         24 |                 1 | Has module integration tests, but package scripts differ from EVM. |
| `wdk-wallet-solana`           |          4 |        119 |                 1 | Broad suite, but some unit tests mix `sign` and `verify`.          |
| `wdk-wallet-solana-gasless`   |          3 |         65 |                 0 | Has unit coverage.                                                 |
| `wdk-wallet-spark`            |          5 |         92 |                 2 | Has integration tests by filename, not under `tests/integration`.  |
| `wdk-wallet-ton`              |          4 |         45 |                 1 | Has module integration tests and TON-specific account coverage.    |
| `wdk-wallet-ton-gasless`      |          3 |         43 |                 0 | Has TON gasless-specific unit coverage.                            |
| `wdk-wallet-tron`             |          3 |         34 |                 0 | Has the common account and manager unit-test shape.                |
| `wdk-wallet-tron-gasfree`     |          3 |         32 |                 0 | Has unit coverage.                                                 |

#### Unit coverage snapshot

The following coverage values were collected from unit-test coverage commands. Integration tests were excluded where the package script includes them by default. Rows marked as failed produced coverage output, but should not be treated as valid coverage results until the failing tests are fixed.

| Module                        | Statements | Branches | Functions |  Lines | Status                                  |
| ----------------------------- | ---------: | -------: | --------: | -----: | --------------------------------------- |
| `wdk-wallet-btc`              |     66.80% |   61.35% |    57.55% | 67.37% | Passed                                  |
| `wdk-wallet-evm`              |     94.77% |   80.76% |   100.00% | 94.92% | Passed                                  |
| `wdk-wallet-evm-7702-gasless` |     74.60% |   57.14% |    88.23% | 75.10% | Passed                                  |
| `wdk-wallet-evm-erc-4337`     |        N/A |      N/A |       N/A |    N/A | No unit test files in the current suite |
| `wdk-wallet-solana`           |     93.82% |   86.77% |    97.77% | 93.65% | Passed                                  |
| `wdk-wallet-solana-gasless`   |     98.20% |   85.89% |   100.00% | 98.79% | Passed                                  |
| `wdk-wallet-spark`            |     77.90% |   54.43% |    85.24% | 78.94% | Passed                                  |
| `wdk-wallet-ton`              |     90.62% |   82.89% |    95.55% | 90.57% | Passed                                  |
| `wdk-wallet-ton-gasless`      |     75.57% |   64.44% |    75.00% | 76.74% | Passed                                  |
| `wdk-wallet-tron`             |     85.38% |   81.15% |    96.42% | 86.22% | Passed                                  |
| `wdk-wallet-tron-gasfree`     |     95.68% |   84.78% |    96.55% | 95.61% | Passed                                  |

#### Test-case volume

Test-case volume differs significantly across modules. That difference may be valid because modules have blockchain-specific behavior, but it also shows that coverage should be mapped against the shared wallet api rather than evaluated by test count alone:

```text
wdk-wallet-solana              119 | ■■■■■■■■■■■■■■■■■■■■■■■■
wdk-wallet-spark                92 | ■■■■■■■■■■■■■■■■■■
wdk-wallet-evm                  73 | ■■■■■■■■■■■■■■■
wdk-wallet-solana-gasless       65 | ■■■■■■■■■■■■■
wdk-wallet-btc                  63 | ■■■■■■■■■■■■■
wdk-wallet-evm-7702-gasless     54 | ■■■■■■■■■■■
wdk-wallet-ton                  45 | ■■■■■■■■■
wdk-wallet-ton-gasless          43 | ■■■■■■■■■
wdk-wallet-tron                 34 | ■■■■■■■
wdk-wallet-tron-gasfree         32 | ■■■■■■
wdk-wallet-evm-erc-4337         24 | ■■■■■
```

#### Integration-test conventions

Integration-test conventions are also inconsistent:

| Convention                         | Modules                                                                                                                            |
| ---------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| `tests/integration/module.test.js` | `wdk-wallet-evm`, `wdk-wallet-ton`, `wdk-wallet-solana`, `wdk-wallet-btc`, `wdk-wallet-evm-erc-4337`                               |
| `*.integration.test.js` files      | `wdk-wallet-spark`                                                                                                                 |
| No integration tests found         | `wdk-wallet-tron`, `wdk-wallet-ton-gasless`, `wdk-wallet-solana-gasless`, `wdk-wallet-evm-7702-gasless`, `wdk-wallet-tron-gasfree` |

```text
Integration-test convention snapshot

Shared module.test.js       5 modules | ■■■■■■■■■■
Other integration shape     1 module  | ■■
No integration tests        5 modules | ■■■■■■■■■■
```

#### Package scripts

The package scripts are not fully aligned either:

| Module group                        | Example modules                                         | Script shape                                                             |
| ----------------------------------- | ------------------------------------------------------- | ------------------------------------------------------------------------ |
| Unit and integration scripts        | `wdk-wallet-evm`, `wdk-wallet-ton`, `wdk-wallet-solana` | `test`, `test:coverage`, `test:integration`, `test:integration:coverage` |
| Explicit unit split                 | `wdk-wallet-btc`                                        | `test:unit`, `test:unit:coverage`, `test:integration`                    |
| Integration without coverage script | `wdk-wallet-evm-erc-4337`                               | `test`, `test:integration`, `test:integration:coverage`                  |
| Integration by filename             | `wdk-wallet-spark`                                      | `test` ignores `integration`, `test:integration` matches `integration`   |
| Testnet-specific script             | `wdk-wallet-tron-gasfree`                               | `test`, `test:coverage`, `test:testnet`                                  |
| Unit-only scripts                   | `wdk-wallet-tron`, remaining gasless modules            | `test`, `test:coverage` only                                             |

### Concrete examples

#### Example 1: verify unit tests should not depend on sign

`wdk-wallet-evm` tests `verify` with a fixed message and a fixed signature. This keeps the test focused on `verify`.

```js
describe("verify", () => {
  const MESSAGE = "Dummy message to sign.";
  const SIGNATURE =
    "0xd130f94c52bf393206267278ac0b6009e14f11712578e5c1f7afe4a12685c5b96a77a0832692d96fc51f4bd403839572c55042ecbcc92d215879c5c8bb5778c51c";

  test("should return true for a valid signature", async () => {
    const result = await account.verify(MESSAGE, SIGNATURE);

    expect(result).toBe(true);
  });
});
```

`wdk-wallet-solana` has a read-only `verify` test that creates the signature by calling `sign` first. This is useful as an integration scenario, but it is not a focused unit test for `verify`.

```js
it("should verify signature for same message across multiple verifications", async () => {
  const account = new WalletAccountSolana(TEST_SEED_PHRASE, "0'/0'/0'", {
    provider: TEST_RPC_URL,
    commitment: "processed",
  });
  const message = "Persistent message";
  const signature = await account.sign(message);

  const readOnlyAccount = new WalletAccountReadOnlySolana(
    await account.getAddress(),
    {},
  );
  const isValid1 = await readOnlyAccount.verify(message, signature);
  const isValid2 = await readOnlyAccount.verify(message, signature);
  const isValid3 = await readOnlyAccount.verify(message, signature);

  expect(isValid1).toBe(true);
  expect(isValid2).toBe(true);
  expect(isValid3).toBe(true);
});
```

The proposed rewrite is to keep the integration scenario under integration tests, and add a unit test with a real precomputed Solana signature:

```js
describe("verify", () => {
  const MESSAGE = "Persistent message";
  const SIGNATURE = "precomputed-signature-for-the-message";

  test("should return true for a valid signature", async () => {
    const result = await readOnlyAccount.verify(MESSAGE, SIGNATURE);

    expect(result).toBe(true);
  });
});
```

#### Example 2: provider failover tests should cover behavior, not only construction

Provider failover is one example of why coverage quality needs to be evaluated by behavior, not only by test count. Several wallet modules support failover-style provider configuration, but tests can still stop at construction or configuration checks without proving that a request moves from a failing provider to the next provider. `wdk-wallet-ton` is a concrete example of this pattern: the latest failover change was validated end-to-end with a custom harness covering the failover network paths, and the existing test suite still passes, but the deterministic failover behavior is not covered by committed unit tests.

`wdk-wallet-ton` currently has a high-level fake TON client:

```js
export default class FakeTonClient extends TonClient {
  async callGetMethod(address, name, stack) {
    const result = await this._blockchain.runGetMethod(address, name, stack);

    return { gasUsed: result.gasUsed, stack: result.stackReader };
  }

  async getContractState(address) {
    const contract = await this._blockchain.getContract(address);

    return {
      code: contract.accountState.state?.code,
      data: contract.accountState.state?.data,
    };
  }
}
```

This is useful for account-level tests, but the failover logic is below this layer. A failover unit test needs a fake object at the `.api` level so the test can force one provider to fail and assert that the next provider receives the call.

`wdk-wallet-ton-gasless` already has a lower-level fake api client shape:

```js
export default class FakeTonApiClient extends TonApiClient {
  constructor(blockchain, paymasterToken) {
    super({ baseUrl: "http://fake-ton-api" });

    this.gasless = {
      gaslessConfig: async () => ({ relayAddress: this.relayAddress }),
      gaslessSend: async () => ({
        success: true,
        message: "Gasless transaction submitted",
      }),
    };
  }
}
```

A deterministic failover test for a module with provider failover should look closer to this:

```js
test("should call the secondary api when the primary api fails", async () => {
  const primaryApi = new FakeHttpApi({ shouldFail: true });
  const secondaryApi = new FakeHttpApi({ getContractState: CONTRACT_STATE });
  const api = new FailoverHttpApi([primaryApi, secondaryApi]);

  const state = await api.getContractState(ADDRESS);

  expect(primaryApi.getContractState).toHaveBeenCalledWith(ADDRESS);
  expect(secondaryApi.getContractState).toHaveBeenCalledWith(ADDRESS);
  expect(state).toEqual(CONTRACT_STATE);
});
```

This example is not about increasing the number of tests. It is about covering the branch that decides whether the failover provider actually switches to the next network client.

## Objectives

The test consistency and coverage work should achieve the following goals:

- Keep all wallet modules aligned to the same unit-test and integration-test structure.
- Use `wdk-wallet-evm` as the reference implementation for common wallet behavior.
- Add blockchain-specific tests only when a module has behavior that does not exist in the common wallet contract.
- Keep unit tests deterministic by mocking depended-on components that interact with external services.
- Keep each wallet module above 80% test coverage.
- Extend the existing StandardJS linting setup to test files to reduce style drift across contributors and modules.
- Create follow-up tasks for low-priority rewrites when existing tests have high coverage but poor test boundaries.

## Proposed approach

### Use wdk-wallet-evm tests as the reference suite

All wallet modules should cover the same common wallet behaviors already covered by `wdk-wallet-evm`, unless the behavior is not supported by the target blockchain. The EVM test suite should be used as the baseline when updating `wdk-wallet-ton`, `wdk-wallet-solana`, and the other wallet modules.

For each module, developers should map the EVM test cases to the equivalent public api in the target module. Missing equivalent test cases should be added. Non-applicable cases should be explicitly skipped from the migration plan with a short reason.

#### Example

```text
EVM reference case:
WalletAccountReadOnlyEvm.getBalance should return the native token balance.

TON equivalent case:
WalletAccountReadOnlyTon.getBalance should return the account ton balance.

Solana equivalent case:
WalletAccountReadOnlySolana.getBalance should return the account lamport balance.
```

### Keep unit tests focused on a single unit

Unit tests should cover one unit at a time. A test for `verify` should verify the behavior of `verify`, and should not depend on `sign` to produce the assertion input. If a test uses `sign` and then `verify`, it is an integration test because it covers two units through one scenario.

When rewriting existing tests, developers should replace these mixed tests with deterministic inputs and expected values. This keeps the failure reason clear and makes the test assert the specification of the method under test.

#### Example

```js
describe("WalletAccountReadOnlySolana", () => {
  describe("verify", () => {
    test("should verify a valid signature", async () => {
      // Arrange:
      const MESSAGE = "Hello world!";
      const SIGNATURE = "valid-signature-for-message";

      // Act:
      const isValid = await account.verify(MESSAGE, SIGNATURE);

      // Assert:
      expect(isValid).toBe(true);
    });
  });
});
```

### Standardize integration tests when they exist

Existing module-level integration tests should follow the same integration-test contract where possible. The integration tests from `wdk-wallet-evm/tests/integration/module.test.js` should be used as the reference for the common module-level behavior.

Shared integration tests should verify that each wallet module can be registered, instantiated and used through the common WDK interfaces. Blockchain-specific integration tests should be added only when the module has behavior that cannot be expressed through the shared wallet contract.

### Add blockchain-specific tests only when required by the module

Blockchain-specific tests should cover behavior that is unique to the target blockchain. They should not duplicate common wallet behavior unless the blockchain requires different fixtures, formats or execution paths.

For example, TON-specific tests can cover cell payload handling, bounceable address behavior, jetton balance flows or provider failover. Solana-specific tests can cover Solana signature formats or transaction instructions. These cases should still follow the same testing guidelines from `CODE_OF_CONDUCT.md`: arrange, act, assert; public api only; real expected values; and mocked depended-on components for external services.

### Keep coverage above the OKR threshold

Each wallet module should keep test coverage above 80%. Coverage should be checked at the module level and should be used as a release gate for changes that add new behavior. If coverage decreases below the threshold, the pull request should either add the missing tests or explicitly create a follow-up task when the uncovered code is accepted as a low-priority risk.

The 80% target is the minimum OKR threshold. For mature modules that already have higher coverage, the goal should be to preserve the current level while improving test quality.

As a potential follow-up, modules with provider failover can add deterministic failover-provider unit tests with fake providers at the correct abstraction layer. For example, `wdk-wallet-ton` would need a `FakeHttpApi` helper at the `.api` layer. This is one concrete class of coverage gap, but it should be treated as an example of the broader coverage-quality work rather than the main goal of the proposal.

### Add linting for test files

Test files should be covered by the existing StandardJS setup used for source code. This should remove the current dependency on review comments for formatting and style consistency, and should make test files follow the same baseline conventions across wallet modules.

The StandardJS setup should account for Jest globals and the ESM module-mocking patterns already used in the test suites. If enabling linting on all existing tests creates too much noise in the first pass, the work can be split into an initial formatting-only cleanup followed by stricter lint rules.

## Execution plan

1. Audit the current test suites and coverage reports across the published `wdk-wallet-*` modules.
2. Create a matrix that maps the EVM reference test cases to each module, marking cases as implemented, missing, not applicable or blockchain-specific.
3. Add missing common unit tests, align existing integration suites with the shared EVM contract and add targeted tests for accepted coverage gaps.
4. Rewrite low-priority tests with poor unit boundaries and add linting for test files to align style across modules.
5. Add coverage checks to the module review process and track every module against the 80% OKR target.

## Appendix

### A. Using StandardJS with Jest

StandardJS does not know about Jest globals (`describe`, `test`, `it`, `expect`, `beforeEach`, `afterEach`, `jest`) by default and reports them as undefined-variable errors in test files. Declare the Jest environment in the module `package.json`:

```json
{
  "standard": {
    "env": ["jest"]
  }
}
```

That is the only change required. No additional plugins or configuration are needed.

---

### B. Unit test checklist

The following test cases are taken from the `wdk-wallet-evm` unit test suite and represent the minimum common behavior every `wdk-wallet-*` module should cover. Cases marked as EVM-specific should not be ported unless the target blockchain supports the same feature. Cases that are not applicable should be explicitly skipped in the migration matrix with a short reason.

> Source reference: `wdk-wallet-evm/tests/`

#### WalletAccountReadOnly

| #   | Test case                                                                                          |
| --- | -------------------------------------------------------------------------------------------------- |
| 1   | `address` should return the correct address                                                        |
| 2   | `getBalance` should return the correct balance of the account                                      |
| 3   | `getBalance` should throw if the account is not connected to a provider                            |
| 4   | `quoteSendTransaction` should successfully quote a transaction                                     |
| 5   | `quoteSendTransaction` should throw if the account is not connected to a provider                  |
| 6   | `quoteTransfer` should successfully quote a transfer operation                                     |
| 7   | `quoteTransfer` should throw if the account is not connected to a provider                         |
| 8   | `getTransactionReceipt` should return the correct transaction receipt                              |
| 9   | `getTransactionReceipt` should return null if the transaction has not been included in a block yet |
| 10  | `getTransactionReceipt` should throw if the account is not connected to a provider                 |
| 11  | `verify` should return true for a valid signature                                                  |
| 12  | `verify` should return false for an invalid signature                                              |
| 13  | `verify` should throw on a malformed signature                                                     |

#### WalletAccount

| #   | Test case                                                                                  |
| --- | ------------------------------------------------------------------------------------------ |
| 14  | `constructor` should successfully initialize an account for the given seed phrase and path |
| 15  | `constructor` should throw if the seed phrase is invalid                                   |
| 16  | `constructor` should throw if the path is invalid                                          |
| 17  | `sign` should return the correct signature                                                 |
| 18  | `signTransaction` should sign a transaction and return a valid hex string                  |
| 19  | `sendTransaction` should successfully send a transaction                                   |
| 20  | `sendTransaction` should throw if the account is not connected to a provider               |
| 21  | `transfer` should successfully transfer tokens                                             |
| 22  | `transfer` should throw if transfer fee exceeds the transfer max fee configuration         |
| 23  | `transfer` should throw if the account is not connected to a provider                      |
| 24  | `toReadOnlyAccount` should return a read-only copy of the account                          |

#### WalletManager

| #   | Test case                                                               |
| --- | ----------------------------------------------------------------------- |
| 25  | `getAccount` should return the account at index 0 by default            |
| 26  | `getAccount` should return the account at the given index               |
| 27  | `getAccount` should throw if the index is a negative number             |
| 28  | `getAccountByPath` should return the account with the given path        |
| 29  | `getAccountByPath` should throw if the path is invalid                  |
| 30  | `getFeeRates` should return the correct fee rates                       |
| 31  | `getFeeRates` should throw if the wallet is not connected to a provider |

Blockchain-specific unit test cases should be added after these common cases in a clearly marked section within the same test file:

```js
describe("WalletAccount<Chain>", () => {
  // Common cases (14–24) ...

  describe("blockchain-specific", () => {
    // Chain-specific cases only
  });
});
```

---

### C. Integration test checklist

The following integration test cases are taken from `wdk-wallet-evm/tests/integration/module.test.js` and represent the minimum common end-to-end scenarios every `wdk-wallet-*` module should cover.

> Source reference: `wdk-wallet-evm/tests/integration/module.test.js`

| #   | Test case                                                                               |
| --- | --------------------------------------------------------------------------------------- |
| 1   | should derive an account, quote the cost of a tx and send the tx                        |
| 2   | should derive two accounts, send a tx from account 1 to 2 and get the correct balances  |
| 3   | should derive an account, sign a message and verify its signature                       |
| 4   | should dispose the wallet and erase the private keys of the accounts                    |
| 5   | should create a wallet with a low transfer max fee, try to transfer and gracefully fail |
| 6   | should sign a transaction, then broadcast manually                                      |

Blockchain-specific integration test cases should be added in a clearly marked section within the same integration test file.
