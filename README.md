### Ki-Chain_IBC Relayer guide

## 1. Installation relayer.

Be sure that Go is installed, if not, you can use this article:

https://golang.org/doc/install 

```
# go version
go version go1.16.5 linux/amd64
```
Install the latest release via GitHub as follows:
```
$ git clone git@github.com:cosmos/relayer.git 
$ git checkout v0.9.3 
$ cd relayer && make install
```
## 2. Chek version:
```
# rly version
version: 0.9.3
commit: 4b81fa59055e3e94520bdfae1debe2fe0b747dc1
cosmos-sdk: v0.42.4
go: go1.16.5 linux/amd64
```


## 3. Prepare our nodes to accept incoming connections on the public IP, we should bit adjust our config.toml files for both nodes, in my case, it was Rizon and Kichain:
```
#laddr = "tcp://127.0.0.1:27657"
laddr = "tcp://161.X.X.122:27657"
```


in the proxy_app, [rpc] and [p2p] section in both config.toml files
then restart nodes and check that these ports are opened and accept the connection, something like this:
```
/relayer# ss -o state listening -t4 -p |sort -n |grep -E "kid|riz" |grep -e 657 -e 658
0 4096 161.x.x.122:26657 0.0.0.0:* users:(("kid",pid=351346,fd=59))
0 4096 161.x.x.122:27657 0.0.0.0:* users:(("rizond",pid=351281,fd=70))
```

if so, it`s well and we can be ready for relayer configuration,

## 4. Configuration Relayer:

-let`s initializate it:
```
rly config init
```
-create configs for Kicahin and Rizon:
```
# cd /root/relayer/configs && nano ki_config.json

{
"chain-id": "kichain-t-4",
"rpc-addr": "http://IP:26657",
"account-prefix": "tki",
"gas-adjustment": 1.5,
"gas-prices": "0.025utki",
"trusting-period": "48h"
}

# cd /root/relayer/configs && nano rizon_config.json
{
"chain-id": "groot-011",
"rpc-addr": "http://IP:27657",
"account-prefix": "rizon",
"gas-adjustment": 1.5,
"gas-prices": "0.025uatolo",
"trusting-period": "48h"
}
```
save files.

-adding our configs to relayer:
```
rly chains add -f ki_config.json
rly chains add -f rizon_config.json
```
-create or recover (import) wallets, in my case I first created new, but then restore from mnemonic my existing wallets that already have balance:
```
:~/relayer/configs# rly keys add kichain-t-4 kichainz
{"mnemonic":"coral reason blue toe **** festival wreck present network wisdom **** board","address":"tki1sqxe36gp4eqh2874jkey0a77vryfjp6nd2avvu"}
:~/relayer/configs# rly keys add groot-011 rizonz
{"mnemonic":"hunt **** please captain work unfold protect **** funny","address":"rizon1gx2qxh8a03gf7hrmwdjuf8gfjn3zq44z0cz34q"}
root@vmi640891:~/relayer/configs#

# rly keys restore groot-011 rizon "slogan *** simple"
# rly keys restore kichain-t-4 kichain "cluster denial **** warfare answe"
```

-check if wallets were added properly:
```
:~/relayer# rly keys list kichain-t-4
key(0): kichain -> tki1fgartwdtmjvh0l4tswsmw7cfhq6mc6vx3vrz49
:~/relayer# rly keys list groot-011
key(0): rizon -> rizon14yr57scz35jmxrl2pxdfr63fku5q6tzw7xlqqt
:~/relayer#
```

