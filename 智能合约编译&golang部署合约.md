# MacOS编译智能合约并使用Golang部署合约

1.安装 [Solidity编译器](https://solidity.readthedocs.io/en/latest/installing-solidity.html) (`solc`).

```shell
brew tap ethereum/ethereum
brew install solidity
```



2.安装`abigen`工具

```shell
go get -u github.com/ethereum/go-ethereum
cd $GOPATH/src/github.com/ethereum/go-ethereum/
make
make devtools
```



3.编写简单的智能合约程序 `Store.sol`

```solidity
pragma solidity ^0.8.15;

contract Store {
  event ItemSet(bytes32 key, bytes32 value);

  mapping (bytes32 => bytes32) public items;

  function setItem(bytes32 key, bytes32 value) external {
    items[key] = value;
    emit ItemSet(key, value);
  }
}
```



4. 进入程序所在目录，执行以下命令：

   ```shell
   solc --abi Store.sol >> Store.abi
   ```

   

5.编辑Store.abi文件，删除json数据之外的内容

6.为了从Go部署智能合约，我们还需要将solidity智能合约编译为EVM字节码。 EVM字节码将在事务的数据字段中发送。 在Go文件上生成部署方法需要bin文件。编辑以下命令生成的Store.bin文件，只保留Binary冒号后的内容。

```shell
solc --bin Store.sol >> Store.bin
#合约内容太大，可以使用优化参数，否则会部署失败
#solc --bin --optimize Store.sol >> Store.bin
```



7.现在我们编译Go合约文件，其中包括deploy方法，因为我们包含了bin文件。

```shell
abigen --bin=Store.bin --abi=Store.abi --pkg=store --out=Store.go
```

```go
func DeployStore() error {
	client, err := ethclient.Dial("https://data-seed-prebsc-1-s1.binance.org:8545")
	if err != nil {
		return err
	}

	privateKey, err := crypto.HexToECDSA("289385cad1cbe8393b66fbac9a4bedd5ab26d5a34b766ebedb97c27b57b3f76a")
	if err != nil {
		return err
	}

	publicKey := privateKey.Public()
	publicKeyECDSA, ok := publicKey.(*ecdsa.PublicKey)
	if !ok {
		return errors.New("cannot assert type: publicKey is not of type *ecdsa.PublicKey")
	}

	fromAddress := crypto.PubkeyToAddress(*publicKeyECDSA)
	nonce, err := client.PendingNonceAt(context.Background(), fromAddress)
	if err != nil {
		return err
	}

	gasPrice, err := client.SuggestGasPrice(context.Background())
	if err != nil {
		return err
	}

	auth := bind.NewKeyedTransactor(privateKey)
	auth.Nonce = big.NewInt(int64(nonce))
	auth.Value = big.NewInt(0)       // in wei
	auth.GasLimit = uint64(300000) // in units
	auth.GasPrice = gasPrice

	address, tx, instance, err := store.DeployStore(auth, client)
	if err != nil {
		return err
	}

	logx.Info(address.Hex())   // 0x35D5774CFD3d122000173F9B2cf82a4bE911Ee38
	logx.Info(tx.Hash().Hex()) // 0x071afb661b19df0ec731f21a73526c8a23b9f595bc3ba717e30ff34e851e3648

	_ = instance

	return nil
}
```

