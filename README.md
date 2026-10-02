# 🪙 DSC — Decentralized Stablecoin

A minimal, algorithmic, crypto-collateralized stablecoin protocol built with [Foundry](https://book.getfoundry.sh/), loosely inspired by MakerDAO's DAI — but with **no governance, no fees, and no interest rates**. The system is designed so that 1 DSC is always meant to be worth $1, backed exclusively by WETH and WBTC collateral.

> Project built following Patrick Collins' Advanced Foundry course on [Cyfrin Updraft](https://updraft.cyfrin.io/).

## 📋 Overview

The protocol is split into two contracts:

- **`DecentralizedStableCoin.sol`** — a burnable ERC-20 (`DSC`) whose minting and burning are controlled exclusively by its owner, which is set to the `DSCEngine` contract.
- **`DSCEngine.sol`** — the core of the system. It holds all the business logic for depositing/redeeming collateral, minting/burning DSC, and liquidating undercollateralized positions. It is stateless with respect to the token itself — the engine owns `DecentralizedStableCoin` and is the only contract allowed to mint or burn it.

### Stability properties

| Property | Value |
|---|---|
| Collateral | Exogenous (WETH, WBTC) |
| Minting mechanism | Decentralized / Algorithmic |
| Relative stability | Anchored — pegged to USD |
| Collateral type | Crypto |

The protocol must always remain **overcollateralized**: the USD value of all deposited collateral must never fall below the USD value of all minted DSC. This is enforced on every state-changing action via each user's **health factor**.

### Key features

- ✅ Deposit WETH/WBTC as collateral and mint DSC against it, in a single transaction (`depositCollateralAndMintDsc`).
- ✅ Redeem collateral and burn DSC in a single transaction (`redeemCollateralForDsc`).
- ✅ Health-factor-based solvency check (`LIQUIDATION_THRESHOLD = 50`, i.e. 200% overcollateralization) enforced after every action that could put a user at risk.
- ✅ Permissionless liquidation of undercollateralized positions, with a **10% liquidation bonus** for liquidators.
- ✅ Chainlink price feeds for WETH/USD and WBTC/USD, wrapped by a custom `OracleLib` that reverts if the price data is stale (3-hour timeout), intentionally freezing the protocol rather than operating on bad data.
- ✅ Reentrancy protection on every state-changing function via OpenZeppelin's `ReentrancyGuard`.
- ✅ Multi-network support via `HelperConfig` (live Sepolia price feeds vs. local `MockV3Aggregator` / `ERC20Mock` on Anvil).
- ✅ Stateful fuzzing / invariant testing (`forge-std`'s `StdInvariant`) with a dedicated `Handler` contract, asserting that the protocol always holds more collateral value than total DSC supply.
- ✅ Full unit test suite for `DSCEngine`.

## 🏗️ Project architecture

```
├── src/
│   ├── DecentralizedStableCoin.sol     # Burnable, owner-minted ERC-20 stablecoin (DSC)
│   ├── DSCEngine.sol                   # Core protocol logic: collateral, minting, liquidation
│   └── libraries/
│       └── OracleLib.sol               # Chainlink price feed wrapper with stale-price protection
├── script/
│   ├── DeployDSC.s.sol                 # Deployment script
│   └── HelperConfig.s.sol              # Network configuration (Sepolia / local Anvil)
├── test/
│   ├── unit/
│   │   └── DSCEngineTest.t.sol         # Unit tests for DSCEngine
│   ├── fuzz/
│   │   ├── Invariants.t.sol            # Stateful invariant tests (targets the Handler)
│   │   ├── OpenInvariantsTest.t.sol    # Open (unguided) invariant tests (targets DSCEngine directly)
│   │   └── Handler.t.sol               # Handler contract that bounds and guides fuzzed calls
│   └── mocks/
│       ├── MockV3Aggregator.sol        # Chainlink price feed mock
│       └── ERC20Mock.sol               # Mock WETH/WBTC tokens
└── README.md
```

### Contracts and scripts

| File | Role |
|---|---|
| `DecentralizedStableCoin.sol` | ERC-20 (`ERC20Burnable` + `Ownable`) representing DSC; `mint`/`burn` are `onlyOwner`, meant to be called exclusively by `DSCEngine` |
| `DSCEngine.sol` | Handles collateral deposits/redemptions, DSC minting/burning, health-factor checks, and liquidations |
| `OracleLib.sol` | Library used by `DSCEngine` to read Chainlink price feeds safely, reverting (`OracleLib__StalePrice`) if a round is stale or incomplete |
| `DeployDSC.s.sol` | Deploys `DecentralizedStableCoin` and `DSCEngine` (configured with WETH/WBTC + their price feeds from `HelperConfig`), then transfers DSC ownership to the engine |
| `HelperConfig.s.sol` | Provides network-specific collateral token and price feed addresses — live Sepolia addresses, or freshly deployed mocks on Anvil |
| `Handler.t.sol` | Bounds and routes invariant-fuzzer calls to valid actions on `DSCEngine`, so invariant runs stay meaningful instead of reverting on nonsensical inputs |
| `Invariants.t.sol` | Stateful invariant suite asserting the core protocol invariant (collateral value ≥ total DSC supply) via the `Handler` |
| `OpenInvariantsTest.t.sol` | Invariant suite that fuzzes `DSCEngine` directly, without the `Handler`'s guidance |
| `DSCEngineTest.t.sol` | Unit tests covering deposits, minting, redemptions, burning, liquidation, price conversion, and health factor calculations |

## ⚙️ How it works

1. **Deposit collateral** — `depositCollateral()`: the user deposits an allowed ERC-20 (WETH or WBTC) into the engine.
2. **Mint DSC** — `mintDsc()`: the user mints DSC against their deposited collateral. The engine recalculates the user's health factor and reverts (`DSCEngine__BreaksHealthFactor`) if it would fall below the minimum.
3. **Combined action** — `depositCollateralAndMintDsc()` does both in one call.
4. **Redeem / burn** — `redeemCollateral()` and `burnDsc()` let a user withdraw collateral or pay down their DSC debt; `redeemCollateralForDsc()` combines both, always re-checking the health factor afterward.
5. **Health factor** — calculated as `(collateralValueInUsd * LIQUIDATION_THRESHOLD / LIQUIDATION_PRECISION) * PRECISION / totalDscMinted`. A value below `MIN_HEALTH_FACTOR` (1) means the position is eligible for liquidation.
6. **Liquidation** — `liquidate()`: anyone can cover part or all of an undercollateralized user's DSC debt in exchange for their collateral plus a 10% bonus, as long as doing so improves (and doesn't just transfer) the user's health factor.
7. **Price feeds** — `getUsdValue()` and `getTokenAmountFromUsd()` convert between token amounts and their USD value using Chainlink feeds via `OracleLib.staleCheckLatestRoundData()`, which reverts on stale data instead of returning an unreliable price.

## 🔧 Prerequisites

- [Foundry](https://book.getfoundry.sh/getting-started/installation) (`forge`, `cast`, `anvil`)
- [Git](https://git-scm.com/)

## 📦 Installation

```bash
git clone <repository-url>
cd <repository-name>
forge install
forge build
```

Expected dependencies (via `forge install`):

- `smartcontractkit/chainlink-brownie-contracts` (or `@chainlink/contracts`)
- `OpenZeppelin/openzeppelin-contracts` (`@openzeppelin`)
- `foundry-rs/forge-std`

## 🧪 Tests

Unit tests:

```bash
forge test --match-path test/unit/*
```

Invariant / fuzz tests:

```bash
forge test --match-path test/fuzz/*
```

Run the full suite with more verbosity or filter by test name:

```bash
forge test -vvv
forge test --match-test invariant_protocolMustHaveMoreValueThanTotalSupply -vvvv
```

Check coverage:

```bash
forge coverage
```

> Invariant runs can be tuned in `foundry.toml` (`[invariant]` section — `runs`, `depth`, `fail_on_revert`).

## 🚀 Deployment

### Local network (Anvil)

```bash
anvil
```

In another terminal, set the deployer key env var expected locally and run:

```bash
export PK_LOCALHOST_ACC_0=<anvil_private_key>
forge script script/DeployDSC.s.sol:DeployDSC --rpc-url http://127.0.0.1:8545 --broadcast
```

On any network other than Sepolia, `HelperConfig` automatically deploys `MockV3Aggregator` price feeds and `ERC20Mock` WETH/WBTC tokens.

### Testnet (Sepolia)

Set up your environment variables (e.g. in a `.env` file):

```bash
SEPOLIA_RPC_URL=<your_rpc_url>
SEPOLIA_PRIVATE_KEY=<your_private_key>
ETHERSCAN_API_KEY=<your_api_key>
```

Deploy:

```bash
forge script script/DeployDSC.s.sol:DeployDSC --rpc-url $SEPOLIA_RPC_URL --broadcast --verify -vvvv
```

`HelperConfig` will automatically use the live Sepolia WETH/USD and WBTC/USD Chainlink price feeds, along with their respective token addresses.

## 🔍 Available view functions

| Function | Returns |
|---|---|
| `getAccountInformation(address user)` | Total DSC minted and total collateral value (USD) for a user |
| `getAccountCollateralValue(address user)` | Total USD value of all collateral deposited by a user |
| `getUsdValue(address token, uint256 amount)` | USD value of a given amount of a collateral token |
| `getTokenAmountFromUsd(address token, uint256 usdAmountInWei)` | Token amount equivalent to a given USD value |
| `getHealthFactor(uint256 totalDscMinted, uint256 collateralValueInUsd)` | Computes a health factor from given figures |
| `getCollateralTokens()` | List of allowed collateral token addresses |
| `getCollateralBalanceOfUser(address user, address token)` | Amount of a specific collateral token deposited by a user |
| `getCollateralTokenPriceFeed(address token)` | Chainlink price feed address for a collateral token |

## ⚠️ Custom errors

- `DSCEngine__NeedsMoreThanZero` — an amount parameter was zero.
- `DSCEngine__TokenAddressesAndPriceFeedAddressesAmountsDontMatch` — constructor arrays length mismatch.
- `DSCEngine__TokenNotAllowed(token)` — the token is not an allowed collateral type.
- `DSCEngine__TransferFailed` — an ERC-20 transfer failed.
- `DSCEngine__BreaksHealthFactor(userHealthFactor)` — the action would leave the user undercollateralized.
- `DSCEngine__MintFailed` / `DSCEngine__BurnFailed` — the DSC mint/burn call failed.
- `DSCEngine__HealthFactorOk` — tried to liquidate a user who is not undercollateralized.
- `DSCEngine__HealthFactorNotImproved` — a liquidation did not actually improve the user's health factor.
- `DecentralizedStableCoin__AmountMustBeMoreThanZero` / `DecentralizedStableCoin__BurnAmountExceedsBalance` / `DecentralizedStableCoin__NotZeroAddress` — guards on direct DSC minting/burning.
- `OracleLib__StalePrice` — a Chainlink price feed returned stale or incomplete round data.

## ⚠️ Known limitations

- `getHealthFactor()` (the parameterless external version) is currently an empty stub and does not return a value.
- As noted in the contract's own comments, if the system ever drops to ≤100% collateralization very quickly (e.g. a crash before anyone can liquidate), there is no incentive left for liquidators — a known, accepted risk of this design.

## 📜 License

MIT

## 🙏 Credits

Author: **David Vidal**. Based on the **Advanced Foundry** course by [Patrick Collins](https://github.com/PatrickAlphaC) on [Cyfrin Updraft](https://updraft.cyfrin.io/).
