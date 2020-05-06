# Eth1x64 Variant 1: Cross-shard token examples

**NOTE: These are insecure contract examples. Even if one would manage to compile them they should not be used.**

## Prologue

Assume [Eth1x64 Variant 1](./variant1.md) and that the same token contract is deployed on each shard, at the same address. It is possible to deploy at the same address by using `CREATE2` or by using `CREATE` with the same creator account and nonce.

These token instances having different addresses would require a mapping to be present (shardId → addressOnShard) in each instance.

## Solidity changes

We extend Solidity to support Eth1x64 primitives:
1. `xmsg(uint16 shardIid, address recipient, uint256 value, bytes payload) returns (uint index)` corresponding to [`XMSG`](./variant1.md#opcode-xmsg)
2. `xclaim(bytes proof)` corresponding to [`XCLAIM`](./variant1.md#opcode-xclaim)
3. `block.shardid` corresponding to [`SHARDID`](./variant1.md#opcode-shardid) (using `block` to be aligned with [`block.chainid`](https://github.com/ethereum/solidity/issues/8854))
4. `receipt.shardid` corresponding to [`XSHARDID`](./variant1.md#opcode-xshardid)
5. `receipt.sender` corresponding to [`XSENDER`](./variant1.md#opcode-xsender)

`msg.sender` and `msg.origin` retain their current meaning (the `CALLER` and `ORIGIN` opcodes, respectively).

## Fixed-supply token

In this example if a user is transferring tokens to another shard, they have to do two transactions:
1) On the sending shard they are calling `createCrosstransfer()` with the shard id, the token holder on the other side and the amount.
2) On the receiving shard they are submitting the receipt. The processing of the receipt will trigger the execution of the contract at address `MessageReceipt.to` and will be passed `MessageReceipt.payload`. This means in our example `applyCrosstransfer` will be executed (Solidity does ABI decoding automatically).

```solidity
contract XToken {
  /// The total supply across all shards.
  uint256 public globalSupply = 100000;

  // The total supply on this shard.
  uint256 public localSupply = 0;

  mapping(address => uint256) public balances;

  constructor() public {
    // Allocate some balance to some known addresses.
    balances[0x157066124f9A75d2F890591279c3a848b3f7e5e1] = 1000;
    balances[0xDab3561678ac24BBDE21797d79D77c76026f9b9E] = 500;
  }

  // Local balance transfer.
  function transfer(address _to, uint256 _value) external returns (bool success) {
    require(balances[msg.sender] >= _value);
    balances[msg.sender] -= _value;
    balances[to] += _value;

    // Note: localSupply is unchanged!
  }

  // Initiate transfer of a balance to a different shard.
  function createCrosstransfer(uint16 _shard, address _to, uint256 _value) external returns (uint) {
    require(_shard != block.shardid);

    require(balances[msg.sender] >= _value);

    // Drop from this shard.
    balances[msg.sender] -= _value;
    localSupply -= _value;

    // Initiate transfer. This creates an ABI encoded blob.
    bytes data = abi.encodeWithSignature("applyCrosstransfer(address,uint256)", _to, _value);

    // The recipient is the same token contract on the other shard, hence `this`. Value is 0 because we are not moving any Ether.
    uint index = xmsg(_shard, this, 0, data);

    // Return the MessageReceipt.index to the user so it is easier for them to find the receipt for claiming.
    return index;
  }

  // Process an incoming transfer.
  function applyCrosstransfer(address _to, uint256 _value) external {
    require(receipt.shardid != block.shardid);

    // We assume the token contract has the same address on the originating shard.
    // Also assume msg.sender is set to the originator of the transfer (this token on the foreign shard).
    require(msg.sender == this);

    balances[_to] += _value;
    localSupply += _value;
  }
}
```

It would be possible to queue and batch outgoing transfers.

## Variable-supply token

Maintaining an accurate view of `globalSupply` across multiple shards can only be done via locks in this proposal. We are not yet providing an example implementation here.

## Wrapped token

*aka Cross-shard Dai (csDai) as our inspiration*

In this example we are enabling existing ERC-20 tokens to be cross-shard capable. On the "controlling shard" we are wrapping an existing token, similarly to how [Wrapped ETH (WETH)](https://weth.io/) encapsulates Ether in an ERC-20 token. It is possible to deposit/withdraw the backing token on this shard only. It is also possible to transfer balances across shards. However in order to redeem balances for the backing token, one must transfer back to the controlling shard.

```solidity
interface ERC20 {
    function transferFrom(address, address, uint) external returns (bool);
    function approve(address, uint) external returns (bool);
    function balanceOf(address) external view returns (uint);
}

contract WrappedXToken {
  // The underlying asset.
  ERC20 public backingToken;

  // The controlling shard on which deposits are held.
  uint16 public controllingShard;

  // The total supply on this shard.
  uint256 public localSupply = 0;

  // Local balances.
  mapping(address => uint256) public balances;

  constructor(ERC20 _backingToken, uint16 _controllingShard) public {
    backingToken = _backingToken;
    controllingShard = _controllingShard;
  }

  // This is easy to support on the controlling shard.
  // Harder elsewhere 😁
  function totalSupply() external returns (uint) {
    require(this.shard == controllingShard);
    return backingToken.balanceOf(address(this));
  }

  // Prior to deposit the caller has to approve this contract on the backingToken.
  function deposit(uint value) external {
    require(block.shardid == controllingShard);
    backingToken.transferFrom(msg.sender, address(this), value);
    balances[msg.sender] += value;
    localSupply += value;
  }

  function withdraw(uint value) external {
    require(block.shardid == controllingShard);
    require(balances[msg.sender] >= value);
    require(localSupply >= value);
    balances[msg.sender] -= value;
    localSupply -= value;
    backingToken.transferFrom(address(this), msg.sender, value);
  }

  // Local balance transfer.
  function transfer(address _to, uint256 _value) external returns (bool success) {
    require(balances[msg.sender] >= _value);
    balances[msg.sender] -= _value;
    balances[to] += _value;

    // Note: localSupply is unchanged!
  }

  // Initiate transfer of a balance to a different shard.
  function createCrosstransfer(uint16 _shard, address _to, uint256 _value) external returns (uint) {
    require(_shard != block.shardid);

    require(balances[msg.sender] >= _value);

    // Drop from this shard.
    balances[msg.sender] -= _value;
    localSupply -= _value;

    // Initiate transfer. This creates an ABI encoded blob.
    bytes data = abi.encodeWithSignature("applyCrosstransfer(address,uint256)", _to, _value);

    // The recipient is the same token contract on the other shard, hence `this`. Value is 0 because we are not moving any Ether.
    uint index = xmsg(_shard, this, 0, data);

    // Return the MessageReceipt.index to the user so it is easier for them to find the receipt for claiming.
    return index;
  }

  // Process an incoming transfer.
  function applyCrosstransfer(address _to, uint256 _value) external {
    require(receipt.shardid != block.shardid);

    // We assume the token contract has the same address on the originating shard.
    // Also assume msg.sender is set to the originator of the transfer (this token on the foreign shard).
    require(msg.sender == this);

    balances[_to] += _value;
    localSupply += _value;
  }
}
```
