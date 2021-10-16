# Coding the Arbitrage Trading Bot
In the previous quest, we installed the dependencies & tools needed to build the bot and understood how arbitrage opportunities can be exploited to make money. Now lets begin coding the bot!
## Code: Constants \+ Utilities
Create a folder “scripts” and in that folder create arbitrageBot.js. This is where we will add js code to interact with the DEXs and swap the tokens.

First, let's import all the classes we will need for interacting with smart contracts, uniswap & sushiswap interfaces. Ethers(introduced in the previous quest of this series) is the library we use to interact with smart contracts. 

Every cryptocurrency can be broken into [smaller units of transfer](https://www.linkedin.com/pulse/cryptocurrency-base-units-tara-annison/?trk=related_artice_Cryptocurrency%20Base%20Units_article-card_title). 1 Ether = 10^18 Wei = 1000,000,000,000,000,000 wei. The power of 10, in this case 18, is referred to as the decimal of the cryptocurrency. Bitcoin has 7 decimals, which means 1 Bitcoin = 10^7 Satoshi. Javascript integer data type does not support such large numbers. Hence we use the [BigNumber](https://docs.ethers.io/v5/api/utils/bignumber/) class from ethers library, to represent and operate on these large numerical value types, when transacting with smaller units.

```

 const ethers = require( 'ethers' );

const BigNumber = ethers.BigNumber;

```

Next, let's import classes from uniswap sdk, which are needed to represent a trade between any two tokens. 

```

const {

    Fetcher,

    Token,

    WETH,

    ChainId,

    TradeType,

    Percent,

    Route,

    Trade,

    TokenAmount,

} = require( '@uniswap/sdk' );

```

The [Token](https://docs.uniswap.org/sdk/2.0.0/reference/token) entity represents an ERC-20 token at a specific address on a specific chain.The [Trade](https://docs.uniswap.org/sdk/2.0.0/reference/trade) entity represents a fully specified trade along a route. It supplies all the information necessary to craft a router transaction. The [Route](https://docs.uniswap.org/sdk/2.0.0/reference/route) entity represents one or more ordered Uniswap token pairs with a fully specified path from input token to output token.[Fetcher](https://docs.uniswap.org/sdk/2.0.0/reference/fetcher)  contains static methods for constructing instances of pairs and tokens from on-chain data. [ChainId](https://docs.uniswap.org/protocol/reference/periphery/libraries/ChainId) represents the ID of the blockchain we want to use, in our case it is MAINNET.

Ether, the native currency used to fuel computation and data storage within the Ethereum Virtual Machine, does not conform to the ERC20 token interface. Hence [WETH](https://blog.0xproject.com/canonical-weth-a9aa7d0279dd) is used to abstract ether as an ERC20 compliant ether token for swapping and simplify smart contracts by eliminating special business logic for handling ETH.

Next, let’s declare constants needed to establish the connection to the mainnet fork we created using HardHat in the previous quest of this series. We need to connect to the RPC server at localhost:8545 and use any testAccount and privateKey such that the account has enough balance to enable testing our bot. A [Provider](https://docs.ethers.io/v5/api/providers/) is an abstraction of a connection to the Ethereum network, in our case we need [JsonRPCProvider](https://docs.ethers.io/v5/api/providers/jsonrpc-provider/) to connect to localhost RPC :

```

const providerURL = 'http://localhost:8545'; 

const provider = new ethers.providers.JsonRpcProvider( providerURL );

const testAccountAddress = '0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266';

const privateKey = '0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80';

```

Let’s also create constants for [ETH address on mainnet](https://etherscan.io/address/0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee), maximum possible value of UInt256 in ethers library and zero. These will be used later, to change the maximum number of tokens, which are allowed to be transferred by DEXes. By default, every DEX needs permission to withdraw tokens (except Ether, which is allowed by default) from our account and this is called allowance.

```

const ETH_ADDRESS = "0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee";

const MAX_UINT256 = ethers.constants.MaxUint256;

const ZERO_BN = ethers.constants.Zero;

```

Now you are ready to create references to ERC20 tokens using the Token class. All you need is smart contract addresses of the ERC20 token on mainnet.

```

const Tokens = {

    WETH: WETH\[ ChainId.MAINNET \],

    DAI: new Token( ChainId.MAINNET, '0x6B175474E89094C44Da98b954EedeAC495271d0F', 18 , 'DAI'),

    BAT: new Token( ChainId.MAINNET, '0x2E642b8D59B45a1D8c5aEf716A84FF44ea665914', 18, 'BAT') ,

    MKR : new Token ( ChainId.MAINNET, '0x9f8F72aA9304c8B593d555F12eF6589cC3A579A2', 18, 'MKR')

};

```

Next, lets create a utility function to create reference to the account wallet we are using:

```

const wallet  = getWallet(privateKey);

function getWallet( privateKey ) {

    return new ethers.Wallet( privateKey, provider )

}

```

Whenever we initiate a swap transaction, we are at risk of slippage as explained in the previous quest, so most of the swap functions ask us for a deadline of executing the swap so that slippage can be minimised. So, adding a utility function to get deadline of any transaction, in this case, its 10 min from now:

```

const getDeadlineAfter = delta =>

    Math.floor( Date.now() / 1000 ) \+ ( 60 \* Number.parseInt( delta, 10 ) );

```

Many functions need hex input, so let’s create a utility to convert a number into hex string:

```

const toHex = n => `0x${ n.toString( 16 ) }`;

```

All transaction amounts are denoted in the smallest unit like wei for Ether, satoshi for Bitcoin. But it is not easily readable by humans (think of ‘1 ether’ vs ‘1000,000,000,000,000,000 wei’). Hence, for showing the transferred amount to users, we need to format the amount by adding decimal representation and converting the smallest unit like wei to a bigger unit like Ether. The function ethers.utils.formatUnits() enables us to do that. Similarly for all function params, we need to specify the amount in smallest units like wei or satoshi, which means input values are going to be larger integer values. As you already know, javascript does not support such large numbers, so we are using BigNumber.from(hexStringRepresentationOfAmount) to represent them. Add some functions to do these conversions and getting the balance of every token in human readable form:

```

async function getTokenBalanceInBN(address, tokenContract) {

    const balance = await tokenContract.balanceOf(address);

    return BigNumber.from(balance);

}

async function getTokenBalance(address, tokenContract) {

    const balance = await tokenContract.balanceOf(address);

    const decimals = await tokenContract.decimals();

    return ethers.utils.formatUnits(balance, decimals);

}

async function printAccountBalance(address, privateKey) {

    const balance = await provider.getBalance(address);

    const wethBalance = await getTokenBalance(address, wethContract);

    const daiBalance = await getTokenBalance(address, daiContract);

    const mkrBalance = await getTokenBalance(address, mkrContract);

    const batBalance = await getTokenBalance(address, batContract);

    console.log(`Account balance: ${ethers.utils.formatUnits(balance,18)} ethers, ${wethBalance} weth, ${daiBalance} DAI, ${mkrBalance} MKR, ${batBalance} BAT`);

}

```

In order to create a reference to any smart contract, we need its address on the blockchain we are using and its ABI. This means we would need to search the smart contract address of uniswap, sushiswap, DAI, MKR on the blockchain. [Etherscan](https://etherscan.io/) is a great tool for discovering smart contracts. All you need to do is google the address then verify on etherscan. For example, google, “uniswap v2 router address” and select the search result link of etherscan, verify the name under the “contract” tab and copy the smart contract address. 

![](https://qb-content-staging.s3.ap-south-1.amazonaws.com/public/fb231f7d-06af-4aff-bca3-fd51cb633f77/3d7ce29b-f16b-4d78-8cf0-fa10af25a0e0.jpg)

Next, scroll down in the “contract” tab of the etherscan page for uniswap and copy the contract ABI. Now create the reference to the uniswap v2 router in js using ethers.Contract() function. Its takes the wallet address to connect this smart contract to, so that all transactions can be linked to this wallet whose private key we provide:

```

const signer = new ethers.Wallet( privateKey ); 
const  uniswapRouter = new ethers.Contract(
            ‘<uniswapAddress>’,
           ‘<uniswapABI>’,
            signer.connect( provider )
        )


```

To save our time, let's use a utility function to create smart contracts given any input address, abi. If we have the privateKey of the account then we can also connect the smart contract to the wallet of the account we want to use for all our transactions.

```

function constructContract( smAddress, smABI, privateKey ) {

    const signer = new ethers.Wallet( privateKey );

    return new ethers.Contract(

            smAddress,

            smABI,

            signer.connect( provider )

        )

}

```

Next, let’s create smart contract references for all the entities we will interact with like uniswap, sushiswap, DAI token on uniswap mainnet, MKR token on uniswap mainnet, etc.

 To ensure readability, let’s move all ABI constants in a new separate file ABIConstants.js and then include it in arbitrageBot.js. Add ABI definitions in ABIConstants.js file such that it has the ABI for uniswap, sushiswap, IERC20 token, like this:

[https://gist.github.com/krati-creatoros/c0bfd976d246dbdd090c1df12cbeb5be\#file-abiconstants-js](https://gist.github.com/krati-creatoros/c0bfd976d246dbdd090c1df12cbeb5be#file-abiconstants-js)

In arbitrageBot.js, declare smart contracts for uniswap, sushiswap, DAI, MKR etc :

```

const wethContract = constructContract(Tokens.WETH.address, IERC20_ABI, privateKey);

const daiContract = constructContract(Tokens.DAI.address, IERC20_ABI, privateKey);

const mkrContract = constructContract(Tokens.MKR.address, IERC20_ABI, privateKey);

const batContract = constructContract(Tokens.BAT.address, IERC20_ABI, privateKey);

const uniswap = constructContract(   //v2 router

    '0x7a250d5630B4cF539739dF2C5dAcb4c659F2488D',

    uniswapABI,

    privateKey,               

);

const sushiswap  = constructContract(

    '0xd9e1cE17f2641f24aE83637ab66a2cca9C378B9F',

    sushiswapABI,

    privateKey,

);

```
## Code: Swap \+ Arbitrage evaluation functions
Before swapping, we will check the token transfer allowance and change it to the amount we want to transfer. Remember from the previous subquest, that by default any smart contract is allowed to transfer Eth but not any other tokens, unless the user specifies the max amount for the token. Here is the function to check and approve token allowance. Basically every ERC20 smart contract holds a mapping of max amount for every user account address. Here we are changing that mapping using approve() function of the ERC20 contract:

```

async function checkAndApproveTokenForTrade(srcTokenContract, userAddress, srcQty, factoryAddress) {

  console.log(`Evaluating : approve ${srcQty} tokens for trade`);

  if (srcTokenContract.address == ETH_ADDRESS) {

    return;

  }

  let existingAllowance = await srcTokenContract.allowance(userAddress, factoryAddress);

    console.log(`Existing allowance ${existingAllowance}`);

  if (existingAllowance.eq(ZERO_BN)) {

    console.log(`Approving contract to max allowance ${srcQty}`);

    await srcTokenContract.approve(factoryAddress, srcQty);

  } else if (existingAllowance.lt(srcQty)) {

    // if existing allowance is insufficient, reset to zero, then set to MAX_UINT256

    //setting approval to 0 and then to a max is suggestible since  if the address already has an approval, 

    //setting again to a max would bump into error

    console.log(`Approving contract to zero, then max allowance ${srcQty}`);

    await srcTokenContract.approve(factoryAddress, ZERO_BN);

    await srcTokenContract.approve(factoryAddress, srcQty);

  } 

  return;

}

```

Uniswap’s swap functions need a minimum amount of output token to be specified, so that swap can be reverted if exchanged token amount is less than this amount. To find out what is the minimum output token amount we will be profitable with, we should also consider a slippage percent, in our case 0.5%. So all we need to do is, fetch pair data for the tokens we want to swap and then create a Trade entity with them. The trade entity function minimumAmountOut takes slippage as input and gives us the minimum amount we can possibly receive after the swap. If output token amount after the swap is lower than this amount, the swap will be reverted by the DEX.

```

async function constructTradeParameters( tokenA, tokenB, tokenAmount ) {

    const slippageTolerance = new Percent( '50', '100' );

    const pair = await Fetcher.fetchPairData( tokenA, tokenB );

    const route = new Route( \[ pair \], tokenA );

    const trade = new Trade(

        route,

        new TokenAmount( tokenA, tokenAmount ),

        TradeType.EXACT_INPUT,

    );

    const minimumAmountOut = trade.minimumAmountOut( slippageTolerance );

    console.log(`minimumAmountOut is ${minimumAmountOut.raw}`);

    

    return {

        amountOutMin : minimumAmountOut,

        amountOutMinRaw : minimumAmountOut.raw,

        value: toHex( trade.inputAmount.raw )

    };

}

```

Let's create the swap function, which takes 2 Token references to be swapped and the smart contract of the DEX we want to swap on. This will be 2 steps: 

1. Check and approve allowance to transfer Token A from our account on DEX. 
2. Swap tokenA with TokenB on DEX and print transaction details

```

async function swap(tokenA, tokenB, userAddress, tokenAContract, dexContract) {

    const inputTokenAmount = await getTokenBalanceInBN(userAddress, tokenAContract);

    const {

        amountOutMin,

        amountOutMinRaw,

        value

    } = await constructTradeParameters( tokenA , tokenB , inputTokenAmount);

    console.log(`Going to swap ${ethers.utils.formatUnits(inputTokenAmount, 18)} ${tokenA.symbol} tokens for ${amountOutMinRaw} ${tokenB.symbol}`);

    await checkAndApproveTokenForTrade(tokenAContract, wallet.address, inputTokenAmount, dexContract.address);

    console.log("Swapping..");

    const tx = await dexContract.swapExactTokensForTokens(

        inputTokenAmount,

        toHex(amountOutMinRaw),

        \[ tokenA.address, tokenB.address\],

        userAddress,

        getDeadlineAfter( 20 ),

        { gasLimit: 300000}

        );

    await printTxDetails(tx);

    await printAccountBalance(userAddress);

}

async function printTxDetails(tx) {

    console.log(`Transaction hash: ${tx.hash}`);

    const receipt = await tx.wait();

    console.log(`Transaction was mined in block ${receipt.blockNumber}`);

}

```

Now that we have functions for fetching trade parameters, swapping tokens and utility functions to print account balance, print transaction details etc, let’s integrate them correctly to create searchProfitableArbitrage() function, which can evaluate if a profitable arbitrage exists for any 2 tokens. For any token pair TokenA and TokenB, the arbitrage opportunity exists if we can make profits using one of these two transaction sequence:

1. Swap Token A on uniswap with Token B. Then Swap Token B with Token A on sushiswap. Include the gasFee. If you end up with a higher TokenA amount than what you started with, it's a profit :) \[Profit 1\]
2. Swap Token A on sushiswap with Token B. Then Swap Token B with Token A on uniswap. Include the gasFee. If you end up with a higher TokenA amount than what you started with, it's a profit :) \[Profit 2\]

getAmountsOut() function of uniswap/sushiswap gives us the amount of tokenB that we will get by swapping tokenA input amount. This is the exchange rate using which we will calculate the buy/sell rates and then eventually calculate profit 1 and profit 2.

```

async function searchProfitableArbitrage(args) {

    const { inputToken, outputToken, inputTokenContract, outputTokenContract } = args

    const inputTokenSymbol = inputToken.symbol

    const outputTokenSymbol = outputToken.symbol

    const tradeAmount = BigNumber.from("1000000000000000000");

    const uniRates1 = await uniswap.getAmountsOut(tradeAmount, \[ inputToken.address, outputToken.address\]);

    console.log(`Uniswap Exchange Rate: ${ethers.utils.formatUnits(uniRates1\[0\], 18)} ${inputTokenSymbol} = ${ethers.utils.formatUnits(uniRates1\[1\], 18)} ${outputTokenSymbol}`);

    const uniRates2 = await uniswap.getAmountsOut(tradeAmount, \[ outputToken.address, inputToken.address\]);

    console.log(`Uniswap Exchange Rate: ${ethers.utils.formatUnits(uniRates2\[0\], 18)} ${outputTokenSymbol} = ${ethers.utils.formatUnits(uniRates2\[1\], 18)} ${inputTokenSymbol}`);

    const sushiRates1 = await sushiswap.getAmountsOut(tradeAmount, \[ inputToken.address, outputToken.address\]);

    console.log(`Sushiswap Exchange Rate: ${ethers.utils.formatUnits(sushiRates1\[0\], 18)} ${inputTokenSymbol} = ${ethers.utils.formatUnits(sushiRates1\[1\], 18)} ${outputTokenSymbol}`);

    const sushirates2 = await sushiswap.getAmountsOut(tradeAmount, \[ outputToken.address, inputToken.address\]);

    console.log(`Sushiswap Exchange Rate: ${ethers.utils.formatUnits(sushiRates2\[0\], 18)} ${outputTokenSymbol} = ${ethers.utils.formatUnits(sushiRates2\[1\], 18)} ${inputTokenSymbol}`);

    

    const sushiswapRates = {

        buy: sushiRates1\[1\],

        sell: sushiRates2\[1\]

    };

    

    const uniswapRates = {

        buy: uniRates1\[1\],

        sell: uniRates2\[1\]

    };

    // profit1 = profit if we buy input token on uniswap and sell it on sushiswap

    const profit1 = tradeAmount \* (uniswapRates.sell - sushiswapRates.buy - gasPrice \* 0.003);

    // profit2 = profit if we buy input token on sushiswap and sell it on uniswap

    const profit2 = tradeAmount \* (sushiswapRates.sell - uniswapRates.buy - gasPrice \* 0.003);

      

    console.log(`Profit from UniswapSushiswap : ${profit1}`)

    console.log(`Profit from SushiswapUniswap : ${profit2}`)

    if(profit1 > 0 && profit1 > profit2) {

        //Execute arb Uniswap  Sushiswap

        console.log(`Arbitrage Found: Make ${profit1} : Sell ${inputTokenSymbol} on Uniswap at ${uniswapRates.sell} and Buy ${outputTokenSymbol} on Sushiswap at ${sushiswapRates.buy}`);

        await swap(inputToken, outputToken, testAccountAddress, inputTokenContract, uniswap);

        await swap(outputToken, inputToken, testAccountAddress, outputTokenContract, sushiswap);

    } else if(profit2 > 0) {

        //Execute arb Sushiswap  Uniswap

        console.log(`Arbitrage Found: Make ${profit2} : Sell ${inputTokenSymbol} on Sushiswap at ${sushiswapRates.sell} and Buy ${outputTokenSymbol} on Uniswap at ${uniswapRates.buy}`);

    

        await swap(inputToken, outputToken, testAccountAddress, inputTokenContract, sushiswap);

        await swap(outputToken, inputToken, testAccountAddress, outputTokenContract, uniswap);

    }

}

```

Last, let's call searchProfitableArbitrage() from another function: monitorPrice(), which monitors exchange rates across uniswap and sushiswap. Then invoke it every 1 second so that we can poll the prices. From within this function, let's first print the balance of different currencies in our account so that we understand how our wallet holds.

```

let isMonitoringPrice = false

async function monitorPrice() {

  if(isMonitoringPrice) {

    return

  }

  await printAccountBalance(testAccountAddress);

  console.log("Checking prices for possible arbitrage opportunities...")

  isMonitoringPrice = true

  try {

    await searchProfitableArbitrage({

      inputToken: Tokens.DAI,

      outputToken: Tokens.MKR

    })

  } catch (error) {

    console.error(error)

    isMonitoringPrice = false

    return

  }

  isMonitoringPrice = false

}

let priceMonitor = setInterval(async () => { await monitorPrice() }, 1000)

```

But wait! The test account that we are using has 10,000 Eth and 0 DAI. In order to trade DAI for MKR, we will need to have an initial DAI balance in our account which is sufficient for any arbitrage trade. Let's convert 2 ETH into DAI as a 1 time operation in monitorPrice() using a function for swapping Eth to Token:

```

async function swapEthToToken(ethAmount, token, userAddress, dexContract) {

    const {

        amountOutMin,

        amountOutMinRaw,

        value

    } = await constructTradeParameters( Tokens.WETH, token, ethAmount );

    console.log(`Going to swap ${ethAmount} ETH for ${token.symbol} tokens`);

    const tx = await dexContract.swapExactETHForTokens(

        toHex(amountOutMinRaw),

        \[ Tokens.WETH.address, token.address \],

        userAddress,

        getDeadlineAfter( 20 ),

        { value }

    );

    await printTxDetails(tx);

    await printAccountBalance(userAddress);

}

```

Here is the updated monitorPrice() :

```

let isMonitoringPrice = false

let isInitialTxDone = false

async function monitorPrice() {

  if(isMonitoringPrice) {

    return

  }

if (!isInitialTxDone) {

    isInitialTxDone = true

    // convert DAI from ETH 

    const twoEther = BigNumber.from("2000000000000000000");

    console.log(ethers.utils.formatUnits(twoEther));

    await printAccountBalance(testAccountAddress);

    await swapEthToToken(twoEther, Tokens.DAI, testAccountAddress, uniswap);

}

  await printAccountBalance(testAccountAddress);

  console.log("Checking prices for possible arbitrage opportunities...")

  isMonitoringPrice = true

  try {

    await searchProfitableArbitrage({

      inputToken: Tokens.DAI,

      outputToken: Tokens.MKR,

      inputTokenContract: daiContract, 

      outputTokenContract: mkrContract

    });

  } catch (error) {

    console.error(error)

    isMonitoringPrice = false

    return

  }

  isMonitoringPrice = false

}

let priceMonitor = setInterval(async () => { await monitorPrice() }, 1000)

```

8. That's all folks! The code is ready! This is how the complete bot code looks:

[https://gist.github.com/krati-creatoros/c0bfd976d246dbdd090c1df12cbeb5be](https://gist.github.com/krati-creatoros/c0bfd976d246dbdd090c1df12cbeb5be)
## Run the bot
To run the bot, let's use someone else’s account. Someone super rich like Vitalik

([https://etherscan.io/address/0xab5801a7d398351b8be11c439e05c5b3259aec9b](https://etherscan.io/address/0xab5801a7d398351b8be11c439e05c5b3259aec9b)) or any of the test accounts listed in the local server terminal (refer to the subquest before).

All you need to do is replace the account and private key in testAccountAddress and privateKey constants we created in our js code and they will automatically link this users wallet to all smart contract references we have created in code. What does this mean? This means the wallet associated with uniswap or sushiswap will be the one whose privateKey and account number we specified and all transactions will be done on that wallet. This is known as impersonation. Following command creates a reference to the impersonated account:

const impersonatedAccount = await ethers.getSigner("");

In a new terminal, deploy the bot using following commands, which uses network as local. Refer to hardhat.config.js to see if you have defined ‘local’ under ‘networks’ properly as mentioned in the previous quest of this series :

npx hardhat run scripts/ArbitrageBot.js --network local

![](https://qb-content-staging.s3.ap-south-1.amazonaws.com/public/fb231f7d-06af-4aff-bca3-fd51cb633f77/c28c0993-35d5-40b1-91ef-2f95c45527b1.jpg)

Observe how the bot displays the account balance first and then interacts with uniswap and sushiswap to fetch exchange rates.

![](https://qb-content-staging.s3.ap-south-1.amazonaws.com/public/fb231f7d-06af-4aff-bca3-fd51cb633f77/a0e6a38f-6a52-46dd-912b-5fe2bc98ce73.jpg)

Whenever there is a profitable arbitrage, it executes the swap and shows the updated account balance too.![](https://qb-content-staging.s3.ap-south-1.amazonaws.com/public/fb231f7d-06af-4aff-bca3-fd51cb633f77/65dc20e2-31c0-4e9b-97fc-f3a03d496d4e.jpg)
## Before we go
Now that you have written your own bot to evaluate and execute opportunities of making money passively, imagine what possibilities are open to you to explore in the world of decentralized finance.

You are now ready to unleash the power of blockchain programming in decentralized finance ! How about modifying this bot to evaluate arbitrage opportunities for more than 2 token exchanges? Example: Swap tokenA for lower amount of tokenB at DEX1, swap tokenB for higher amount of tokenC at DEX2, swap tokenC for higher amount of token A at any other DEX so that you make more tokenA than you began with :)

You could also try integrating with other DEXes like paraswap or kyber :) The world is yours my friend!
