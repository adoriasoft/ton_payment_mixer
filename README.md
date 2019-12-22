# Ring signature based mixer contract

### Preparation

  You have to already build the func compiler, fift interpreter and the lite client of the TON blockchain.
  You can download it from [https://github.com/ton-blockchain/ton].
    

### Description

  This is a smart contract which can help to obfuscate your Grams. It means that noone should be able to link the recipient and the sender of the coins.
  This smart contract is based on the ring signatures https://en.wikipedia.org/wiki/Ring_signature and uses RSA encryption/decryption algorithms with their public keys https://en.wikipedia.org/wiki/RSA_(cryptosystem).

    
### Manual

1. Compile the app

    ```bash 
    $ ./func -AP -o generated/laundromat_code.fif src/stdlib.fc src/RSA.fc src/ring_sign.fc src/laundromat.fc
    ```
    
    Generated code will be placed in the 'generated' folder.

2. Find participants

    You have to find at least 2 participants of this smart contract with their wallet addresses from which their will be able to create a ring signature.
    (For this purpose you can you use the standard wallet.fif)
    
    Each participant should generate his public key pair of the RSA signature ('e' and 'n').
    In the tests we will take hardcoded public keys:

    ```
    e1 = 1275, e2 = 7887, e3 = 10811, n1 = 1357, n2 = 8083, n3 = 11021
    ```

    and

    ```
    d1, d2, d3 which are the inverse elements of the e_i by the mod.
    ```

3. Generate real public keys

    Use 'src/generate_sign.fc' to generate new public keys.

4. Configure contract
    
    After collecting all of the public keys and user addresses you can configure the deploy script 'deploy/deploy.fif'
    
    Put user addresses here:
    ```
    103835235685005910186741582083202827840114002218372192284974940710643891936425 constant addr1
    103835235685005910186741582083202827840114002218372192284974940710643891936424 constant addr2
    103835235685005910186741582083202827840114002218372192284974940710643891936423 constant addr3
    ```

    Put user public keys here:
    
    ```
    1275 constant e1
    7887 constant e2
    10811 constant e3

    1357 constant n1
    8083 constant n2
    11021 constant n3
    ```
    
    Fill the number of participants and payout value:
    ```
    3 constant number_of_participants
    10000000000 constant payment_amount
    ```

5. Compile deploy script

    ```bash 
    $ ./fift -s deploy/deploy.fif wallets/contract-wallet
    ```

    Save output somewhere, we will use it later. Take a look on the next fields:

    - "Non-bounceable address (for init)" - Will use it as a {contract id}.
    - "Bounceable address (for later access)" - Will use it as a {contract id inited}.

6. Deploy contract

    Send the deployment query (deploy.boc) using the lite-client:

      Get last block
      ```bash
      lite-client> last 
      ```
    
      Publish contract
      ```bash
      lite-client> sendfile deploy.boc
      ```

      *Deposit some test grams in to the contract address.*

      - Use TestTonBot. (Telegram)
      - Send {contract id}.
      - Chose any amount.
      
      Update state
      ```bash
      lite-client> last
      ```
    
      We can see new information about contract if it is deployed.
      ```bash
      lite-client> getaccount {contract id}
      ```

7. Collect Grams from the users
    
    Every participant should send 11 Grams to the mixer contract.
    
    **Important note: _1 Gram is a fee for the contract processing. This value may vary and is being used for the gas purposes._**

8. Prepare signatures
    
    The next step is to generate valid signatures and send it to the contract as an internal message.
    You should use 'src/generate_sign.fc' code for the signature generation.
    You should insert your private key ('d' value from the RSA) and an array of public keys (you can use the 'get_public_keys_array()' function from the 'laundromat' contract).

    **Important note: _The order of the public keys should be the same as in the deploy.fif_**
    
    ```
    pubkey1 0 dictnew 32 idict! . 
    pubkey2 swap 1 swap 32 idict! .
    pubkey3 swap 2 swap 32 idict! . constant pubkey_array
    ```

    The parameters of the signature generation are the 'message' random 'v' and the random 'x' values. You should have them as many as '\<number of public keys\> - 1'
    As a result of the signature generation you have to obtain the last 'x' value which satisfies the ring signature equation.
    The 'sign_message()' function will return to you an array with all 'x' values which correspond to the public keys array and you have to send it in the same order.
    
    As soon as you have your signature you can send it to the mixer contact to get your funds.

9. Configure send script

    You can configure the 'deploy/send_sign.fif' script now.
   
    Insert the payout addresses
    ```
    103835235685005910186741582083202827840114002218372192284974940710643891936422 constant payout_addr
    ```

    Then insert your set of the x values and 'v', 'message' values
    ```
    1355 constant x_1
    485 constant x_2
    11019 constant x_3
    ```
    
    ```
    354 constant v
    547 constant message
    ```
    **Important note: _Query body should not be directly sent to the network. Use wallet.fif script with -B <query-body> parameter to embed query body in the wallet contract query._**
    
10. Wait for your funds

    Contract should automatically send your funds now.

    
### Testing
    
  Offline test scripts are in the 'tests' folder
    
  You can simply run them
  ```bash
  $ ./fift -I lib tests/test1.fif
  ```
  
1. Positive test with correct values

  The expected result of the 'test1.fif'
  ```
  0 3bash 10000000000 1 1 1
  ```

  The first '0' is result of the runvmctx command
  '3' - the number of participants
  '10000000000' - payout amount
  '1' - 'init' flag is only for recv_external() function
  '1' - 'money_available' flag; it means that the smart contract has enough money ( > 'number_of_participants * payout_amount') and ready to send funds
  '1' - 'is_successful' flag; it means that the signature is correct and funds has successfully been sent. (Only for test purposes)
    
2. Negative test where one of the participants is trying to get more funds

  The expected result of the 'test2.fif'
  ```bash
  0 3 10000000000 1 1 0
  ```
 
3. Negative test with incorrect signature values (x_1 value was changed)
  
  The expected result of the 'test3.fif'
  ```bash
  0 3 10000000000 1 1 0 
  ```
  
4. Negative test where the contract does not have enough funds and not ready to send money

  The expected result of the 'test4.fif'
  ```
  0 3 10000000000 1 0 0 
  ```

### Deployment

Test contract is deployed at address kf-93zlLn8V8_oAG91k3cWSEOD-GpXMxI_LwSFtBNPT5RXOu in the TON testnet2.

### Future improvements

1. Better public keys handling

Users should not directly change hardcoded script values.

2. Better ring signatures with some fraction of the private key added

This will allow to not require users to collect output address prior to creating a mixer contract.