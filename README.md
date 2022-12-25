# Axellar-cross-chain-bounty

Axellar Scan https://testnet.axelarscan.io/gmp/0x0f737af42caf0cb23934b694d8ba54d7fff0f15f2d95cf2026dc03ad8a84b3c6:12

-   DistributionExecutable Contract Address on Polygon Testnet : [0xa8272B63e34AAcB71855ADE2970f1620ce896353](https://mumbai.polygonscan.com/address/0xa8272B63e34AAcB71855ADE2970f1620ce896353)
-   DistributionExecutable Contract Address on Avalanche Testnet : [0xF9d36DdC07Cf0F21b7ff35d5e4f5C40dB5b30Cea](https://testnet.snowtrace.io/address/0xF9d36DdC07Cf0F21b7ff35d5e4f5C40dB5b30Cea)
-   Axelar Transaction Detail on Testnet : [0x0f737af42caf0cb23934b694d8ba54d7fff0f15f2d95cf2026dc03ad8a84b3c6:12](https://testnet.axelarscan.io/gmp/0x0f737af42caf0cb23934b694d8ba54d7fff0f15f2d95cf2026dc03ad8a84b3c6:12)


# Allow sending tokens with a custom message.
## Followed Steps

1.  Cloned the [axelar-local-gmp-examples](https://github.com/axelarnetwork/axelar-local-gmp-examples) repo.

    Executed all the commands mentioned in the repo to set up.

    -   `git clone https://github.com/axelarnetwork/axelar-local-gmp-examples.git` to clone the repository.
    -   `npm ci` to install all dependency
    -   `cp .env.example .env` and add private-key details in `.env` file.

2.  In our file, created a struct called TransactionInfo to contain payment information such as the sender's address, the address of the token sent, the amount, and finally the message.

    -   Firstly Added `TransactionInfo` struct to `/examples/call-contract-with-token/DistributionExecutable.sol`.

    ```solidity
    struct TransactionInfo {
        address sender;
        address tokenAddress;
        uint256 amount;
        string message;
    }

    ```

    -   Then we Added two mappings recipientsToTransactions and recipientsTransactionCounter to store recipient's transaction info and count of recipient's transaction respectively.

    ```solidity
    mapping(address => TransactionInfo[]) public recipientsToTransactions;
    mapping(address => uint) public recipientsTransactionCounter;
    ```

    -  After that I modified `sendToMany` function.
    
    1, to add a function parameter for message and 
    
    2, to accomodate message and sender address into payload to ensure it is included in the transaction.

    ```solidity
    function sendToMany(
        string memory destinationChain,
        string memory destinationAddress,
        address[] calldata destinationAddresses,
        string memory symbol,
        uint256 amount,
        string memory message //  * Added paramter for message
    ) external payable {
        address tokenAddress = gateway.tokenAddresses(symbol);
        IERC20(tokenAddress).transferFrom(msg.sender, address(this), amount);
        IERC20(tokenAddress).approve(address(gateway), amount);
        bytes memory payload = abi.encode(destinationAddresses, message, msg.sender); //  <-- Included message and sender details in
        if (msg.value > 0) {
            gasReceiver.payNativeGasForContractCallWithToken{ value: msg.value }(
                address(this),
                destinationChain,
                destinationAddress,
                payload,
                symbol,
                amount,
                msg.sender
            );
        }
        gateway.callContractWithToken(destinationChain, destinationAddress, payload, symbol, amount);
    }

    ```

    -   Finally I modified the `_executeWithToken` function. 
    
    1. To correct type detail in `abi.decode` function call to extract message and sender address
    
    2. Create and store TransactionInfo on chain.

    ```solidity
    function _executeWithToken(
        string calldata,
        string calldata,
        bytes calldata payload,
        string calldata tokenSymbol,
        uint256 amount
    ) internal override {
        (address[] memory recipients, string memory message, address sender) = abi.decode(payload, (address[], string, address)); // * updated decode types to store message and sender
        address tokenAddress = gateway.tokenAddresses(tokenSymbol);
        uint256 sentAmount = amount / recipients.length;
        for (uint256 i = 0; i < recipients.length; i++) {
            IERC20(tokenAddress).transfer(recipients[i], sentAmount);
            TransactionInfo memory txnInfo = TransactionInfo(sender, tokenAddress, sentAmount, message); 
            // * Creating transactionInfo struct 
            recipientsToTransactions[recipients[i]].push(txnInfo); 
            //   * Adding TransactionInfo for that recipient in mapping .
            recipientsTransactionCounter[recipients[i]]++; //   
            * Increment recipient counter
        }
    }

    ```

3.  Code change in `//examples/index.js` were made to add below features

    -   Added **`message`** parameter to get message passed as argument from command line.
    -   Change to support **`Multiple Recipients`** was also added.
    -   Update **`sendToMany`** function call to include message paramater.
    -   Few Cosmatic change we also done to increase readability of command output.

4.  Testing was done on local env by executing the following commands taken from the gmp repo.

    -  `node scripts/createLocal.js` to start local node and kept that command running on terminal.
    ![Screenshot_20221219_105040](https://user-images.githubusercontent.com/95926324/209458469-757212c3-970f-4612-8727-9c82a1a6c11b.png)

    -  `node scripts/deploy.js examples/call-contract-with-token local` to deploy **DistributionExecutable** contract on local node.
    
    ![Screenshot_20221219_105131](https://user-images.githubusercontent.com/95926324/209458480-8bfd623e-df0d-47cc-af87-990698f94d1b.png)

    - `node scripts/test examples/call-contract-with-token local "Polygon" "Avalanche" 20 0xCF8D2Da12A032b3f3EaDC686AB18551D8fD6c132,0x0439427C42a099E7E362D86e2Bbe1eA27300f6Cb "local testing"` to transfer 20 aUSDC to `two different accounts` from `Polygon` to `Avalanche`. Amount will be divided equally among recipients.

    ![Screenshot_20221219_105534](https://user-images.githubusercontent.com/95926324/209458660-329a9f3f-ab41-40f0-a255-b978b3fe842a.png)



