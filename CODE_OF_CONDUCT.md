# Code of conduct for wallet development kit modules

## Index

1. [Code guidelines](#code-guidelines)
    * [C001: Avoid code smells](#c001-avoid-code-smells)
    * [C002: Provide proper documentation for all public and protected components](#c002-provide-proper-documentation-for-all-public-and-protected-components)
    * [C003: Use narrow types for properties and methods](#c003-use-narrow-types-for-properties-and-methods)
    * [C004: Avoid defensive programming](#c004-avoid-defensive-programming)
    * [C005: Always mark errors that a method can throw with @throws](#c005-always-mark-errors-that-a-method-can-throw-with-throws)
2. [Testing guidelines](#testing-guidelines)
    * [T001: Follow the arrange, act, assert pattern](#t001-follow-the-arrange-act-assert-pattern)
    * [T002: Test only components that are part of the public api](#t002-test-only-components-that-are-part-of-the-public-api)
    * [T003: Always test against real values, not properties](#t003-always-test-against-real-values-not-properties)
    * [T004: Define and assign jest functions at the top-level, stub them in test cases](#t004-define-and-assign-jest-functions-at-the-top-level-stub-them-in-test-cases)
    * [T005: Mock all depended-on components that interact with external services](#t005-mock-all-depended-on-components-that-interact-with-external-services)


## Code guidelines

### C001: Avoid code smells

In order to achieve cleaner, more readable and maintainable code, developers should always make sure to avoid any code smell. If you don't know what code smells are, you can check out [refactoring.guru's list](https://refactoring.guru/refactoring/smells) or the [code smell catalog](https://luzkan.github.io/smells/).

#### Example

```javascript
export default WalletManagerEvm extends WalletManager {
    async getFeeRates () {
        [...]

        // Bad: Magic numbers are usually a code smell.
        return { 
            normal: feeRate * 110n / 100n,
            [...]
        }

        // Good:
        return {
            normal: feeRate * WalletManagerEvm._FEE_RATE_NORMAL_MULTIPLIER / 100n,
            [...]
        }
  }
}
```

### C002: Provide proper documentation for all public and protected components

All components that are part of the public api need proper documentation, including meaningful descriptions and types. Documenting private fields and methods is not as essential, but still a significant improvement in readability (especially for complex components).

#### Example

```javascript
export default class WalletAccountEvm extends WalletAccountReadOnlyEvm {
    constructor (seed, path, config = { }) {
        // Bad: the _config property has protected visibility and so it's part of the public api and
        // requires proper documentation.
        /** @protected */
        this._config = config

        // Good:
        /** 
         * The wallet account configuration.
         *
         * @protected
         * @type {EvmWalletConfig}
         */
        this._config = config
    }
}
```

### C003: Use narrow types for properties and methods

When typing properties and methods, developers should always extract and use the narrowest type that best fits them. For fields that can assume any type or whose type is not known, use `unknown` over `any`. For fields that should accept objects with arbitrary data chosen by the user, use `Record<string, unknown>` instead of `Object`.

#### Example

```javascript
// Bad: the type of the 'body' field is too wide, which makes it accept more values than it should. Other
// than providing terrible types to external code, this also moves the effort to check that 'body'
// actually contains a valid value from the type checker to our code.
/**
 * @typedef {Object} TonTransaction
 * @property {string} to - The transaction's recipient.
 * @property {number | bigint} value - The amount of tons to send to the recipient (in nanotons).
 * @property {boolean} [bounceable] - If set, overrides the bounceability of the transaction.
 * @property {Object} [body] - Optional message body for smart contract interactions.
 */

// Good:
/**
 * @typedef {Object} TonTransaction
 * @property {string} to - The transaction's recipient.
 * @property {number | bigint} value - The amount of tons to send to the recipient (in nanotons).
 * @property {boolean} [bounceable] - If set, overrides the bounceability of the transaction.
 * @property {string | Cell} [body] - Optional message body for smart contract interactions.
 */
```

### C004: Avoid defensive programming

Defensive programming requires developers to programmatically check that the pre-conditions of a method hold before proceeding with the call. The only pre-conditions we use in the wallet development kit are the types of the method's arguments, which we can and should always assume to be correct. The .d.ts type definitions along with type checkers already take care of reporting type errors to the client.

#### Example

```javascript
export default class WalletAccountReadOnlyTon {
    /**
     * Returns the balance of the account for a specific token.
     *
     * @param {string} tokenAddress - The smart contract address of the token.
     * @returns {Promise<bigint>} The token balance (in base unit).
     */
    async getTokenBalance (tokenAddress) {
        // Bad: the contract of the 'getTokenBalance' method defines the 'tokenArgument' as a string.
        // Since we do not implement defensive programming, we should assume that its value is always
        // valid and of the correct type. For this reason, there's no real need to check for nullability
        // or that its type is actually 'string'. 
        if (!tokenAddress || typeof tokenAddress !== 'string') {
            throw new Error('Invalid value for the token address.')
        }

        [...]
    }
}
```

### C005: Always mark errors that a method can throw with @throws

Developers should mark all errors that a method might throw in its documentation. This must be done by using the @throws tag to properly document the type of the error and the condition under which it will be thrown. This allows users to know which errors they can expect the method to throw, and also under which circumstances.

#### Example

```javascript
export default class WalletAccountEvm extends WalletAccountReadOnlyEvm {
    /**
     * Approves a specific amount of tokens to a spender.
     *
     * @param {ApproveOptions} options - The approve options.
     * @returns {Promise<TransactionResult>} The transaction's result.
     * @throws {Error} If trying to approve usdts on ethereum with allowance not equal to zero (due to the usdt allowance reset requirement).
     */
    async approve (options) {
        [...]
    }
}
```

## Testing guidelines

### T001: Follow the arrange, act, assert pattern

Unit and integration tests should follow the arrange, act and assert pattern. In the arrange step, tests should set up all constants, mocks and data they will need for the act and assert steps. This also includes any before or after hook. During the act stage, tests should run the code to test e.g., a method call on the [SUT](http://xunitpatterns.com/SUT.html). Finally, the assert stage should include all the necessary assertions on the result of the act step.

#### Example

```javascript
describe('WalletAccountTron', () => {
    describe('signTransaction', () => {
        test('should sign a transaction', async () => {
            // Arrange:
            const EXPECTED_SIGNATURE = 'e2fbd0590d2a6150952afdcdb8c0b137a8828fe45dacc6f17f552b10234baa9231811488104983c4d4333ad51c90343801aa72e41b1d576719cc798c4c98546100'

            sendTrxMock.mockResolvedValue(DUMMY_SEND_TRX_RESULT)

            // Act:
            const transaction = await account.signTransaction(TRANSACTION)

            // Assert:
            expect(sendTrxMock).toHaveBeenCalledWith(TRANSACTION.to, TRANSACTION.value, ACCOUNT.address)

            expect(transaction).toEqual({
                ...DUMMY_SEND_TRX_RESULT,
                signature: [EXPECTED_SIGNATURE]
            })
        })
    })
})
```

### T002: Test only components that are part of the public api

Unit and integration tests should only cover and use components that are part of the public api. This hides implementation details and allows tests to keep working even if the internal implementation changes. It also encourages developers to test against the specification, and from the user's point of view (which will only have access to the public api).

#### Example

```javascript
describe('WalletAccountTron', () => {
    // Bad: the '_signTransaction' method has private visibility, so it's not part of the public
    // api and tests should not cover it. Note that by testing methods that are part the public
    // api, code coverage will indirectly reach protected and private components as well.
    describe('_signTransaction', () => {
        // [...]
    })
})
```

### T003: Always test against real values, not properties

Tests should always assert and compare outputs against real values, instead of properties that should apply to them.

#### Example

```javascript
describe('WalletAccountTron', () => {
    describe('sign', () => {
        test('should return the correct signature', async () => {
            // Bad: the following assertion checks that the output of the 'sign' method is a valid
            // signature, but not that it is correct. If the 'sign' method returns a string that
            // matches the pattern above, the test will pass regardless of whether the signature
            // actually encodes the given message.
            const SIGNATURE_PATTERN = /^(?:0x)?[0-9a-fA-F]{128}(?:00|01|1b|1c|1B|1C)$/

            const signature = await account.sign('Hello world!')

            expect(signature).toMatch(SIGNATURE_PATTERN)

            // Good:
            const EXPECTED_SIGNATURE = '0x0e6d4a25bc8da8fcc08a227612d1caa5f33635f2c9b490f26ea08e228ec879a670607c5bfb68f45fbb1e67972420a366e66aadf18c9dc7806201ea202354c6fc1b'

            const signature = await account.sign('Hello world!')

            expect(signature).toBe(EXPECTED_SIGNATURE)
        })
    })
})
```

### T004: Define and assign jest functions at the top-level, stub them in test cases

When unit tests need to mock one of the system under test's [depended-on components](http://xunitpatterns.com/DOC.html#:~:text=An%20individual%20class%20or%20a,of%20delegation%20via%20method%20calls.), developers should define and assign the necessary jest functions at the top-level of the test suite. Since the wallet development kit uses ESM, this usually requires the use of jest's '[unstable_mockModule](https://jestjs.io/docs/ecmascript-modules#module-mocking-in-esm)'. On the other hand, stubs should only happen inside before hooks or test cases. The assert step should always include assertions to verify that any stub has been called with the proper arguments.

#### Example

```javascript
const sendTrxMock = jest.fn()

jest.unstable_mockModule('tronweb', () => ({
    transactionBuilder: {
        sendTrx: sendTrxMock
    }
}))

describe('WalletAccountTron', () => {
    describe('signTransaction', () => {
        test('should sign a transaction', async () => {
            const EXPECTED_SIGNATURE = 'e2fbd0590d2a6150952afdcdb8c0b137a8828fe45dacc6f17f552b10234baa9231811488104983c4d4333ad51c90343801aa72e41b1d576719cc798c4c98546100'

            sendTrxMock.mockResolvedValue(DUMMY_SEND_TRX_RESULT)

            [...]

            expect(sendTrxMock).toHaveBeenCalledWith(TRANSACTION.to, TRANSACTION.value, ACCOUNT.address)
        })
    })
})
```

### T005: Mock all depended-on components that interact with external services

Unit tests should mock all the system under test's [depended-on components](http://xunitpatterns.com/DOC.html#:~:text=An%20individual%20class%20or%20a,of%20delegation%20via%20method%20calls.) that interact with external services. This allows tests to run faster, in isolation and in a deterministic environment. Integration tests will take care of covering the actual integration with the external service.

#### Example

```javascript
const getNetworkMock = jest.fn()

jest.unstable_mockModule('ethers', () => ({
  ...ethers,
  JsonRpcProvider: jest.fn().mockImplementation(() => ({
    getNetwork: getNetworkMock
  }))
}))
```