-Let`s add the relayer chain keys to the specific chain's configuration:
```
# rly chains edit kichain-t-4 key kichainz
# rly chains edit groot-011 key rizonz
```

-check balance (it should not be empty):
```
# rly q balance kichain-t-4
2000000transfer/channel-44/uatolo,442587utki
# rly q balance groot-011
50000transfer/channel-12/utki,7994815uatolo
```

if you don't have anything you should faucet a fund:

http://faucet.rizon.world/ 

-go to initialize the light clients:
```
# rly light init kichain-t-4 -f
successfully created light client for kichain-t-4 by trusting endpoint http://161.x.x.122:26657...
# rly light init groot-011 -f
successfully created light client for groot-011 by trusting endpoint http://161.x.x.122:27657...
```

-let's create a path across two networks:
```
rly paths generate kichain-t-4 groot-011 ibcpath --port=transfer
```
and then add the channel and connections ID, you can create it own or use existing from there:

https://dev.mintscan.io/rizon/relayers 
```
nano ~/.relayer/config/config.yaml
# rly paths show ibc --yaml
path:
src:
chain-id: kichain-t-4
client-id: 07-tendermint-32
connection-id: connection-29
channel-id: channel-44
port-id: transfer
order: UNORDERED
version: ics20-1
dst:
chain-id: groot-011
client-id: 07-tendermint-44
connection-id: connection-4
channel-id: channel-12
port-id: transfer
order: UNORDERED
version: ics20-1
strategy:
type: naive
status:
chains: true
clients: true
connection: true
channel: true
```

I used exiting,

-How to create new:
```
rly tx link transfer
```
and then check:
```
rly paths list -d
```
-start relayer, I used a screen to start it, but if you wish you can create service:
```
# screen -S relayer
# screen -r relayer
# rly start ibcpath
```

if you will see this something like this output, all goes well:
```
I[2021-09-07|17:10:12.673] - listening to tx events from kichain-t-4...
I[2021-09-07|17:10:12.673] - listening to block events from kichain-t-4...
I[2021-09-07|17:10:12.689] - listening to tx events from groot-011...
I[2021-09-07|17:10:12.689] - listening to block events from groot-011...
I[2021-09-07|17:10:13.157] - No packets to relay between [kichain-t-4]port{transfer} and [groot-011]port{transfer}
I[2021-09-07|17:10:14.914] • [kichain-t-4]@{232808} - actions(0:delegate) hash(E3E1F981836981C7B3A8C86BCAC1125FBA44B5B8CA885AB266F0662EDA9C04F3)
I[2021-09-07|17:10:17.894] • [groot-011]@{429820} - actions(0:withdraw_delegator_reward,1:withdraw_validator_commission) hash(102BD997841613F4E131CEC6202FC963FB2332AAD83183C188627FC5645933D5)
I[2021-09-07|17:10:17.895] • [groot-011]@{429820} - actions(0:delegate) hash(C192A5B426DD762F158C57E2C51D4ACC46077F88BE3AC53998F1A9A52BF4B6A1)
I[2021-09-07|17:10:17.895] • [groot-011]@{429820} - actions(0:delegate) hash(8EC3DCAF84672E667317D5C0BD481282C3E1C33D8084CBB7B65CBDBD60179D11)
I[2021-09-07|17:10:17.895] • [groot-011]@{429820} - actions(0:delegate) hash(BF5730D8365FCC39D59C05605686AB957E481F0D0834F367B05EAA7C2F9234A7)
I[2021-09-07|17:10:24.143] • [groot-011]@{429821} - actions(0:withdraw_delegator_reward,1:withdraw_validator_commission) hash(D242EFD4C6497855618A60FABC031AEC197BE5D1F024FFAACCE2FF4D1ADA7655)
I[2021-09-07|17:10:24.144] • [groot-011]@{429821} - actions(0:send) hash(B61AD116E12A468FA1A22EF22ADF3AC8003A183D3556953041D554D23CE2E83B)
I[2021-09-07|17:10:24.144] • [groot-011]@{429821} - actions(0:delegate) hash(A37ACCC1E0441E0522F4E2B0D2F2DF1DBBF5DCF9E6EBE660926EC54A262C2073)
I[2021-09-07|17:10:24.144] • [groot-011]@{429821} - actions(0:withdraw_delegator_reward,1:withdraw_validator_commission) hash(59E2793B81BED268D1FDF82910FE41163F53E319F8D03E33DA12923C5988D5CC)
```


## 5. Time for transactions, we can do it via relayer, from Rizon side and Kichain side, let`s try all variants:

#Relayer:
```
root@vmi640891:~/relayer/configs# rly tx transfer groot-011 kichain-t-4 1000000uatolo tki1fgartwdtmjvh0l4tswsmw7cfhq6mc6vx3vrz49
I[2021-09-07|18:26:44.488] ✔ [groot-011]@{430480} - msg(0:transfer) hash(58B0ABA435E7E89258E62B92F1148B1F6A1CD5C0A5B8CCDA672DC5C13C619A70)

s# rly tx transfer groot-011 kichain-t-4 1000000uatolo tki1fgartwdtmjvh0l4tswsmw7cfhq6mc6vx3vrz49
I[2021-09-07|18:30:27.769] ✔ [groot-011]@{430512} - msg(0:transfer) hash(8AB53ED241C3358AC526EF268BFF603DCAA4BC407BA2568137A8873C166A9E8F)
root@vmi640891:~/relayer/configs#

# rly q balance kichain-t-4
2000000transfer/channel-44/uatolo,500101utki
root@vmi640891:~/relayer/configs#

# rly tx transfer kichain-t-4 groot-011 50000utki rizon14yr57scz35jmxrl2pxdfr63fku5q6tzw7xlqqt -d
I[2021-09-07|18:31:59.042] ✔ [kichain-t-4]@{233589} - msg(0:transfer) hash(56A931E5D9F5E611699DA3790116FA097C7A09CA47C7FC61CCE592D57003F292)
root@vmi640891:~/relayer/configs#
```




