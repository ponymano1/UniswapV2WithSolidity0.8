# Uniswap v2 with Solidity Version 0.8

The Solidity version of Uniswap V2 is quite old, and if I want to modify some code according to business needs, compatibility is required. Therefore, I upgraded the Solidity version to 0.8.

# Uniswap V2 Core Code Note
The core code of Uniswap primarily consists of the following parts:
**Factory Contract**: This is where new pair contracts are created. When users want to create a new pair, they call the createPair function of the factory contract.
*Important*: createPair calls create2 to create pair contract.
```
bytes memory bytecode = type(UniswapV2Pair).creationCode;
bytes32 salt = keccak256(abi.encodePacked(token0, token1));
assembly {
    pair := create2(0, add(bytecode, 32), mload(bytecode), salt)
}
```
```
function pairFor(address factory, address tokenA, address tokenB) internal pure returns (address pair) {
    (address token0, address token1) = sortTokens(tokenA, tokenB);
    pair = address(uint160(uint(keccak256(abi.encodePacked(
            hex'ff',
            factory,
            keccak256(abi.encodePacked(token0, token1)),
            hex'b3adfcd878e487d3bb5e7a47d04a2410563ef8162dbd7d432ea2f85650f18731' // init code hash
        )))));
}
```
So, any code change of pair contract need to recalcultate the pair address.

***Pair Contract***: This is where liquidity is stored and managed. Each pair has its own contract, which contains the reserves of two tokens. Users can add or remove liquidity to the pair contract, or make trades.

***Router Contract***: This is the main interface for users to interact with Uniswap. It contains functions for adding liquidity, removing liquidity, and making trades. These functions typically call functions of the pair contract to perform their tasks.


## mathematical model
TokenA * TokenB = K

## UML
```mermaid
classDiagram
    class UniswapV2Router02 {
        +address public immutable factory
        +address public immutable WETH
        +function addLiquidity(...)
        +function removeLiquidity(...)
        +function swapExactTokensForTokens(...)
        +function swapTokensForExactTokens(...)
    }
    class IUniswapV2Factory {
        +function createPair(address tokenA, address tokenB) external returns (address pair)
        +function getPair(address tokenA, address tokenB) external view returns (address pair)
    }
    class IUniswapV2Pair {
        +function mint(address to) external returns (uint liquidity)
        +function burn(address to) external returns (uint amount0, uint amount1)
        +function swap(uint amount0Out, uint amount1Out, address to, bytes calldata data) external
    }
    class IERC20 {
        +function balanceOf(address account) external view returns (uint256)
        +function transfer(address recipient, uint256 amount) external returns (bool)
        +function approve(address spender, uint256 amount) external returns (bool)
    }
    class IWETH is IERC20 {
        +function deposit() external payable
        +function withdraw(uint wad) external
    }
    class UniswapV2Pair is IUniswapV2Pair {
        +function initialize(address, address) external
    }
    class UniswapV2Factory is IUniswapV2Factory {
        +address public feeTo
        +address public feeToSetter
        +function setFeeTo(address) external
        +function setFeeToSetter(address) external
    }
    UniswapV2Router02 --> IUniswapV2Factory : Uses
    UniswapV2Router02 --> IUniswapV2Pair : Uses
    UniswapV2Router02 --> IERC20 : Uses
    UniswapV2Router02 --> IWETH : Uses
    UniswapV2Pair --> IUniswapV2Pair : Implements
    UniswapV2Factory --> IUniswapV2Factory : Implements
```

## Sequence
### addLiquidity
```mermaid
sequenceDiagram
    participant User
    participant UniswapV2Router02
    participant IUniswapV2Factory
    participant IUniswapV2Pair
    participant IERC20
    User->>UniswapV2Router02: addLiquidity(tokenA, tokenB, amountADesired, amountBDesired, amountAMin, amountBMin, to, deadline)
    UniswapV2Router02->>IUniswapV2Factory: getPair(tokenA, tokenB)
    Note over IUniswapV2Factory, UniswapV2Router02: Check if pair exists
    alt Pair does not exist
        UniswapV2Router02->>IUniswapV2Factory: createPair(tokenA, tokenB)
        Note over IUniswapV2Factory, UniswapV2Router02: Pair is created
    end
    UniswapV2Router02->>IERC20: transferFrom(User, UniswapV2Router02, amountADesired)
    UniswapV2Router02->>IERC20: transferFrom(User, UniswapV2Router02, amountBDesired)
    UniswapV2Router02->>IERC20: approve(IUniswapV2Pair, amountADesired)
    UniswapV2Router02->>IERC20: approve(IUniswapV2Pair, amountBDesired)
    UniswapV2Router02->>IUniswapV2Pair: mint(to)
```
#### 
**Note**: addLiquidity not use all of tokenA and tokenB. it uses quote function.
amountB = amountA * ReserveB / ReserveA
```solidity
function quote(uint amountA, uint reserveA, uint reserveB) internal pure returns (uint amountB)
```
The specific method of calculating the value, whether using asset A as the standard to calculate asset B or using asset B as the standard to calculate asset A, depends on which method can provide higher liquidity.

