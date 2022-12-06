# 51-attack

This project has migrated to [Michael Keleti's repository.](https://github.com/mkeleti/docker-eth-attack)

## How it happened

When the DAO (Decentralized Autonomous Org) was first implemented on the ethereum blockchain it had
a severe vulnerability. This vulnerability was initially missed by its creators and developers had already began creating their own DAOs.

The most invested in DAO was just called 'The DAO', this was based off of the [original DAO Framework](https://github.com/blockchainsllc/DAO). This severe vulnerability involved recursively calling a function that drains the DAO's ether wallet. This vulnerability is described [here](https://medium.com/ursium-blog/no-dao-funds-at-risk-following-the-ethereum-smart-contract-recursive-call-bug-discovery-29f482d348b). As the creators of ethereum were writing this blog post about fixing this critical vulnerability, the attack was already in progress.

The vulnerability exists within the SplitDao function in the Dao.sol file of the original framework.By recursively calling this function repeatedly, an attacker can drain the contents of the DAO. This is called a recursive send pattern. I have provided a copy of the splitDao function below:

```
function splitDAO(
  uint _proposalID,
  address _newCurator
) noEther onlyTokenholders returns (bool _success) {

  ...
  // XXXXX Move ether and assign new Tokens.  Notice how this is done first!
  uint fundsToBeMoved =
      (balances[msg.sender] * p.splitData[0].splitBalance) /
      p.splitData[0].totalSupply;
  if (p.splitData[0].newDAO.createTokenProxy.value(fundsToBeMoved)(msg.sender) == false) // XXXXX This is the line the attacker wants to run more than once
      throw;

  ...
  // Burn DAO Tokens
  Transfer(msg.sender, 0, balances[msg.sender]);
  withdrawRewardFor(msg.sender); // be nice, and get his rewards
  // XXXXX Notice the preceding line is critically before the next few
  totalSupply -= balances[msg.sender]; // XXXXX AND THIS IS DONE LAST
  balances[msg.sender] = 0; // XXXXX AND THIS IS DONE LAST TOO
  paidOut[msg.sender] = 0;
  return true;
} 
```

*The basic idea is this: propose a split. Execute the split. When the DAO goes to withdraw your reward, call the function to execute a split before that withdrawal finishes. The function will start running without updating your balance, and the line we marked above as "the attacker wants to run more than once" will run more than once. What does that do? Well, the source code is in TokenCreation.sol, and it transfers tokens from the parent DAO to the child DAO. Basically the attacker is using this to transfer more tokens than they should be able to into their child DAO.*

- [Hacking, Distributed](https://hackingdistributed.com/2016/06/18/analysis-of-the-dao-exploit/)

## How to implement

From this information we can derive a few toplevel steps in order to initiate this attack on our own.

1. Run a local ethereum chain using geth (frontier release).

2. Implement our own DAO on the chain using ether drawn from a faucet and the original vulnerable framework.

3. Create a child 'attacker' DAO to draw funds into.

4. Initiate the attack.

Once we have this finished we just need to create a few methods and connect to an API:

- REVERT: Simulates the fork that took place after the DAO attack, reverting the chain to a lower blockheight

- WALLET: Returns an object that contains the wallet balances of both the DAO and the attacker

- ATTACK: Attacks the DAO with the recursive send pattern, draining the DAO to the attacker. Loops through WALLET calls so that the Attack can be witnessed in realtime.

This API can then be called so as to actively simulate this attack repeatedly.


