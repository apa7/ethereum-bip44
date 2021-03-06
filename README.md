# 概述
1. 以太坊通用钱包助记词钱包创建; <BR>遵循比特币改进建议[BIP44](https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki)规则， 可参考 [ethfans:【虚拟货币钱包】从 BIP32、BIP39、BIP44 到 Ethereum HD Ｗallet](https://ethfans.org/posts/from-BIP-to-ethereum-HD-wallet);
2. 借鉴 [ethereum-bip44-python](https://github.com/michailbrynard/ethereum-bip44-python) 进行再次封装;
3. eth_bip44.EtherCommon 实现功能;
    - 创建助记词钱包
    - 生成/解析Keystore
    - 通过私钥或KeyStore进行私链交易
    - 通过私钥或KeyStore进行私链智能合约远程部署（Deploying smart contract with keystore）
    - 通过私钥或KeyStore进行私链智能合约远程交易（Sending contract transaction with keystore）


# 测试代码
## 功能测试
- 助记词创建测试， keystore解码, 私钥导入；
- 智能合约发布及转账：
    - 1 创建钱包， 使用coinbase转入Ether；
    - 2 使用新的钱包签名智能合约并进行远程发布；
    - 3 智能合约token转账；
    - 4 智能合约余额查询；
- `BoreyToken.json` 使用truffle compile 生成， [truffle 部署智能合约](https://github.com/zhuquanbin/solidity-note/tree/master/notes/truffle/erc20-token)。

## 代码
`test.py`
```python
# -*- encoding:utf8 -*-
"""
    author: quanbin_zhu
    time  : 2018/4/18 10:52
"""

import json
import time
from web3 import Web3, HTTPProvider
from eth_utils import to_checksum_address
from eth_bip44 import EtherCommon, EtherError

# 用户钱包 keystore
keystore_json   = {"address":"9b4eabea5d69a3c434c40f84f65282f6b4d9b232","crypto":{"cipher":"aes-128-ctr","ciphertext":"0c1a562d3a28682f28a02de89927adbacd99168e9efa48fe3ff0a85df70febac","cipherparams":{"iv":"6cdadf4f3f38af7a4aee1843198a9c00"},"kdf":"scrypt","kdfparams":{"dklen":32,"n":262144,"p":1,"r":8,"salt":"20b525e4dbfac089dd9c5c65fb873a9e530d42f47610647fe31b23f6e348f58e"},"mac":"1651c2597ccedb675f372ad49f0ad30fb5a6c604ee5bda7283e35ed3f9da3ba7"},"id":"6d3f052d-bdf7-411f-bc99-86b7e73fb6a3","version":3}
# 钱包密码
password = "123456"
http_url = "http://192.168.3.193:8541"

class TestBip44(object):
    def __init__(self, ck=keystore_json, cp=password):
        # web3 实例
        self.w3 = Web3(HTTPProvider(http_url))
        self.w3.eth.enable_unaudited_features()
        self.coinbase_keystore = ck
        self.coinbase_password = cp
        self.account1 = EtherCommon.generate_eth_mnemonic()
        self.account2 = EtherCommon.generate_eth_mnemonic()
        # compile with truffle
        with open("BoreyToken.json") as fp:
            data = fp.read()
            self.contract_info = json.loads(data)

    def create_wallet_with_mnemonic(self):
        """
        TODO: 助记词创建测试， keystore解码
        """
        ec_obj = EtherCommon.generate_eth_mnemonic()
        print (u"==" * 15, u"\n账号带助记词的以太坊钱包：  助记词 => 私钥 认证网址：https://iancoleman.io/bip39/\n", u"==" * 15)
        print (u"助记词\t: ", ec_obj.mnemonic_codes)
        print (u"地址\t: ", ec_obj.address)
        print (u"公钥\t: ", ec_obj.public_key)
        print (u"私钥\t: ", ec_obj.private_key)

        print ("\n", u"==" * 15)
        print (u"生成keystore文件： 私钥 => keystore \n", u"==" * 15)
        keystore = ec_obj.keystore("password")
        print (u"KeyStore\t: ", json.dumps(keystore, indent=4))

        print ("\n", u"==" * 15)
        print (u"解码keystore文件： keystore => 私钥 \n", u"==" * 15)
        ec_obj2 = EtherCommon.decode_keystore_from_json(keystore, "password")
        print (u"地址\t: ", ec_obj2.address)
        print (u"公钥\t: ", ec_obj2.public_key)
        print (u"私钥\t: ", ec_obj2.private_key)

        print ("\n", u"==" * 15)
        print (u"解码私钥： 私钥 => 公钥 + 地址 \n", u"==" * 15)
        ec_obj3 = EtherCommon.decode_private_key_from_str(ec_obj2.private_key)
        print (u"地址\t: ", ec_obj3.address)
        print (u"公钥\t: ", ec_obj3.public_key)
        print (u"私钥\t: ", ec_obj3.private_key)

    def send_tx(self, keystore, password, to, value,):
        """
        TODO: 私链发起转账交易测试
        """
        # ether common object
        gec_obj = EtherCommon.decode_keystore_from_json(keystore, password)
        v = self.w3.toWei(value, "ether")
        tx_hash = gec_obj.web3py_transaction(self.w3, to, v)
        print("Send Ether from: %s, to: %s, value: %s" % (gec_obj.address, to, v))
        _ = self.waiting_for_transaction(tx_hash)

    def deploy_contract_and_tx(self):
        """
        TODO: 使用keystore部署智能合约并进行转账测试
        """
        # 1. 发送ether到新创建的钱包地址
        print(
            """################################################################\n########    send 3 ether from coinbase to account1     #########\n################################################################""")

        self.send_tx(self.coinbase_keystore, self.coinbase_password, self.account1.address, 3)

        # 2. 使用新的钱包地址进行发现只能合约
        print("\n\n\n")
        print(
            """################################################################\n########         deploy contract with account1         #########\n################################################################""")
        print(self.contract_info["source"])
        nonce = self.w3.eth.getTransactionCount(to_checksum_address(self.account1.address))
        tx_hash = self.account1.web3py_contract_deploy(self.w3, nonce, self.contract_info["abi"], self.contract_info["bytecode"], (10 ** 8))
        tx_receipt = self.waiting_for_transaction(tx_hash)
        contract_address = tx_receipt['contractAddress']
        print("Contract Address : %s" % contract_address)

        # 3. 智能合约中token 交易
        print("\n\n\n")
        print(
            """################################################################\n########     send token from account1 to account2      #########\n################################################################""")

        contract = self.w3.eth.contract(address=contract_address, abi=self.contract_info["abi"])
        nonce += 1
        # 连续操作一个账号时， 需手动赋值 nonce， 防止   Error: replacement transaction underpriced
        for i in range(0,5):
            v = 100 * i
            tx_hash = self.account1.web3py_contract_transaction(self.w3, contract, self.account2.address, v, nonce+i)
            print("Transaction Hash :", tx_hash)

        _ = self.waiting_for_transaction(tx_hash)

        # 4. 查询智能合约token的余额
        print("\n\n\n")
        print("[Wallet] %s token: " % self.account1.address, contract.functions.balanceOf(to_checksum_address(self.account1.address)).call())
        print("[Wallet] %s token: " % self.account2.address, contract.functions.balanceOf(to_checksum_address(self.account2.address)).call())

    def waiting_for_transaction(self, tx_hash, msg = "Waiting transaction receipt", secs = 2):
        # tx = "0x%s" % tx_hash if not tx_hash.startswith("0x") else tx_hash
        print("Get transaction receipt: %s" % tx_hash)
        tx_receipt = None
        while not tx_receipt:
            tx_receipt = self.w3.eth.getTransactionReceipt(tx_hash)
            if not tx_receipt:
                print("%s, sleep %s seconds ... " % (msg, secs))
                time.sleep(secs)
        return tx_receipt

if __name__ == "__main__":
    TestBip44().deploy_contract_and_tx()

```
## 结果
```bash
(ethereum-bip44) F:\ethereum-bip44>python test.py
################################################################
########    send 3 ether from coinbase to account1     #########
################################################################
Send Ether from: 0x9b4eabea5d69a3c434c40f84f65282f6b4d9b232, to: 0xb64cfda8ce181b2f4aa88a4fb3582ec8bd325c80, value: 3000000000000000000
Get transaction receipt: 53fe81aea28b1256c04d51df74a21ca58ae458427761d3cf4540bac6a7885e97
Waiting transaction receipt, sleep 2 seconds ...
Waiting transaction receipt, sleep 2 seconds ...
Waiting transaction receipt, sleep 2 seconds ...


################################################################
########         deploy contract with account1         #########
################################################################
pragma solidity ^0.4.23;

contract BoreyToken {
    event Transfer(address indexed _from, address indexed _to, uint256 _value);

    address  public owner;
    mapping (address => uint)  public balanceOf;

    constructor(uint256 supply) public {
        owner = msg.sender;
        balanceOf[msg.sender] = supply;
    }

    function transfer(address _to, uint256 _value) public returns (bool) {
        require(_to != address(0));
        require(_value <= balanceOf[msg.sender]);

        balanceOf[msg.sender] = balanceOf[msg.sender] - _value;
        balanceOf[_to] = balanceOf[_to] + _value;
        emit Transfer(msg.sender, _to, _value);
        return true;
    }
}
Get transaction receipt: 63b4537990a6d973a3bcfdfcc9d8dc88309aa7329c85458708c27e52744791eb
Waiting transaction receipt, sleep 2 seconds ...
Waiting transaction receipt, sleep 2 seconds ...
Contract Address : 0xe09bB0A2073113D7B789Ab9b7fAA9f7C2b9C59D2


################################################################
########     send token from account1 to account2      #########
################################################################
Transaction Hash : 69f44c59fd694f81a12a3ab76b2c8f15487c8b8df25d418e81689c10fc4c2e33
Transaction Hash : 52a1196134893cd5789843022062f5c0ce56908bf056e73c63b1744b30cee3d2
Transaction Hash : 3f2be65df95fd6ce7618c3e6650651b3dbe176e7a82eb0398b624256293bcd24
Transaction Hash : 983457dea49f947f390d5000ec564bdd477818ddbdfb0c76f4d4437c01106b84
Transaction Hash : 3642209697bae6a0e373991750e937dbe4f21e4b26ef3a9a30e9fd21ec6c9d98
Get transaction receipt: 3642209697bae6a0e373991750e937dbe4f21e4b26ef3a9a30e9fd21ec6c9d98
Waiting transaction receipt, sleep 2 seconds ...
Waiting transaction receipt, sleep 2 seconds ...
Waiting transaction receipt, sleep 2 seconds ...


[Wallet] 0xb64cfda8ce181b2f4aa88a4fb3582ec8bd325c80 token:  99999000
[Wallet] 0x4610101d066ada5b76f33cfc4f8b2567b46a0807 token:  1000

```

# Issue
1. error replacement transaction underpriced：

    因为每个账号每次发起的交易nonce是递增的， 当连续从一个账号上发起交易时，上次的交易可能没有被记录到block上; <BR>
    该Address本次交易的nonce = Address获取的nonce + Address对应的Pending状态的数目<BR>
    参考:
     [Transaction having low gasPrice makes all the other transactions hang on the pending state](https://github.com/ethereum/go-ethereum/issues/16284)


