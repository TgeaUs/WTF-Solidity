---
title: 58. ERC1820 合约接口注册
tags:
  - solidity
  - ERC165
  - ERC1820
---

# WTF Solidity极简入门: 58. ERC1820 合约接口注册

我最近在重新学 Solidity，巩固一下细节，也写一个“WTF Solidity极简入门”，供小白们使用（编程大佬可以另找教程），每周更新 1-3 讲。

推特：[@0xAA_Science](https://twitter.com/0xAA_Science)｜[@WTFAcademy_](https://twitter.com/WTFAcademy_)

社区：[Discord](https://discord.gg/5akcruXrsk)｜[微信群](https://docs.google.com/forms/d/e/1FAIpQLSe4KGT8Sh6sJ7hedQRuIYirOoZK_85miz3dw7vA1-YjodgJ-A/viewform?usp=sf_link)｜[官网 wtf.academy](https://wtf.academy)

所有代码和教程开源在 github: [github.com/AmazingAng/WTFSolidity](https://github.com/AmazingAng/WTFSolidity)

---

ERC1820主要用作于接口注册。可以做到A合约使用B合约的时候，A合约可以通过ERC1820判断B合约是否实现了某一个接口，从而避免出错。同时，ERC1820向下兼容ERC165。

# IERC1820

IERC1820是ERC1820的接口合约，但实际上[官方](https://eips.ethereum.org/EIPS/eip-1820)给出的例子没有写接口，而是直接写了合约。一切尽在注释中。。。

```solidity
interface IERC1820Registry {
    // 用于设置_implementer实现的接口_interfaceHash，普遍的用法是_addr = _implementer
    // 如果_addr设置了管理者，那么msg.sender = _addr管理者 是必要条件
    function setInterfaceImplementer(address _addr, bytes32 _interfaceHash, address _implementer) external;

    // 获取_addr的管理者
    function getManager(address _addr) external view returns (address);

    // 把地址_addr的管理者设置为_newManager
    function setManager(address _addr, address _newManager) external;

    // _addr是否实现了_interfaceHash接口，返回真正的实现者
    function getInterfaceImplementer(address _addr, bytes32 _interfaceHash) external view returns (address);
    
    // 工具函数，为了更好的方便生成_interfaceHash
    function interfaceHash(string memory _interfaceName) external pure returns (bytes32);

    // 兼容ERC165合约，调用后缓存
    function updateERC165Cache(address _contract, bytes4 _interfaceId) external;

    // 兼容ERC165合约，调用后不缓存
    function implementsERC165InterfaceNoCache(address _contract, bytes4 _interfaceId) external view returns (bool);

    // 对缓存的结果进行查询
    function implementsERC165Interface(address _contract, bytes4 _interfaceId) external view returns (bool);

    // 当一个地址（addr）被设置为某个接口（interfaceHash）的实现者（implementer）触发事件
    event InterfaceImplementerSet(address indexed addr, bytes32 indexed interfaceHash, address indexed implementer);

    // 管理者变更时触发事件
    event ManagerChanged(address indexed addr, address indexed newManager);
}
```

# ERC1820

接下来我们具体看看每一个接口是如何实现的。

### setInterfaceImplementer 函数

参数_addr似乎和_implementer有同质化的问题，好像看起来_addr是多余的。实际上不是这样，_addr 作用是声明接口支持的地址，而_implementer 是实际提供接口实现逻辑的地址。但在通常的用法来说，_addr 应该与 _implementer 相等。

```solidity
function setInterfaceImplementer(address _addr, bytes32 _interfaceHash, address _implementer) external {
    address addr = _addr == address(0) ? msg.sender : _addr;

    // 获得addr的管理者
    require(getManager(addr) == msg.sender, "Not the manager");

    // ERC165合约不可以与ERC1820混着使用，ERC165有额外的接口
    require(!isERC165Interface(_interfaceHash), "Must not be an ERC165 hash");
    
    // 安全性防护，请看安全性章节
    if (_implementer != address(0) && _implementer != msg.sender) {
        require(
            ERC1820ImplementerInterface(_implementer)
                .canImplementInterfaceForAddress(_interfaceHash, addr) == ERC1820_ACCEPT_MAGIC,
            "Does not implement the interface"
        );
    }
    
    // 记录_interfaceHash对应的实现者
    interfaces[addr][_interfaceHash] = _implementer;
    emit InterfaceImplementerSet(addr, _interfaceHash, _implementer);
}

// ERC165只有4个字节后面全是0，所以0与任何数相&都是0，很好的检测ERC165的方法
function isERC165Interface(bytes32 _interfaceHash) internal pure returns (bool) {
    return _interfaceHash & 0x00000000FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF == 0;
}
```

### getManager 和 setManager 函数

`getManager` 函数用于获得_addr的管理者，`setManager` 函数用于设置_addr的管理者。

```solidity
function setManager(address _addr, address _newManager) external {
    require(getManager(_addr) == msg.sender, "Not the manager");
    managers[_addr] = _newManager == _addr ? address(0) : _newManager; 
    emit ManagerChanged(_addr, _newManager);
}

function getManager(address _addr) public view returns(address) {
    if (managers[_addr] == address(0)) {
        return _addr;
    } else {
        return managers[_addr];
    }
}
```

### getInterfaceImplementer 函数

对于地址_addr，返回真正实现的实现_interfaceHash接口的_implementer地址。通常来说_addr = _implementer，具体要看`setInterfaceImplementer`函数如何设置的。

```solidity
function getInterfaceImplementer(address _addr, bytes32 _interfaceHash) external view returns (address) {
    address addr = _addr == address(0) ? msg.sender : _addr;
    if (isERC165Interface(_interfaceHash)) {
        bytes4 erc165InterfaceHash = bytes4(_interfaceHash);
        return implementsERC165Interface(addr, erc165InterfaceHash) ? addr : address(0);
    }
    return interfaces[addr][_interfaceHash];
}
```

### interfaceHash 函数

ERC1820贴心的为我们定义了生成一个接口哈希值的函数，以便更好的兼容ERC1820。可以看到就是abi.encodePacked后再来一次keccak256算法。关于abi的使用方法请前往[27课 ABI编码](https://github.com/AmazingAng/WTF-Solidity/blob/main/27_ABIEncode/readme.md)。

注意哈，这里填写的是函数名字，不需要写参数！
```solidity
function interfaceHash(string calldata _interfaceName) external pure returns(bytes32) {
    return keccak256(abi.encodePacked(_interfaceName));
}
```
### updateERC165Cache 和 implementsERC165Interface函数

在ERC165标准中，我们通常会使用该合约的`supportsInterface`函数来判断一个合约是否实现了某个接口。ERC1820也向下兼容ERC165标准。

这两个函数是混着来使用的，使用了一次updateERC165Cache后合约地址_contract会进行缓存，后续就可以使用implementsERC165Interface函数来查询了。

```solidity
// _interfaceId填写0x01ffc9a7
function updateERC165Cache(address _contract, bytes4 _interfaceId) external {
    // 缓存_contract地址的结果
    interfaces[_contract][_interfaceId] = implementsERC165InterfaceNoCache(
        _contract, _interfaceId) ? _contract : address(0);

    // 证明_contract已经被缓存了
    erc165Cached[_contract][_interfaceId] = true;
}

function implementsERC165Interface(address _contract, bytes4 _interfaceId) public view returns (bool) {
    // 没被缓存
    if (!erc165Cached[_contract][_interfaceId]) {
        return implementsERC165InterfaceNoCache(_contract, _interfaceId);
    }
    // 被缓存了看一下是否有这个合约
    return interfaces[_contract][_interfaceId] == _contract;
}
```

### implementsERC165InterfaceNoCache函数

这个函数是最核心的函数，实现也是最复杂的。

核心函数是`noThrowCall`， 汇编的使用方法在注释中已经写了，重要的是返回值。在Solidity中，staticcall调用成功会返回true，否则是false。所以success是对staticcall本身调用进行判断，result是对staticcall调用返回的结果进行判断。

```solidity
function implementsERC165InterfaceNoCache(address _contract, bytes4 _interfaceId) public view returns (bool) {
    uint256 success;
    uint256 result;

    (success, result) = noThrowCall(_contract, ERC165ID);
    if (success == 0 || result == 0) {
        return false;
    }

    (success, result) = noThrowCall(_contract, INVALID_ID);
    if (success == 0 || result != 0) {
        return false;
    }

    (success, result) = noThrowCall(_contract, _interfaceId);
    if (success == 1 && result == 1) {
        return true;
    }
    return false;
}

function noThrowCall(address _contract, bytes4 _interfaceId)
    internal view returns (uint256 success, uint256 result)
{
    assembly {
        let x := mload(0x40)               // 获取0x40位置的内存（32字节）
        mstore(x, 0x01ffc9a7)              // 将0x01ffc9a7放入0x40-0x44位置
        mstore(add(x, 0x04), _interfaceId) // 将_interfaceId放入0x44-0x48位置

        success := staticcall(
            30000,                         // 查询ERC165标准30k gas就足够了
            _contract,                     // 调用该合约
            x,                             // 选择器+参数位置
            0x24,                          // 0x24 = 36位，前4字节是选择器，后32位是参数
            x,                             // 返回值
            0x20                           // 返回值大小
        )
        result := mload(x)                 // Load the result
    }
}
```

### ERC1820的使用

ERC1820合约 0x1820a4B7618BdE71Dce8cdc73aAB6C95905faD24
BAYC合约 0xbc4ca0eda7647a8ab7c2061c2e118a18a936f13d

1. 打开etherscan，输入ERC1820合约地址，进入合约的只读界面。
2. 点开`implementsERC165InterfaceNoCache`函数，第一个参数填BAYC地址，第二个参数填0x01ffc9a7，结果返回true


### ERC1820的安全性

`setInterfaceImplementer`函数中有这样一段代码。在ERC1820标准中，是允许EOA账户和合约账户同时使用的，而ERC165没有这样的功能。这样会不会造成ERC1820滥用呢？比如EOA账户随随便便的就把别的合约放到ERC1820里面。

1. 假设第一个参数是EOA，第三个参数是合约地址。如果合约地址没有实现`canImplementInterfaceForAddress`则会被revert。

2. 假设第一个参数是EOA，第三个参数是合约地址，并且实现了`canImplementInterfaceForAddress`。但是由于需要传递两个参数，所以我们很容易在`canImplementInterfaceForAddress`回调中判断是不是合法调用。

```solidity
if (_implementer != address(0) && _implementer != msg.sender) {
    require(
        ERC1820ImplementerInterface(_implementer)
            .canImplementInterfaceForAddress(_interfaceHash, addr) == ERC1820_ACCEPT_MAGIC,
        "Does not implement the interface"
    );
}

```

`setManager` 函数是否可以把自己设置成任意合约的管理者？这个其实也没有关系，因为有这样一行代码。由此可见首次设置_addr的管理者必须是本身才行，要么是合约，要么是EOA。

```solidity
require(getManager(_addr) == msg.sender, "Not the manager");
```

### 总结

在本节中，我们探讨了一种高度灵活的接口查询ERC1820标准，它扩展了接口检测的可能性，不仅限于合约，还包括外部拥有账户（EOA）。ERC1820还确保了与已有的ERC165标准的兼容性。

在下一章节中，ERC777 可交互代币标准利用ERC1820的接口查询功能，为代币合约和其它合约之间的交互提供了新的可能性。

### 参考

代码实现 [EIP1820](https://eips.ethereum.org/EIPS/eip-1820)