#Rizon end:
```
# rizond tx ibc-transfer transfer transfer channel-12 tki1fgartwdtmjvh0l4tswsmw7cfhq6mc6vx3vrz49 500000uatolo --from rizon1jnkeq3hdgzvp0nyu9urqu9hqxsdye56xd5d49v --chain-id=groot-011 --fees="25uatolo" --gas=auto --node "http://161.x.x.122:27657"
Enter keyring passphrase:
gas estimate: 78763
{"body":{"messages":[{"@type":"/ibc.applications.transfer.v1.MsgTransfer","source_port":"transfer","source_channel":"channel-12","token":{"denom":"uatolo","amount":"500000"},"sender":"rizon1jnkeq3hdgzvp0nyu9urqu9hqxsdye56xd5d49v","receiver":"tki1fgartwdtmjvh0l4tswsmw7cfhq6mc6vx3vrz49","timeout_height":{"revision_number":"4","revision_height":"234590"},"timeout_timestamp":"1631032918523567503"}],"memo":"","timeout_height":"0","extension_options":[],"non_critical_extension_options":[]},"auth_info":{"signer_infos":[],"fee":{"amount":[{"denom":"uatolo","amount":"25"}],"gas_limit":"78763","payer":"","granter":""}},"signatures":[]}

confirm transaction before signing and broadcasting [y/N]: y
code: 0
codespace: ""
data: ""
gas_used: "0"
gas_wanted: "0"
height: "0"
info: ""
logs: []
raw_log: '[]'
timestamp: ""
tx: null
txhash: 0895EA74172586C9173C6D9F183D3A729C252E0461041B666AE2F4EA22DB3085
rizon#
```

