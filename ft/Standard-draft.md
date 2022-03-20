# Summary

A standard interface for Jettons (fungible tokens).

# Motivation

A standard interface will greatly simplify interaction and display of different tokenized assets.

Jetton standard describes:
* The way of jetton transfers.
* The way of retrieving common information (name, circulating supply, etc) about given Jetton asset.


# Specification

Here and following we use "Jetton" with capital `J` as designation for entirety of tokens of the same type, while "jetton" with `j` as designation of amount of tokens of some type.

Jettons are organized as follows: each Jetton has master smart-contract which is used to mint new jettons, account for circulating supply and provide common information. At the same time information about amount of jettons owned by each user is stores in decentralized manner in individual (for each owner) smart-contracts called "jetton-wallets".

Example: if you release a Jetton with circulating supply of 200 jetton which are ownerd by 3 people, then you will deploy 4 contracts: 1 Jetton-master and 3 jetton-wallets.

## Jetton wallet smart contract

Must implement:

### Internal message handlers

#### 1. `transfer`

**Request**

TL-B schema of inbound message: 

```
transfer#f8a7ea5 query_id:uint64 amount:(VarUInteger 16) destination:MsgAddress
                 response_destination:MsgAddress custom_payload:(Maybe ^Cell)
                 forward_ton_amount:(VarUInteger 16) forward_payload:(Either Cell ^Cell)
                 = InternalMsgBody;
```

`query_id` - arbitrary request number.

`amount` - amount of transferred jettons in elementary units

`destination` - address of the new owner of the jettons.

`response_destination` - address where to send a response with confirmation of a successful transfer and the rest of the  incoming message coins.

`custom_payload` - optional custom data (which is used by either sender or receiver jetton wallet for inner logic).

`forward_ton_amount` - the amount of nanotons to be sent to the destination address.

`forward_payload` - optional custom data that should be sent to the destination address.

**Should be rejected if:**

1. message is not from the owner.

2. there is no enough jettons on the sender wallet

3. there is no enough TON (with respect to jetton own storage fee guidelines and operation costs) to process operation, deploy receiver's jetton-wallet and send `forward_ton_amount`.

