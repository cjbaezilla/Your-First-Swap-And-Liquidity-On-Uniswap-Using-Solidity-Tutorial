# Uniswap V2 Solidity Contracts: Complete Technical Guide

## Table of Contents
1. [Introduction to Uniswap V2](#introduction-to-uniswap-v2)
2. [Core Contracts Architecture](#core-contracts-architecture)
3. [Key Interfaces and Contracts](#key-interfaces-and-contracts)
4. [Fee Mechanism and Tokenomics](#fee-mechanism-and-tokenomics)
5. [Price Oracle and TWAP](#price-oracle-and-twap)
6. [Security Considerations](#security-considerations)
7. [Sepolia Testnet Configuration](#sepolia-testnet-configuration)
8. [Basic Operations with Solidity](#basic-operations-with-solidity)
9. [Best Practices and Warnings](#best-practices-and-warnings)
10. [References and Resources](#references-and-resources)

---

## Introduction to Uniswap V2

Uniswap V2 is a decentralized exchange protocol that allows users to trade ERC20 tokens without relying on traditional order books. Instead, it uses an automated market making system where liquidity providers deposit tokens into smart contracts and traders swap against those reserves. The protocol operates through a series of interconnected smart contracts that handle everything from token pairing to fee distribution.

Think of Uniswap V2 as a digital marketplace where anyone can become a market maker. Instead of matching individual buyers and sellers, the system uses mathematical formulas to determine prices based on the ratio of tokens in each trading pool. This approach ensures continuous liquidity and democratizes trading on Ethereum.

The protocol consists of three main layers: the core contracts that handle the fundamental trading logic, the periphery contracts that provide user-friendly interfaces, and a set of interfaces that define how these components interact.

---

## Core Contracts Architecture

The Uniswap V2 system is built around three primary contracts that work together to enable decentralized trading. Understanding their relationships is crucial for anyone working with the protocol.

### UniswapV2Factory

The Factory contract serves as the central registry for all Uniswap V2 trading pairs. It's the only contract responsible for creating new Uniswap V2 Pair contracts. When someone wants to create a new trading pair between two tokens, they interact with the Factory, which deploys a new Pair contract.

The Factory maintains a complete mapping of all created pairs and tracks which tokens are supported. It also stores the fee receive address and has the ability to set a fee to protocol parameter. The Factory doesn't hold any user funds; it's purely a coordination mechanism.

Key responsibilities include:
- Creating new trading pairs via the `createPair` function
- Tracking all existing pairs through the `getPair` and `allPairs` mappings
- Setting and updating the protocol fee recipient
- Managing the fee to protocol parameter

### UniswapV2Pair

The Pair contract is the heart of each trading pool. Each pair represents a specific token combination (like ETH/USDC or DAI/USDT). When the Factory creates a new pair, it deploys a fresh Pair contract that will hold reserves of both tokens and execute trades.

Each Pair contract:
- Holds the reserves of both tokens in the pool
- Implements the constant product formula (x * y = k)
- Mints and burns liquidity provider (LP) tokens
- Collects the 0.3% trading fees
- Updates price accumulators for the time-weighted average price (TWAP) oracle

The Pair contract inherits from both `UniswapV2ERC20` (for LP token functionality) and `UniswapV2PermissionlessMintBurn` (for minting/burning LP tokens). It also implements the `IUniswapV2Pair` interface.

### UniswapV2Router01 and UniswapV2Router02

The Router contracts provide user-friendly interfaces for interacting with the core Pair contracts. They handle complex operations like calculating optimal swap paths, managing token approvals, and executing multi-step trades.

Router02 is an enhanced version of Router01 with additional features:
- Support for fee-on-transfer tokens
- Better handling of ETH interactions
- Support for advanced routing scenarios

Routers are not permissioned; anyone can deploy their own router. The official routers provide a standardized way to interact with Uniswap V2 pairs.

---

## Key Interfaces and Contracts

### Interface Structure

Uniswap V2 uses a comprehensive set of interfaces that define how different components interact. These interfaces ensure compatibility and allow for flexible implementations.

**IUniswapV2Factory** defines the functions for pair creation and registry management:
- `createPair(address tokenA, address tokenB)` creates a new trading pair
- `getPair(address tokenA, address tokenB)` returns the address of a pair
- `allPairs(uint256)` returns all created pairs
- `feeTo()` and `feeToSetter()` manage protocol fees

**IUniswapV2Pair** defines the core trading functions:
- `swap(uint amount0Out, uint amount1Out, address to, bytes calldata data)` executes a swap
- `mint(address to)` mints LP tokens for liquidity providers
- `burn(address to)` burns LP tokens and returns reserves
- `skim(address to)` removes excess tokens not accounted for in reserves
- `sync()` updates reserves after external transfers

**IUniswapV2ERC20** extends the standard ERC20 interface for LP tokens:
- `DOMAIN_SEPARATOR()` for permit functionality
- `permit(address owner, address spender, uint value, uint deadline, uint8 v, bytes32 r, bytes32 s)` enables gasless approvals

**IUniswapV2Router01** and **IUniswapV2Router02** provide the main user-facing functions:
- `swapExactTokensForTokens(uint amountIn, uint amountOutMin, address[] calldata path, address to, uint deadline)`
- `addLiquidity(address tokenA, address tokenB, uint amountADesired, uint amountBDesired, uint amountAMin, uint amountBMin, address to, uint deadline)`
- `removeLiquidity(address tokenA, address tokenB, uint liquidity, uint amountAMin, uint amountBMin, address to, uint deadline)`

### Contract Inheritance Hierarchy

The Uniswap V2 contracts follow a clear inheritance structure that promotes code reuse and modularity. Understanding this hierarchy helps developers extend or customize the protocol.

At the base level, we have `UniswapV2Pair` which inherits from multiple contracts:

```
UniswapV2Pair
├── UniswapV2ERC20 (Provides LP token functionality)
├── UniswapV2PermissionlessMintBurn (Allows minting/burning without permissions)
└── ReentrancyGuard (Security against reentrancy attacks)
```

The `UniswapV2ERC20` contract itself inherits from:
```
UniswapV2ERC20
├── ERC20 (Standard token implementation)
└── ERC20Permit (Enables gasless token approvals via signatures)
```

This modular design allows each component to be independently tested and maintained. The ReentrancyGuard is crucial for security, preventing malicious contracts from re-entering during token transfers.

---

## Important Functions and Their Purposes

### Factory Functions

`createPair(address tokenA, address tokenB)` is the only function that creates new trading pairs. It calculates a unique pair address using the CREATE2 opcode, ensuring the same pair always has the same address regardless of who creates it.

`setFeeTo(address)` allows the fee collector to receive a portion of swap fees. This function can only be called by the fee setter.

`setFeeToSetter(address)` transfers the ability to set the fee recipient to another address.

### Pair Functions

`swap(uint amount0Out, uint amount1Out, address to, bytes calldata data)` is the core trading function. It transfers tokens out of the pool, calculates the new reserves, and sends the output tokens to the recipient. The `data` parameter allows for custom callbacks.

`mint(address to)` is called by liquidity providers to receive LP tokens. It calculates how many tokens were added to the reserves relative to the existing supply and mints accordingly.

`burn(address to)` allows liquidity providers to withdraw their share of the pool. It burns LP tokens and transfers the corresponding reserves to the recipient.

`skim(address to)` handles cases where someone accidentally transfers tokens directly to the Pair contract without updating reserves. This function removes the excess tokens.

`sync()` updates the price accumulator with the current reserve ratio. This is important for the TWAP oracle.

### Router Functions

The router functions handle complex multi-step operations. For example, `swapExactTokensForTokens` takes an exact input amount and calculates the minimum acceptable output, then finds the best path through multiple pairs to achieve the best price.

`addLiquidity` and `addLiquidityETH` calculate the optimal amounts of each token to deposit based on current reserves, ensuring the provider receives LP tokens proportional to their contribution.

`removeLiquidity` and `removeLiquidityETH` perform the reverse operation, burning LP tokens and returning the underlying tokens in the correct proportions.

---

## Fee Mechanism (0.3% Swap Fee)

Uniswap V2 charges a 0.3% fee on every swap executed through the Pair contracts. This fee is collected in the tokens being traded and stays within the pool, increasing the reserves and therefore the value of LP tokens.

Here's how the fee calculation works in practice:

When a trader swaps tokens, they receive slightly less than the theoretical amount based on the constant product formula. For example, if you swap 100 tokens in a perfect world with no fees, you might expect to receive 99 tokens out. With the 0.3% fee, you'll receive only about 98.7 tokens. The 0.3 difference stays in the pool.

The fee structure is implemented at the Pair level. When a swap occurs, the contract first checks if the required output amount is available, then it increments the output amount by 0.3% before applying the constant product formula. This effectively reduces the input amount seen by the formula, leaving the remainder in the pool.

There's also an optional protocol fee that can be set by the governance (in V2, controlled by the fee setter). This fee is a fraction of the 0.3% swap fee that goes to a designated address rather than remaining in the pool. By default, the protocol fee is set to 0.

---

## ERC20 Implementation for LP Tokens

Each Uniswap V2 Pair contract also functions as an ERC20 token representing the liquidity provider's share in that pool. These LP tokens are minted when liquidity is added and burned when liquidity is removed.

The LP token implementation extends the standard ERC20 with additional functionality. Most importantly, it includes the `permit` function which allows token holders to approve spender addresses without executing a separate transaction. This is achieved through EIP-712 typed structured data hashing.

The LP token contract tracks total supply, balances, and standard ERC20 events. When liquidity is added via `mint`, new LP tokens are created and assigned to the provider. The number of tokens minted depends on how much liquidity they're adding relative to the existing reserves.

When burning LP tokens, the contract calculates how much of each underlying token the provider should receive based on their share of the total supply. This calculation happens in the `burn` function before the tokens are actually transferred out.

---

## Price Oracle Mechanism (TWAP)

Uniswap V2 includes a built-in time-weighted average price oracle that provides reliable, manipulation-resistant price data. This oracle is crucial for many DeFi applications that need accurate token prices without relying on external sources.

The TWAP (Time-Weighted Average Price) oracle works by accumulating price information over time. Each Pair contract maintains a cumulative price variable that gets updated every time a swap occurs. The update happens in the `update` function, which is called by `_update` after every trade.

The formula is straightforward: the price (as a ratio of reserves) is multiplied by the time elapsed since the last update, and this product is added to the cumulative price. This creates a running total that represents the integral of price over time.

To get the TWAP over a specific period, you query the cumulative price at two different timestamps and divide the difference by the time elapsed. This gives you the average price during that interval.

The oracle has some security properties: it's resistant to short-term manipulation because an attacker would need to maintain a manipulated price for a significant duration to meaningfully affect the average. However, it's not designed for long-term price feeds.

---

## Security Considerations

Security is paramount when interacting with Uniswap V2 contracts. The protocol has been extensively audited and has been operating safely for years, but developers must still be aware of common pitfalls.

**Reentrancy Protection**: The Pair contract includes a reentrancy guard to prevent recursive calls during token transfers. However, developers should still follow the checks-effects-interactions pattern when building on top of Uniswap.

**Front-Running Protection**: Always use slippage tolerance parameters. The `amountOutMin` parameters in swap functions protect against unfavorable price movements between when you sign a transaction and when it's mined.

**Token Security**: Not all ERC20 tokens behave correctly. Be aware of tokens that:
- Charge fees on transfers (use fee-on-transfer safe functions)
- Have incorrect decimal implementations
- Freeze or blacklist addresses
- Revert on empty approvals

**Deadline Parameter**: Always set a reasonable deadline for router operations. Prevents transactions from being executed much later when prices may have changed drastically.

**Approval Management**: Clear approvals after use where possible to reduce attack surface. Consider using the `permit` pattern to avoid separate approval transactions.

**Pair Address Validation**: Always verify that a pair address is legitimate and has sufficient liquidity before interacting with it.

---

## Sepolia Testnet Configuration

### Uniswap V2 Contract Addresses on Sepolia

The following addresses are for the Ethereum Sepolia testnet. These are the official deployment addresses.

| Contract | Address | Notes |
|----------|---------|-------|
| UniswapV2Factory | `0x5C69bEe701ef814a2B6a3EDD4B1652CB9cc5aA6f` | Creates all trading pairs |
| UniswapV2Router02 | `0x7a250d5630B4cF539739dF2C5dAcb4c659F2488D` | Main router interface |
| UniswapV2Router01 | `0xf164fC0Ec4E93095b804a4795bbe1e041497b92` | Legacy router |
| WETH | `0xfFf9976782d46CC05630D1f6eBAb18b2324d6B14` | Wrapped ETH token |
| USDC | `0x1c7D4B196Cb0C7B01d743Fbc6116a902379C7238` | Circle's USD Coin |
| USDT | `0x3b00ef435fa4fef362a5bc0c4724c99b062ac9c7` | Tether |
| DAI | `0x6b175474e89094c44da98b954eedeac495271d0f` | MakerDAO's DAI |

*Note: Token addresses on testnets may differ from mainnet and can change. Always verify current addresses from official sources.*

### Sepolia ETH Faucets

Getting testnet ETH is essential for deploying contracts and paying gas fees. Here are reliable faucets:

**Official Alchemy Faucet**: Visit alchemy.com and sign up for a free account to receive Sepolia ETH daily.

**Paradigm Faucet**: Available at faucet.paradigm.xyz, requires a Twitter account verification.

**Chainlink Faucet**: Provides both ETH and LINK tokens at faucet.chain.link.

**Infura Faucet**: Available through the Infura dashboard after registration.

**QuickNode Faucet**: quicknode.com/faucet offers daily ETH allocations.

Remember that testnet faucets have rate limits and may require social media verification. Always check the latest faucet status as they can change or become temporarily unavailable.

### Recommended RPC Endpoints for Sepolia

Using reliable RPC endpoints is crucial for stable development:

**Public Endpoints**:
- `https://rpc.sepolia.org` (Ethereum Foundation)
- `https://sepolia.drpc.org`
- `https://rpc.sepolia.dev`

**Provider Endpoints** (require registration):
- Alchemy: `https://eth-sepolia.g.alchemy.com/v2/your-api-key`
- Infura: `https://sepolia.infura.io/v3/your-project-id`
- QuickNode: `https://sepolia.g.alchemy.com/v2/your-endpoint`

For production applications, always use paid provider endpoints with proper rate limiting.

### Block Explorer

The official Sepolia block explorer is `sepolia.etherscan.io`. Use it to:
- Verify deployed contracts
- View transaction history
- Check token balances
- Debug failed transactions
- Monitor gas prices

---

## Basic Operations with Solidity

### Swap Operations

Swapping tokens is the most common Uniswap V2 operation. The router provides several variants to handle different scenarios.

#### swapExactTokensForTokens

This function swaps a precise amount of input tokens for as many output tokens as possible, with a minimum limit specified by the developer.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@uniswap/v2-periphery/contracts/interfaces/IUniswapV2Router02.sol";

contract TokenSwapper {
    IUniswapV2Router02 public immutable router;
    
    constructor(address _router) {
        router = IUniswapV2Router02(_router);
    }
    
    function swapExactTokensForTokens(
        uint amountIn,
        uint amountOutMin,
        address[] calldata path,
        address to,
        uint deadline
    ) external {
        // First, approve the router to spend our tokens
        IERC20(path[0]).approve(address(router), amountIn);
        
        // Execute the swap
        router.swapExactTokensForTokens(
            amountIn,
            amountOutMin,
            path,
            to,
            deadline
        );
    }
}
```

**Parameters**:
- `amountIn`: Exact amount of input tokens to swap
- `amountOutMin`: Minimum acceptable output (slippage protection)
- `path`: Array of token addresses representing the swap path (e.g., [USDC, WETH, DAI])
- `to`: Recipient of output tokens
- `deadline`: Transaction must be mined before this timestamp

**Slippage Considerations**: Setting `amountOutMin` too low risks unfavorable exchange rates. A typical slippage tolerance is 0.5-1%. You can calculate expected minimum amounts using `getAmountsOut` function:

```solidity
function calculateMinOut(uint amountIn, address[] memory path, uint slippageBPS) external view returns (uint) {
    uint[] memory amounts = router.getAmountsOut(amountIn, path);
    uint expectedOut = amounts[amounts.length - 1];
    return expectedOut * (10000 - slippageBPS) / 10000;
}
```

#### swapTokensForExactTokens

This function calculates how many input tokens are needed to receive an exact amount of output tokens.

```solidity
function swapTokensForExactTokens(
    uint amountOut,
    uint amountInMax,
    address[] calldata path,
    address to,
    uint deadline
) external {
    IERC20(path[0]).approve(address(router), amountInMax);
    
    router.swapTokensForExactTokens(
        amountOut,
        amountInMax,
        path,
        to,
        deadline
    );
}
```

**Key Difference**: You specify the exact output you want, and the contract calculates the required input, up to your maximum limit. This is useful when you need a specific amount of tokens (e.g., to repay a loan).

#### swapExactETHForTokens and swapExactTokensForETH

These functions handle swaps involving ETH (WETH actually).

```solidity
// Swap exact ETH for tokens
function swapExactETHForTokens(
    uint amountOutMin,
    address[] calldata path,
    address to,
    uint deadline
) external payable {
    router.swapExactETHForTokens(
        amountOutMin,
        path,
        to,
        deadline
    );
}

// Swap exact tokens for ETH
function swapExactTokensForETH(
    uint amountIn,
    uint amountOutMin,
    address[] calldata path,
    address to,
    uint deadline
) external {
    IERC20(path[0]).approve(address(router), amountIn);
    
    router.swapExactTokensForETH(
        amountIn,
        amountOutMin,
        path,
        to,
        deadline
    );
}
```

**Important**: For `swapExactETHForTokens`, you must send ETH with the transaction (`msg.value` should equal the amount you want to swap).

#### swapExactTokensForTokensSupportingFeeOnTransferTokens

This variant handles tokens that charge fees on transfers (like some scam tokens or tokens with reflection mechanisms).

```solidity
function swapExactTokensForTokensSupportingFeeOnTransferTokens(
    uint amountIn,
    uint amountOutMin,
    address[] calldata path,
    address to,
    uint deadline
) external {
    IERC20(path[0]).approve(address(router), amountIn);
    
    router.swapExactTokensForTokensSupportingFeeOnTransferTokens(
        amountIn,
        amountOutMin,
        path,
        to,
        deadline
    );
}
```

**How it differs**: The normal swap function checks that the exact `amountIn` arrives at the pair. Fee-on-transfer tokens send less than expected, causing reverts. This function uses the actual amount received in the pair, which may be less than `amountIn`.

---

### Add Liquidity Operations

Providing liquidity earns you trading fees proportional to your share of the pool. You must deposit both tokens in the correct ratio.

#### addLiquidity

This function deposits two ERC20 tokens into a pool and receives LP tokens in return.

```solidity
function addLiquidity(
    address tokenA,
    address tokenB,
    uint amountADesired,
    uint amountBDesired,
    uint amountAMin,
    uint amountBMin,
    address to,
    uint deadline
) external returns (uint amountA, uint amountB, uint liquidity) {
    IERC20(tokenA).approve(address(router), amountADesired);
    IERC20(tokenB).approve(address(router), amountBDesired);
    
    (amountA, amountB, liquidity) = router.addLiquidity(
        tokenA,
        tokenB,
        amountADesired,
        amountBDesired,
        amountAMin,
        amountBMin,
        to,
        deadline
    );
}
```

**How LP tokens are minted**: The number of LP tokens you receive depends on your share of the pool's total liquidity. The formula is approximately:

```
LP tokens minted = (Your contribution / Current pool reserves) × Total LP supply
```

**Optimal amounts calculation**: The router automatically calculates the optimal amounts based on current reserves. You specify desired amounts, but the actual deposited amounts may be less to maintain the correct ratio. The excess tokens are returned to you.

**Slippage protection**: Use `amountAMin` and `amountBMin` to protect against unfavorable price movements during your transaction. Setting these too low risks front-running, while setting them too high may cause the transaction to fail if the pool's ratio shifts.

**Example calculation**: If a pool has 1000 USDC and 1 WETH, the optimal ratio is 1000:1. If you want to add 200 USDC, you should add about 0.2 WETH. Your LP tokens would be (200/1000) × current supply = 20% of current supply if you're the only depositor that block.

#### addLiquidityETH

This function handles pools that include ETH/WETH.

```solidity
function addLiquidityETH(
    address token,
    uint amountTokenDesired,
    uint amountTokenMin,
    uint amountETHMin,
    address to,
    uint deadline
) external payable returns (uint amountToken, uint amountETH, uint liquidity) {
    IERC20(token).approve(address(router), amountTokenDesired);
    
    (amountToken, amountETH, liquidity) = router.addLiquidityETH{value: msg.value}(
        token,
        amountTokenDesired,
        amountTokenMin,
        amountETHMin,
        to,
        deadline
    );
}
```

**Key differences**:
- You send ETH as `msg.value` along with the call
- The token path is [token, WETH, router.WETH()]
- The router automatically wraps your ETH into WETH

**Important**: `msg.value` should equal the amount of ETH you want to deposit (or slightly more for gas), and the router will handle wrapping and unwrapping automatically.

---

### Remove Liquidity Operations

Removing liquidity burns your LP tokens and returns the underlying tokens in proportion to your share.

#### removeLiquidity

```solidity
function removeLiquidity(
    address tokenA,
    address tokenB,
    uint liquidity,
    uint amountAMin,
    uint amountBMin,
    address to,
    uint deadline
) external returns (uint amountA, uint amountB) {
    router.removeLiquidity(
        tokenA,
        tokenB,
        liquidity,
        amountAMin,
        amountBMin,
        to,
        deadline
    );
}
```

**How LP tokens are burned**: You specify how many LP tokens to burn. The contract calculates your share of the current reserves and transfers that amount to you.

**Proportional withdrawal**: The amounts you receive are proportional to your share of the total LP supply. The formula is:

```
AmountA = (liquidity burned / total supply) × reserveA
AmountB = (liquidity burned / total supply) × reserveB
```

**Important**: After burning, your LP tokens are transferred to the Pair contract and effectively destroyed (total supply decreases). You cannot redeem the same LP tokens again.

#### removeLiquidityETH

```solidity
function removeLiquidityETH(
    address token,
    uint liquidity,
    uint amountTokenMin,
    uint amountETHMin,
    address to,
    uint deadline
) external returns (uint amountToken, uint amountETH) {
    router.removeLiquidityETH(
        token,
        liquidity,
        amountTokenMin,
        amountETHMin,
        to,
        deadline
    );
}
```

This returns both the ERC20 token and unwrapped ETH.

#### removeLiquidityWithPermit and removeLiquidityETHWithPermit

These variants allow gasless approval using signed permits instead of separate `approve` transactions.

```solidity
function removeLiquidityWithPermit(
    address tokenA,
    address tokenB,
    uint liquidity,
    uint amountAMin,
    uint amountBMin,
    address to,
    uint deadline,
    bool approveMax,
    uint8 v,
    bytes32 r,
    bytes32 s
) external {
    router.removeLiquidityWithPermit(
        tokenA,
        tokenB,
        liquidity,
        amountAMin,
        amountBMin,
        to,
        deadline,
        approveMax,
        v,
        r,
        s
    );
}
```

**How permits work**: Instead of calling `approve` first, you sign a message saying "I authorize this spender to withdraw up to X tokens from my account." This can be done off-chain and submitted with the main transaction, saving gas.

The `approveMax` parameter indicates whether to set the allowance to `type(uint256).max` (infinite) or to the exact amount being withdrawn.

---

## Best Practices and Warnings

### Common Pitfalls to Avoid

**Never use `transfer` or `send` for token transfers**: These functions only forward 2300 gas and fail with tokens that require more gas. Always use `transferAndCall` or direct `call` with sufficient gas.

**Always check return values**: Some tokens don't revert on failure but return false. Your code should handle both scenarios.

**Reentrancy protection**: Even though Uniswap contracts have reentrancy guards, your custom contracts should implement them too when handling external calls.

**Approval race conditions**: When using `permit`, be aware that signature validity depends on nonces and chain IDs. Always verify these parameters.

### Gas Optimization Tips

**Use `permit`**: The gas savings from using `permit` instead of separate `approve` transactions can be substantial, typically 20-30% for liquidity operations.

**Batch operations**: When adding or removing liquidity from multiple pools, consider batching operations where possible.

**Deadline management**: Setting deadlines too far in the future (e.g., `block.timestamp + 365 days`) can allow transactions to execute under unfavorable conditions for extended periods.

### Testing Recommendations

Always test on Sepolia first before deploying to mainnet. Use a variety of scenarios:
- Normal swaps with sufficient liquidity
- Swaps with minimal liquidity (slippage testing)
- Max slippage scenarios
- Token transfer failures
- Fee-on-transfer tokens
- Deadline expiration
- Reentrancy attempts

---

## References and Resources

### Official Documentation
- Uniswap V2 Documentation: https://uniswap.org/docs/v2
- Core Contracts Repository: https://github.com/Uniswap/v2-core
- Periphery Contracts Repository: https://github.com/Uniswap/v2-periphery
- V2 Specification: https://uniswap.org/docs/v2/specification/

### Additional Resources
- Uniswap V2 Whitepaper: https://uniswap.org/whitepaper.pdf
- Ethereum Block Explorers: sepolia.etherscan.io, etherscan.io
- Solidity Documentation: https://soliditylang.org/
- OpenZeppelin Contracts: https://openzeppelin.com/contracts/

### Community and Support
- Uniswap Governance Forum: https://forum.uniswap.org
- Ethereum Stack Exchange: https://ethereum.stackexchange.com
- Solidly: https://soliditylang.org/

---

*This documentation is intended for educational purposes. Always perform thorough testing and security audits before deploying contracts to mainnet. Uniswap V2 contracts are complex financial instruments that require careful handling.*