#Ki-Chain end:
```
# kid tx ibc-transfer transfer transfer channel-44 rizon1jnkeq3hdgzvp0nyu9urqu9hqxsdye56xd5d49v 50000utki --from tki1fgartwdtmjvh0l4tswsmw7cfhq6mc6vx3vrz49 --fees=5000utki --gas=auto --chain-id kichain-t-4 --home ./kid --node "http://161.x.x.122:26657"
Enter keyring passphrase:
gas estimate: 78098
{"body":{"messages":[{"@type":"/ibc.applications.transfer.v1.MsgTransfer","source_port":"transfer","source_channel":"channel-44","token":{"denom":"utki","amount":"50000"},"sender":"tki1fgartwdtmjvh0l4tswsmw7cfhq6mc6vx3vrz49","receiver":"rizon1jnkeq3hdgzvp0nyu9urqu9hqxsdye56xd5d49v","timeout_height":{"revision_number":"0","revision_height":"431527"},"timeout_timestamp":"1631032929961276203"}],"memo":"","timeout_height":"0","extension_options":[],"non_critical_extension_options":[]},"auth_info":{"signer_infos":[],"fee":{"amount":[{"denom":"utki","amount":"5000"}],"gas_limit":"78098","payer":"","granter":""}},"signatures":[]}

confirm transaction before signing and broadcasting [y/N]: y
{"height":"234349","txhash":"B026833BFDA36EE24A528424EEB4A6D9B7A4775F602C16F2AE072323714559F7","codespace":"","code":0,"data":"0A0A0A087472616E73666572","raw_log":"[{\"events\":[{\"type\":\"ibc_transfer\",\"attributes\":[{\"key\":\"sender\",\"value\":\"tki1fgartwdtmjvh0l4tswsmw7cfhq6mc6vx3vrz49\"},{\"key\":\"receiver\",\"value\":\"rizon1jnkeq3hdgzvp0nyu9urqu9hqxsdye56xd5d49v\"}]},{\"type\":\"message\",\"attributes\":[{\"key\":\"action\",\"value\":\"transfer\"},{\"key\":\"sender\",\"value\":\"tki1fgartwdtmjvh0l4tswsmw7cfhq6mc6vx3vrz49\"},{\"key\":\"module\",\"value\":\"ibc_channel\"},{\"key\":\"module\",\"value\":\"transfer\"}]},{\"type\":\"send_packet\",\"attributes\":[{\"key\":\"packet_data\",\"value\":\"{\\\"amount\\\":\\\"50000\\\",\\\"denom\\\":\\\"utki\\\",\\\"receiver\\\":\\\"rizon1jnkeq3hdgzvp0nyu9urqu9hqxsdye56xd5d49v\\\",\\\"sender\\\":\\\"tki1fgartwdtmjvh0l4tswsmw7cfhq6mc6vx3vrz49\\\"}\"},{\"key\":\"packet_timeout_height\",\"value\":\"0-431527\"},{\"key\":\"packet_timeout_timestamp\",\"value\":\"1631032929961276203\"},{\"key\":\"packet_sequence\",\"value\":\"4\"},{\"key\":\"packet_src_port\",\"value\":\"transfer\"},{\"key\":\"packet_src_channel\",\"value\":\"channel-44\"},{\"key\":\"packet_dst_port\",\"value\":\"transfer\"},{\"key\":\"packet_dst_channel\",\"value\":\"channel-12\"},{\"key\":\"packet_channel_ordering\",\"value\":\"ORDER_UNORDERED\"},{\"key\":\"packet_connection\",\"value\":\"connection-1\"}]},{\"type\":\"transfer\",\"attributes\":[{\"key\":\"recipient\",\"value\":\"tki1hcea3h0ykw4jqyjhj3tnydsk22s06adhmn0qy3\"},{\"key\":\"sender\",\"value\":\"tki1fgartwdtmjvh0l4tswsmw7cfhq6mc6vx3vrz49\"},{\"key\":\"amount\",\"value\":\"50000utki\"}]}]}]","logs":[{"msg_index":0,"log":"","events":[{"type":"ibc_transfer","attributes":[{"key":"sender","value":"tki1fgartwdtmjvh0l4tswsmw7cfhq6mc6vx3vrz49"},{"key":"receiver","value":"rizon1jnkeq3hdgzvp0nyu9urqu9hqxsdye56xd5d49v"}]},{"type":"message","attributes":[{"key":"action","value":"transfer"},{"key":"sender","value":"tki1fgartwdtmjvh0l4tswsmw7cfhq6mc6vx3vrz49"},{"key":"module","value":"ibc_channel"},{"key":"module","value":"transfer"}]},{"type":"send_packet","attributes":[{"key":"packet_data","value":"{\"amount\":\"50000\",\"denom\":\"utki\",\"receiver\":\"rizon1jnkeq3hdgzvp0nyu9urqu9hqxsdye56xd5d49v\",\"sender\":\"tki1fgartwdtmjvh0l4tswsmw7cfhq6mc6vx3vrz49\"}"},{"key":"packet_timeout_height","value":"0-431527"},{"key":"packet_timeout_timestamp","value":"1631032929961276203"},{"key":"packet_sequence","value":"4"},{"key":"packet_src_port","value":"transfer"},{"key":"packet_src_channel","value":"channel-44"},{"key":"packet_dst_port","value":"transfer"},{"key":"packet_dst_channel","value":"channel-12"},{"key":"packet_channel_ordering","value":"ORDER_UNORDERED"},{"key":"packet_connection","value":"connection-1"}]},{"type":"transfer","attributes":[{"key":"recipient","value":"tki1hcea3h0ykw4jqyjhj3tnydsk22s06adhmn0qy3"},{"key":"sender","value":"tki1fgartwdtmjvh0l4tswsmw7cfhq6mc6vx3vrz49"},{"key":"amount","value":"50000utki"}]}]}],"info":"","gas_wanted":"78098","gas_used":"76574","tx":null,"timestamp":""}
root@vmi640891:~/kinode#
```

# HASH lists: 
https://dev.mintscan.io/rizon/txs/58B0ABA435E7E89258E62B92F1148B1F6A1CD5C0A5B8CCDA672DC5C13C619A70 
https://api-challenge.blockchain.ki/txs/56A931E5D9F5E611699DA3790116FA097C7A09CA47C7FC61CCE592D57003F292 
https://dev.mintscan.io/rizon/txs/0895EA74172586C9173C6D9F183D3A729C252E0461041B666AE2F4EA22DB3085 
https://api-challenge.blockchain.ki/txs/B026833BFDA36EE24A528424EEB4A6D9B7A4775F602C16F2AE072323714559F7 
https://api-challenge.blockchain.ki/txs/153E00DD6C0473F50EEE77DCAE4B3873F0B4B78BEBFC20C74C83C652C8ABA921