The choice between the two methods is determined by which one can achieve greater liquidity.

#### How to calculate LP token:
The goal is to ensure that the value of the assets provided by the user is proportional to the number of LP tokens they receive.

if amount of LP token == 0
LPToken = sqrt(tokenA * tokenB)
if amount of LP token != 0
LPToken = min{amountA * priceA, amountB * priceB}
LPToken = min{amountA * LP totalSupply/ reserveA, amountB * LP totalSupply/ reserveB}

### Remove Liquidity
```mermaid
sequenceDiagram
    participant User
    participant UniswapV2Router02
    participant UniswapV2Library
    participant UniswapV2Pair
    participant ERC20TokenA
    participant ERC20TokenB
    participant WETH

    User->>UniswapV2Router02: removeLiquidity(tokenA, tokenB, liquidity, amountAMin, amountBMin, to, deadline)
    UniswapV2Router02->>UniswapV2Library: getReserves(factory, tokenA, tokenB)
    UniswapV2Library-->>UniswapV2Router02: reserves
    UniswapV2Router02->>UniswapV2Library: quote(liquidity, totalSupply, reserveA, reserveB)
    UniswapV2Library-->>UniswapV2Router02: amountA, amountB
    UniswapV2Router02->>UniswapV2Router02: require(amountA >= amountAMin && amountB >= amountBMin, 'UniswapV2Router: INSUFFICIENT_OUTPUT_AMOUNT')
    UniswapV2Router02->>UniswapV2Pair: transferFrom(msg.sender, address(this), liquidity)
    UniswapV2Pair->>UniswapV2Pair: burn(address(this))
    UniswapV2Pair->>ERC20TokenA: transfer(to, amountA)
    UniswapV2Pair->>ERC20TokenB: transfer(to, amountB)
    UniswapV2Pair-->>UniswapV2Router02: amountA, amountB
    alt tokenA == WETH
        UniswapV2Router02->>WETH: withdraw(amountA)
        WETH-->>User: amountA
    else
        UniswapV2Router02->>ERC20TokenA: transfer(to, amountA)
    end
    alt tokenB == WETH
        UniswapV2Router02->>WETH: withdraw(amountB)
        WETH-->>User: amountB
    else
        UniswapV2Router02->>ERC20TokenB: transfer(to, amountB)
    end
    UniswapV2Router02-->>User: amountA, amountB
```
tokenA = LPToken / PriceA = LPToken * reserveA / LP totalSupply

tokenB = LPToken / PriceB = LPToken * reserveB / LP totalSupply

### Swap
```mermaid
sequenceDiagram
    participant User
    participant UniswapV2Router02
    participant UniswapV2Library
    participant UniswapV2Factory
    participant UniswapV2Pair
    participant ERC20TokenA
    participant ERC20TokenB
    participant WETH

    User->>UniswapV2Router02: swapExactTokensForTokens(amountIn, amountOutMin, path, to, deadline)
    UniswapV2Router02->>UniswapV2Factory: getPair(tokenA, tokenB)
    UniswapV2Factory-->>UniswapV2Router02: pairAddress
    loop for each pair in path
        UniswapV2Router02->>UniswapV2Pair: getReserves()
        UniswapV2Pair-->>UniswapV2Router02: reserveA, reserveB
        UniswapV2Router02->>UniswapV2Library: getAmountOut(amountIn, reserveIn, reserveOut)
        UniswapV2Library-->>UniswapV2Router02: amountOut
        UniswapV2Router02->>ERC20TokenA: transferFrom(msg.sender, pairAddress, amountIn)
        UniswapV2Router02->>UniswapV2Pair: swap(amount0Out, amount1Out, to, data)
        UniswapV2Pair->>ERC20TokenA: transfer(to, amountOut)
        UniswapV2Pair->>ERC20TokenB: transfer(to, amountOut)
        UniswapV2Pair-->>UniswapV2Router02: amountOut
    end
    UniswapV2Router02->>UniswapV2Router02: require(amountOut >= amountOutMin, 'UniswapV2Router: INSUFFICIENT_OUTPUT_AMOUNT')
    alt tokenA == WETH
        UniswapV2Router02->>WETH: deposit()
        WETH-->>UniswapV2Router02: amountIn
    else
        UniswapV2Router02->>ERC20TokenA: transferFrom(msg.sender, address(this), amountIn)
    end
    alt tokenB == WETH
        UniswapV2Router02->>WETH: withdraw(amountOut)
        WETH-->>User: amountOut
    else
        UniswapV2Router02->>ERC20TokenB: transfer(to, amountOut)
    end
    UniswapV2Router02-->>User: amountOut
```