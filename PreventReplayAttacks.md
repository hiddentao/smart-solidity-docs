# Prevent replay attacks

On July 20th, 2016, Ethereum hard-forked. Due to ideological disagreements some users didn't accept the fork and decided to continue the original blockchain under a new name - Ethereum Classic  and thus split into two networks - Ethereum (ETH) and Ethereum Classic (ETC).

Now, every transaction sent to the Ethereum network can be *replayed* on the ETC network to the same effect, and vice versa. if you send someone (usually the person you sent money to) ETH from an account which held ETH before the hard-fork, then someone else can *replay* your transaction to cause you to lose ETC from the equivalent account on the ETC network.

If your account was created and populated with ETH (e.g. through mining or transfer from an exchange or purchase service) after the hard-fork date then there's nothing to worry about.

In any case, there are steps you can take to protect yourself and others.

## Protect your personal accounts

You can *split* your ETH/ETC holdings using the splitter contract located at [http://etherscan.io/address/0xaa1a6e3e6ef20068f7f8d8c835d2d22fd5116444#code](http://etherscan.io/address/0xaa1a6e3e6ef20068f7f8d8c835d2d22fd5116444#code)

```solidity
contract AmIOnTheFork {
    function forked() constant returns(bool);
}

contract ReplaySafeSplit {
    // Fork oracle to use
    AmIOnTheFork amIOnTheFork = AmIOnTheFork(0x2bd2326c993dfaef84f696526064ff22eba5b362);

    // Splits the funds into 2 addresses
    function split(address targetFork, address targetNoFork) returns(bool) {
        if (amIOnTheFork.forked() && targetFork.send(msg.value)) {
            return true;
        } else if (!amIOnTheFork.forked() && targetNoFork.send(msg.value)) {
            return true;
        }
        throw; // don't accept value transfer, otherwise it would be trapped.
    }

    // Reject value transfers.
    function() {
        throw;
    }
}
```

You can either use the existing contract a the given address above - **0xaa1a6e3e6ef20068f7f8d8c835d2d22fd51164448** - or deploy the contract again yourself to a new address and use it from there.

You call the `split()` method with two addresses - one for ETH should end up, and one for where ETC should end up. Obviously you should use different destination addresses for both.

The `AmIOnTheFork` contract shown above is initialised with address **0x2bd2326c993dfaef84f696526064ff22eba5b362** which, if you [check it out](http://etherscan.io/address/0x2bd2326c993dfaef84f696526064ff22eba5b362#code), contains logic to set its only member variable (a `boolean`) depending on whether its on the forked (ETH) or non-forked (ETC) network:

```solidity
contract AmIOnTheFork {
    bool public forked = false;
    address constant darkDAO = 0x304a554a310c7e546dfe434669c62820b7d83490;
    // Check the fork condition during creation of the contract.
    // This function should be called between block 1920000 and 1921200.
    // Approximately between 2016-07-20 12:00:00 UTC and 2016-07-20 17:00:00 UTC.
    // After that the status will be locked in.
    function update() {
        if (block.number >= 1920000 && block.number <= 1921200) {
            forked = darkDAO.balance < 3600000 ether;
        }
    }
    function() {
        throw;
    }
}
```

This contract's internal `forked` value will not change going forward and so you can rely on it *(unless someone forks to change it :p).

## Protect your contracts

Technically speaking, any contracts created after the hard-fork should not be susceptible to replay attacks since they *should* be created at different addresses on both networks. However, since [the address of a contract could be calculated deterministically](http://ethereum.stackexchange.com/a/761) it's best to play it safe and include replay-protection within your contracts.