4. After processing the request, the receiver's jetton-wallet **must** send at least `in_msg_value - forward_amount - 2 * max_tx_gas_price` to the `response_destination` address.

    If the sender jetton-wallet cannot guarantee this, it must immediately stop executing the request and throw error.

    `max_tx_gas_price` is the price in Toncoins of maximum transaction gas limit of NFT habitat workchain. For the basechain it can be obtained from [`ConfigParam 21`](https://github.com/ton-blockchain/ton/blob/78e72d3ef8f31706f30debaf97b0d9a2dfa35475/crypto/block/block.tlb#L660) from `gas_limit` field.

**Otherwise should do:**

1. decrease jetton amount on sender wallet by `amount` and send message which increase jetton amount on receiver wallet (and optionally deploy it).

2. if `forward_amount > 0` ensure that receiver's jetton-wallet send message to `destination` address with `forward_amount` nanotons attached and with the following layout:

    TL-B schema: 

```
transfer_notification#1b26d24 query_id:uint64 amount:(VarUInteger 16) jetton_master:MsgAddress
                              sender:MsgAddress forward_payload:(Either Cell ^Cell)
                              = InternalMsgBody;
```

    `query_id` should be equal with request's `query_id`.

    `jetton` is address of the Jetton master-contract.

    `amount` amount of transferred jettons.
    
    `sender` is address of the previous owner of transferred jettons.

    `forward_payload` should be equal with request's `forward_payload`.

    If `forward_amount` is equal to zero, notification message should not be sent.

3. Receiver's wallet should send all excesses of incoming message coins to `response_destination` with the following layout:

    TL-B schema: `excesses#d53276db query_id:uint64 = InternalMsgBody;`

    `query_id` should be equal with request's `query_id`.

#### 2. `burn`

**Request**

TL-B schema of inbound message: 

```
burn#595f07bc query_id:uint64 amount:(VarUInteger 16) 
              response_destination:MsgAddress custom_payload:(Maybe ^Cell)
              = InternalMsgBody;
```

`query_id` - arbitrary request number.

`amount` - amount of burned jettons

`response_destination` - address where to send a response with confirmation of a successful burn and the rest of the  incoming message coins.

`custom_payload` - optional custom data.

**Should be rejected if:**

1. message is not from the owner.

2. there is no enough jettons on the sender wallet


3. There is no enough TONs to send after processing the request at least `in_msg_value -  max_tx_gas_price` to the `response_destination` address.

    If the sender jetton-wallet cannot guarantee this, it must immediately stop executing the request and throw error.

**Otherwise should do:**

1. decrease jetton amount on burner wallet by `amount` and send notification to jetton master with information about burn.

2. Jetton master should send all excesses of incoming message coins to `response_destination` with the following layout:

    TL-B schema: `excesses#d53276db query_id:uint64 = InternalMsgBody;`

    `query_id` should be equal with request's `query_id`.


### Get-methods
1. `get_wallet_data()` returns `(int balance, slice owner, slice jetton, cell token_wallet_code)`

    `balance` - (uint256) amount of jettons on wallet.
    `owner` - (MsgAddress) address of wallet owner;
    `jetton` - (MsgAddress) address of Jetton master-address;
    `token_wallet_code` - (cell) with code of this wallet;



## Jetton master contract



### Get-methods

1. `get_jetton_data()` returns `(int supply, int mintable, slice controller, cell jetton_content, cell token_wallet_code)`

    `supply` - (integer) - the total number of issues jettons

    `mintable` - (-1/0) - flag which indicates whether number of jettons can increase

    `controller` - (MsgAddressInt) - address of smart-contrac which control Jetton
    
    `jetton_content` - cell - data in accordance to https://github.com/ton-blockchain/TIPs/issues/64
    
    `token_wallet_code` - cell - code of wallet for that jetton

# TL-B schema

```
nothing$0 {X:Type} = Maybe X;
just$1 {X:Type} value:X = Maybe X;
left$0 {X:Type} {Y:Type} value:X = Either X Y;
right$1 {X:Type} {Y:Type} value:Y = Either X Y;
var_uint$_ {n:#} len:(#< n) value:(uint (len * 8))
         = VarUInteger n;

addr_none$00 = MsgAddressExt;
addr_extern$01 len:(## 9) external_address:(bits len) 
             = MsgAddressExt;
anycast_info$_ depth:(#<= 30) { depth >= 1 }
   rewrite_pfx:(bits depth) = Anycast;
addr_std$10 anycast:(Maybe Anycast) 
   workchain_id:int8 address:bits256  = MsgAddressInt;
addr_var$11 anycast:(Maybe Anycast) addr_len:(## 9) 
   workchain_id:int32 address:(bits addr_len) = MsgAddressInt;
_ _:MsgAddressInt = MsgAddress;
_ _:MsgAddressExt = MsgAddress;

transfer query_id:uint64 amount:(VarUInteger 16) destination:MsgAddress
           response_destination:MsgAddress custom_payload:(Maybe ^Cell)
           forward_ton_amount:(VarUInteger 16) forward_payload:(Either Cell ^Cell)
           = InternalMsgBody;

transfer_notification query_id:uint64 amount:(VarUInteger 16) jetton_master:MsgAddress
           sender:MsgAddress forward_payload:(Either Cell ^Cell)
           = InternalMsgBody;

excesses query_id:uint64 = InternalMsgBody;

burn query_id:uint64 amount:(VarUInteger 16) 
       response_destination:MsgAddress custom_payload:(Maybe ^Cell)
       = InternalMsgBody;

// ----- Unspecified by standard, but suggested format of internal message

internal_transfer  query_id:uint64 amount:(VarUInteger 16) from:MsgAddress
                     response_address:MsgAddress
                     forward_ton_amount:(VarUInteger 16)
                     forward_payload:(Either Cell ^Cell) 
                     = InternalMsgBody;
burn_notification query_id:uint64 amount:(VarUInteger 16) 
       sender:MsgAddress response_destination:MsgAddress
       = InternalMsgBody;
master_request query_id:uint64 amount:(VarUInteger 16) from:MsgAddress
                     response_address:MsgAddress
                     forward_ton_amount:(VarUInteger 16)
                     forward_payload:(Either Cell ^Cell) 
                     = InternalMsgBody;
  
```

`crc32('transfer query_id:uint64 amount:VarUInteger 16 destination:MsgAddress response_destination:MsgAddress custom_payload:Maybe ^Cell forward_ton_amount:VarUInteger 16 forward_payload:Either Cell ^Cell = InternalMsgBody') = 0x8f8a7ea5 & 0x7fffffff = 0xf8a7ea5`

`crc32('transfer_notification query_id:uint64 amount:VarUInteger 16 jetton_master:MsgAddress sender:MsgAddress forward_payload:Either Cell ^Cell = InternalMsgBody') = 0x81b26d24 & 0x7fffffff = 0x1b26d24`

`crc32('excesses query_id:uint64 = InternalMsgBody') = 0x553276db | 0x80000000 = 0xd53276db`

`crc32('burn query_id:uint64 amount:VarUInteger 16 response_destination:MsgAddress custom_payload:Maybe ^Cell = InternalMsgBody') = 0x595f07bc & 0x7fffffff = 0x595f07bc`

`crc32('internal_transfer query_id:uint64 amount:VarUInteger 16 from:MsgAddress response_address:MsgAddress forward_ton_amount:VarUInteger 16 forward_payload:Either Cell ^Cell = InternalMsgBody') = 0x978d4519 & 0x7fffffff = 0x178d4519`

`crc32('burn_notification query_id:uint64 amount:VarUInteger 16 sender:MsgAddress response_destination:MsgAddress = InternalMsgBody') = 0x7bdd97de & 0x7fffffff = 0x7bdd97de`

`crc32('master_request query_id:uint64 amount:VarUInteger 16 from:MsgAddress response_address:MsgAddress forward_ton_amount:VarUInteger 16 forward_payload:Either Cell ^Cell = InternalMsgBody') = 0xc932b42c & 0x7fffffff = 0x4932b42c`