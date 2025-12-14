# Hardhat基础：部署交互FundMe

## 一、环境搭建

## 安装node.js

nvm作为一款管理nodejs版本工具，通过命令行切换，实现在开发环境中安装使用多个nodejs版本。

1.打开终端，安装nvm

```shell
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.1/install.sh | bash
source ~/.profile
```

2.安装指定版本node.js

```shell
nvm install 22
```



### 安装VS Code插件

1.安装js插件

![image-20251214182854259](E:\web3笔记\image-20251214182854259.png)

2.安装solidity插件

![image-20251214183115776](E:\web3笔记\image-20251214183115776.png)

## 二、创建Hardhat项目

1.创建一个目录,进入目录并初始化

```shell
mkdir hardhat-tutorial
cd hardhat-tutorial
npm init
npm install --save-dev hardhat@hh2
npx hardhat init

```

2.选择创建一个空的js项目

![image-20251214183258447](E:\web3笔记\image-20251214183258447.png)





## 三、编译，部署合约

1.编译合约

```shell
 npx hardhat compile
```

2.部署合约,创建部署脚本deployFundMe.js

```javascript
//1.import ethers.js
// 2.create main function
// 3.execute deploy command

const { ethers } = require("hardhat")

async function main() {
    const fundMeFactory = await ethers.getContractFactory("FundMe01")
    console.log("contract deploying")
    const fundMe = await fundMeFactory.deploy(300)
    await fundMe.waitForDeployment()
    console.log("contract has been deployed successfully, contract address is " + fundMe.target)
    if (hre.network.config.chainId == 11155111 && process.env.ETHERSCAN_API_KEY) {
        console.log("waiting for 5 confirmations")
        await fundMe.deploymentTransaction().wait(5)
        await verifyFundMe(fundMe.target, [300])
    } else {
        console.log("verification skipped..")
    }

    //init 2 accounts
    const [firstAccount, SecondAccount] = await ethers.getSigners()
    //fund contract with firstContract
    const fundTx = await fundMe.fund({ value: ethers.parseEther("0.5") })
    await fundTx.wait()
    //check balance of contract
    balanceOfConmtract = await ethers.provider.getBalance(fundMe.target)
    console.log("Balance of the contract is " + balanceOfConmtract)

    //fund contract with secondContract
    const fundTxWithSecondAccount = await fundMe.connect(SecondAccount).fund({ value: ethers.parseEther("0.5") })
    await fundTxWithSecondAccount.wait()

    //check balance of contract
    balanceOfConmtract = await ethers.provider.getBalance(fundMe.target)
    console.log("Balance of the contract is " + balanceOfConmtract)
    //check mapping
    const firstAccountBalanceOfContract = await fundMe.fundersToAmount(firstAccount.address)
    const secondAccountBalanceOfContract = await fundMe.fundersToAmount(SecondAccount.address)
    console.log("balance of first account " + firstAccount.address + " is " + firstAccountBalanceOfContract)
    console.log("balance of second account " + SecondAccount.address + " is " + secondAccountBalanceOfContract)






}

async function verifyFundMe(fundMeAddr, args) {
    await hre.run("verify:verify", {
        address: fundMeAddr,
        constructorArguments: args,
    })

}

main().then().catch((error) => {
    console.error(error)
    process.exit(0)
})
```

3.执行脚本

```shell
npx hardhat run deployFundMe.js
```

4.部署完成通过https://sepolia.etherscan.io/查看

![image-20251214184454339](E:\web3笔记\image-20251214184454339.png)