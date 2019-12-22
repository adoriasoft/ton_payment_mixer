# Laundromat contract

### Preparation

    You have to already build the func compiler and fift interpreter with the lite client of the TON blockchain.
    You can downloaded it from this link https://github.com/ton-blockchain/ton .
    

### Description

    This is a smart contract which can help to obfuscate your money. It means that any person will not be able to understand from whom to whom your coins was sent.
    This smart contract is based on the ring signatures https://en.wikipedia.org/wiki/Ring_signature and used RSA encryption/decryption algorithms with their public keys https://en.wikipedia.org/wiki/RSA_(cryptosystem).

    
### Manual

    Compile the laundromat.fc
    ``` 
    .\func -AP -o /laundromat/generated/laundromat_code.fif  /laundromat/src/stdlib.fc /laundromat/src/RSA.fc /laundromat/src/ring_sign.fc /laundromat/src/laundromat.fc
    ```
    
    Generated code will be placed in the generated folder.
    
    Next you have to find at least 2 participants of this smart contract with their wallet addresses from which their will be send the valid signature.
    (For these purposes you can you use the standard wallet.fif )
    
    After that all these participants have to generate their own public key pair of the RSA signature ('e' and 'n').
    In the tests we will take such values of the public keys e1 = 1275, e2 = 7887, e3 = 10811, n1 = 1357, n2 = 8083, n3 = 11021, and d1, d2 ,d3 which are the inverse elements of the e_i by the mod.
    (You look into the 'laundromat/src/generate_sign.fc' and you the code from there to generate your own values)
    
    After you will have all these values the public keys off all participants and their addresses you can set up the 'laundromat/deploy/deploy.fif' script.
    
    Here put the value of your addresses
    ```
    103835235685005910186741582083202827840114002218372192284974940710643891936425 constant addr1
    103835235685005910186741582083202827840114002218372192284974940710643891936424 constant addr2
    103835235685005910186741582083202827840114002218372192284974940710643891936423 constant addr3
    
    <b addr1 s, 0 8 u, b> <s constant addr1
    <b addr2 s, 0 8 u, b> <s constant addr2
    <b addr3 s, 0 8 u, b> <s constant addr3

    addr1 0 dictnew 32 idict! . 
    addr2 swap 1 swap 32 idict! .
    addr3 swap 2 swap 32 idict! . constant approve_addresses
    ```
    Here put the value of your public keys in the correspond order for the addresses
    ```
    1275 constant e1
    7887 constant e2
    10811 constant e3

    1357 constant n1
    8083 constant n2
    11021 constant n3
    
    <b e1 256 u, n1 256 u, b> <s constant pubkey1
    <b e2 256 u, n2 256 u, b> <s constant pubkey2
    <b e3 256 u, n3 256 u, b> <s constant pubkey3
    
    pubkey1 0 dictnew 32 idict! . 
    pubkey2 swap 1 swap 32 idict! .
    pubkey3 swap 2 swap 32 idict! . constant pubkey_array
    
    ```
    
    And the last step is too fill the number of participants and payout value
    ```
    3 constant number_of_participants
    10000000000 constant payment_amount
    ```
    
    After that you have to deploy it and every participant have to send for 11 Grams to this smart contract.
    (1 Gram more it is a fee for the successful work of the smart contract. It can be changeable it only needs for the gas purposes.)
    
    The next step is to generate the valid signatures and send it to the smart contract as a internal message.
    For these you can also use the 'laundromat/src/generate_sign.fc' code for the signature generation.
    For must to have your private key ('d' value from the RSA) number array of public keys (you can use the 'get_public_keys_array()' function from the laundromat smart contract).
    !!!!! The order of the public keys should be the same that their are was in the deploy.fif
    ```
    pubkey1 0 dictnew 32 idict! . 
    pubkey2 swap 1 swap 32 idict! .
    pubkey3 swap 2 swap 32 idict! . constant pubkey_array
    ```
    The parameters of the signature generation are the 'message' random 'v' and the random 'x' values the number of them should be '<number of public keys> - 1'
    As a result of the signature generation you have to obtain the last 'x' value which will satisfies the ring sign equation.
    The 'sign_message()' function will return to you an array with all 'x' values which are corresponds to the public keys array and you have to send it in the same order.
    
    After you will have your signature you can send it to the smart contact to get your funds.
    You can set up the 'laundromat/deploy/send_sign.fif' script.
    First put the payout addresses
    ```
    103835235685005910186741582083202827840114002218372192284974940710643891936422 constant payout_addr
    ```
    Then put your set of the x values and 'v', 'message' values
    ```
    1355 constant x_1
    485 constant x_2
    11019 constant x_3

    <b x_1 256 u, b> <s constant x_1
    <b x_2 256 u, b> <s constant x_2
    <b x_3 256 u, b> <s constant x_3
    ```
    
    ```
    354 constant v
    547 constant message
    ```
    Query body should not be directly sent to the network. Use wallet.fif script with -B <query-body> parameter to embed query body in the wallet contract query.
    
    After all this steps you will get your funds.
    
### Testing
    Also you cant test the work of the smart contract, they are in the laundromat/tests folder
    
    You can just run them
    ```
        .\fift -I /laundromat/lib /laundromat/tests/test1.fif
    ```
    
    The expected result of the test1. It is a positive test with the correct values)
    ```
    0 3 10000000000 1 1 1
    ```
    The first '0' it is from the runvmctx command
    '3' - the number of participants
    '10000000000' - payout amount
    '1' - 'init' flag it is only for recv_external() function
    '1' - 'money_available' flag, it means that the smart contract has enough money ( > 'number_of_participants * payout_amount') and ready to send funds
    '1' - 'is_successful' flag, it means that the signature is correct and funds has successfully been sent. (Only for test purposes)
    
    The expected result of the test2. It is a negative test where from the one address participant tries to get one more funds.
    ```
    0 3 10000000000 1 1 0
    ```
    
    The expected result of the test3. It is a negative test with the incorrect signature values (was changed an x_1 value).
    ```
    0 3 10000000000 1 1 0 
    ```
    
    The expected result of the test4. It is a negative test where the smart contract have not enough funds and not be ready to send money.
    ```
    0 3 10000000000 1 0 0 
    ```
    
    
    
    
    
    
    