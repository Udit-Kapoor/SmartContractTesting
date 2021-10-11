# Introduction
Hey Reader! Welcome to this Tutorial on Testing a smart contract on Tezos in SmartPy. We will be learning how to run unit tests and make sure our code works before deployment because once deployed, smart contracts are immutable (and we do not want to waste our XTZ tokens).


- Tezos is an open-source blockchain protocol for assets and applications backed by a global community of validators, researchers, and builders. The Tezos protocol is secure, upgradable, and built to last.
- SmartPy is an intuitive and powerful smart contract development platform for Tezos and is available through a Python library for building and analyzing Tezos smart contracts.


# Prerequisites

To complete this tutorial, you will need a basic understanding of the Python programming language ans a know-how of how to write and deploy Tezos Contracts.
I will suggest you take a look at this [Tutorial](https://learn.figment.io/tutorials/using-the-smartpy-ide-to-deploy-tezos-smart-contracts) first.

# Requirements
Any modern Browser!


# Smart Contract
Before we go to the depths of testing the contract , we need to first have a sample contract which is complex enough so that we can understand it's testing.

Below I have taken a simple time based Escrow Contract : 
```python
import smartpy as sp

class Escrow(sp.Contract):
    def __init__(self, owner, fromOwner, counterparty, fromCounterparty, epoch, hashedSecret):
        self.init(fromOwner           = fromOwner,
                  fromCounterparty    = fromCounterparty,
                  balanceOwner        = sp.tez(0),
                  balanceCounterparty = sp.tez(0),
                  hashedSecret        = hashedSecret,
                  epoch               = epoch,
                  owner               = owner,
                  counterparty        = counterparty)

    @sp.entry_point
    def addBalanceOwner(self):
        sp.verify(self.data.owner == sp.sender , "Wrong Owner")
        sp.verify(self.data.balanceOwner == sp.tez(0) , "There is already some stake")
        sp.verify(sp.amount == self.data.fromOwner , "Only the stake amount is allowed")
        self.data.balanceOwner = self.data.fromOwner

    @sp.entry_point
    def addBalanceCounterparty(self):
        sp.verify(self.data.counterparty == sp.sender , "Wrong CounterParty")
        sp.verify(self.data.balanceCounterparty == sp.tez(0) , "There is already some stake")
        sp.verify(sp.amount == self.data.fromCounterparty , "Only the stake amount is allowed")
        self.data.balanceCounterparty = self.data.fromCounterparty

    def claim(self, identity):
        sp.verify(sp.sender == identity , "Wrong Identity. Internal call only")
        sp.send(identity, self.data.balanceOwner + self.data.balanceCounterparty)
        self.data.balanceOwner = sp.tez(0)
        self.data.balanceCounterparty = sp.tez(0)

    @sp.entry_point
    def claimCounterparty(self, params):
        sp.verify(sp.now < self.data.epoch , "Time limit expired")
        sp.verify(self.data.hashedSecret == sp.blake2b(params.secret) , "Wrong Secret Key")
        self.claim(self.data.counterparty)

    @sp.entry_point
    def claimOwner(self):
        sp.verify(self.data.epoch < sp.now , "Time Limit not yet reached")
        self.claim(self.data.owner)
```

An Escrow contract acts as a security between two untrusting parties whenever there is an amount involved.
A task and a deadline is agreed upon and both parties stake some amount **fromOwner** and **fromCounterparty**.In this case we have **owner** and **counterParty** as the two parties and the latter can claim the amount till the deadline **time** does not expire. Once the deadline has expired and if the **counterParty** hasn't claimed the amount yet then **owner** can claim all the amount. 
A secret code is hashed and stored in the contract which acts as the password for the **counterParty** and is revealed to the **counterParty** only after the agreed upon task is completed making sure that both parties are serious about the task!

Now a contract of such high importance must be iron-clad and no bug should allow either of the parties to claim the amount before the specified task is completed. Hence the need for testing it extensively arises.

# Test Scenarios
In SmartPy we can simulate all kinds of transaction possibilities and test out our contract without having to spend a single XTZ.

To implement testinf we must examine the concept of **test scenarios** :

Test scenarios are an important tool to make sure our smart contracts are working correctly.
- A new test is a method marked with `@sp.add_test`
- A new scenario is instantiated by `sp.test_scenario`.
- Scenarios describe a sequence of actions: originating contracts, computing expressions or calling entry points, etc.
- In the online editor of SmartPy.io, the scenario is computed and then displayed as an HTML document on the output panel.

Let's start by defining a method named `test()`:

```python
@sp.add_test(name = "Escrow")
    def test():
        pass
```
Now we need to create a test scenario:

```python
@sp.add_test(name = "Escrow")
def test():
    scenario = sp.test_scenario()
    scenario.h1("Escrow")
```

# Test Accounts

Test Accounts are unique dummy accounts provided to us via SmartPy library so that we can simulate real-world user accounts which will interact with our contract.
It is instatiated the following way:

```python
bob = sp.test_account("Bob")
udit = sp.test_account("Udit")
```
The string parameter acts like a seed phrase so that no two test accounts are same.

Test Accounts have the following properties for us to use:
- *admin*.address
- *admin*.public_key_hash
- *admin*.public_key
- *admin*.secret_key

These represent the values that the user will have in their account. For us the most important is their **Address**.

# Originating a Contract
Now we have the two parties of the Escrow contract namely **bob** and **udit** and we are ready to Originate our contract in our **test scenario**.
According to our contract above we need the following parameters:
- Owner (bob)
- Owner's Stake
- Counter Party (udit)
- Counter Party's Stake
- Time Limit
- Secret

```python
s = sp.pack("SECRETKEY") #String to Bytes
secret = sp.blake2b(s) #Hashing bytes to secret key
ob = Escrow(bob.address, sp.tez(25), udit.address, sp.tez(5), sp.timestamp(1634753427), secret)
scenario += ob
```
Now in our **test scenario** we have added a Smart Contract between two users (bob and udit) with each of them staking 50 XTZ and 5 XTZ respectively with a deadline of 20th October 2021 and a hashed secret key

> You can read up on Human Date to Epoch Timestamp conversion [HERE](https://www.epochconverter.com/).

> I have used blake2b as my cryptographic hash function. Read more about it [HERE](https://www.blake2.net/)

# Run Method
As we know that we can directly call our contract's EntryPoints using the ```.``` operator like ```ob.addBalanceOwner()``` but to simulate the intricate parameters of a real world transaction we user the ```.run()``` method which has the following parameters :

Parameter | Function
--------- | --------
sender | It simulates the user who is sending the transaction to the contract. Sets the value of ```sp.sender```
source | It simulates the source of the transaction. Sets the value of ```sp.source```
amount | It simulates the amount sent by the user in the transaction. Sets the value of ```sp.amount```
now | It simulates the timestamp of the transaction. Sets the value of ```sp.now```
level | It simulates the block level of the transaction. Sets the value of ```sp.level```
chain_id | It simulates the chain_id of the transaction. Sets the value of ```sp.chain_id```
voting_powers | It simulates the voting power of different users in the contract's implementaion. It is a dictionary. Sets the value of ```sp.voting_power```
valid | If we expect a transaction to fail i.e. testing out edge cases we put this parameter as **FALSE** so that the compiler won't throw an error
show | If we do not want to show a transaction in the HTML Output we set this parameter as **FALSE** 
exception | If we expect a transaction to fail then we can also specify the expected exception that it will raise. **Valid** must be **False**

For our tutorial we will be focusing on the ```sender``` , ```amount```, ```now```, ```valid``` , ```show``` parameters.

# Unit Tests
We write all our verifying conditions in the contract and create transactions which would test all the functionalities of our contract. 
My advice here is to proceed by isolating an EntryPoint and then testing all it's variables and then moving on to the next EntryPoint:

- ```addBalanceOwner()```
```python
ob.addBalanceOwner().run(sender=udit , amount = sp.tez(25) , valid = False)
ob.addBalanceOwner().run(sender=bob , amount = sp.tez(1) , valid = False)

ob.addBalanceOwner().run(sender = bob, amount = sp.tez(25))

ob.addBalanceOwner().run(sender = bob , amount = sp.tez(25) , valid = False)
```
As we can see in the first statement we are testing our entrypoint by sending *udit*

# Conclusion

In this Tutorial we learned about coding in SmartPy, how to get testnet XTZ from the faucet, how to deploy a contract on the blockchain and how to interact with the contract using a block explorer. We also saw how our entrypoints can change a contract's storage.

# Next Steps

I would like for you to try out all the entrypoints that we have created and check what changes they make to the storage! Once you are comfortable with this basic contract you can go about creating more complex contracts, NFT tokens and much more.

# About The Author

This tutorial was written by Udit Kapoor, who is a Tezos India 2.0 Fellow and a blockchain enthusiast. Their signature project is CryptoWill and they like to dabble in Flutter as well! Reach out to Udit on [GitHub](https://github.com/Udit-Kapoor) and you can find them on Discord as user **pichkari#56**.

# References

- Calculator example code from SmartPy.io
- [OpenTezos](https://opentezos.com/)

