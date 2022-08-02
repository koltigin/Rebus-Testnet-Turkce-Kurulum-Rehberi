# Rebus Testnet Türkçe Kurulum Rehberi
![image](https://user-images.githubusercontent.com/102043225/182368442-0bf47f34-3105-48c8-961e-ac2112e6996b.png)
##  Sistem Gereksinimleri
* 4vCPU
* 8GB RAM (Ekibin önerisi 16GB)
* 200GB SSD

## Sistemi Güncelleme
```shell
sudo apt update && sudo apt upgrade -y
```

## Gerekli Kütüphanelerin Kurulması
```shell
sudo apt install curl make build-essential wget gcc tmux jq chrony htop -y < "/dev/null"
```

## Go Kurulumu
```shell
ver="1.18.4"
wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz"
rm -rf /usr/local/go
tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz"
rm -rf "go$ver.linux-amd64.tar.gz"
echo 'export GOROOT=/usr/local/go' >> $HOME/.bash_profile
echo 'export GOPATH=$HOME/go' >> $HOME/.bash_profile
echo 'export GO111MODULE=on' >> $HOME/.bash_profile
echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> $HOME/.bash_profile
source $HOME/.bash_profile
go version
```

## Değişkenleri Yükleme
aşağıda değiştirmeniz gereken yerleri yazıyorum.
* '$NODENAME' validator adınız
* '$WALLET' cüzdan adınız
```shell
echo "export NODENAME=$NODENAME"  >> $HOME/.bash_profile
echo "export WALLET=$WALLET" >> $HOME/.bash_profile
echo "export REBUS_PORT=21" >> $HOME/.bash_profile
echo "export REBUS_CHAIN_ID=rebus_3333-1" >> $HOME/.bash_profile
source $HOME/.bash_profile
```
Düzenlediğiniz dosya aşağıdaki gibi olmalı;
---
echo "export NODENAME=Bilge"  >> $HOME/.bash_profile
echo "export WALLET=Bilge" >> $HOME/.bash_profile
echo "export REBUS_PORT=21" >> $HOME/.bash_profile
echo "export REBUS_CHAIN_ID=rebus_3333-1" >> $HOME/.bash_profile
source $HOME/.bash_profile
---

## Rebus Kurulumu
```shell
cd $HOME
git clone https://github.com/rebuschain/rebus.core
cd rebus.core && git checkout testnet
make install
```

## Uygulamayı Yapılandırma
```shell
rebusd config chain-id $REBUS_CHAIN_ID
rebusd config keyring-backend test
rebusd config node tcp://localhost:${REBUS_PORT}657
```

## Uygulamayı Başlatma
```shell
rebusd init $NODENAME --chain-id $REBUS_CHAIN_ID
```

## Genesis ve Addrbook Dosyalarının İndirilmesi
```shell
curl https://raw.githubusercontent.com/rebuschain/rebus.testnet/master/rebus_3333-1/genesis.json > ~/.rebusd/config/genesis.json
```

## Minimum GAS Ücretinin Ayarlanması
```shell
sed -i -e "s/^minimum-gas-prices *=.*/minimum-gas-prices = \"0ustrd\"/" $HOME/.rebusd/config/app.toml
```

## SEED ve PEERS Ayarlanması
```shell
SEEDS="a6d710cd9baac9e95a55525d548850c91f140cd9@3.211.101.169:26656,c296ee829f137cfe020ff293b6fc7d7c3f5eeead@54.157.52.47:26656"
PEERS=""
sed -i -e "s/^seeds *=.*/seeds = \"$SEEDS\"/; s/^persistent_peers *=.*/persistent_peers = \"$PEERS\"/" $HOME/.rebusd/config/config.toml
```

## Prometheus'u Aktif Etme
```shell
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.rebusd/config/config.toml
```

## Pruning'i Ayarlama
```shell
pruning="custom"
pruning_keep_recent="100"
pruning_keep_every="0"
pruning_interval="50"
sed -i -e "s/^pruning *=.*/pruning = \"$pruning\"/" $HOME/.rebusd/config/app.toml
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"$pruning_keep_recent\"/" $HOME/.rebusd/config/app.toml
sed -i -e "s/^pruning-keep-every *=.*/pruning-keep-every = \"$pruning_keep_every\"/" $HOME/.rebusd/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"$pruning_interval\"/" $HOME/.rebusd/config/app.toml
```

## Port Ayarlarını Yapılandırma
```shell
sed -i.bak -e "s%^proxy_app = \"tcp://127.0.0.1:26658\"%proxy_app = \"tcp://127.0.0.1:${REBUS_PORT}658\"%; s%^laddr = \"tcp://127.0.0.1:26657\"%laddr = \"tcp://127.0.0.1:${REBUS_PORT}657\"%; s%^pprof_laddr = \"localhost:6060\"%pprof_laddr = \"localhost:${REBUS_PORT}060\"%; s%^laddr = \"tcp://0.0.0.0:26656\"%laddr = \"tcp://0.0.0.0:${REBUS_PORT}656\"%; s%^prometheus_listen_addr = \":26660\"%prometheus_listen_addr = \":${REBUS_PORT}660\"%" $HOME/.rebusd/config/config.toml
sed -i.bak -e "s%^address = \"tcp://0.0.0.0:1317\"%address = \"tcp://0.0.0.0:${REBUS_PORT}317\"%; s%^address = \":8080\"%address = \":${REBUS_PORT}080\"%; s%^address = \"0.0.0.0:9090\"%address = \"0.0.0.0:${REBUS_PORT}090\"%; s%^address = \"0.0.0.0:9091\"%address = \"0.0.0.0:${REBUS_PORT}091\"%" $HOME/.rebusd/config/app.toml
```

## Zincir Verilerini Sıfırlama
```shell
rebusd tendermint unsafe-reset-all --home $HOME/.rebusd
```

## Servis Dosyası Oluşturma
```shell
sudo tee /etc/systemd/system/rebusd.service > /dev/null <<EOF
[Unit]
Description=rebus
After=network-online.target

[Service]
User=$USER
ExecStart=$(which rebusd) start --home $HOME/.rebusd
Restart=on-failure
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```

## Servisi Başlatma
```shell
systemctl daemon-reload
systemctl enable rebusd
systemctl restart rebusd
```

## Logları Kontrol Etme
```shell
journalctl -u rebusd.service -f -n 100
```  

## Cüzdan Oluşturma

### Yeni Cüzdan Oluşturma
`$WALLET` bölümünü değiştirmiyoruz kurulumun başında cüzdanımıza isim belirledik.
```shell 
rebusd keys add $WALLET
```  

### Var Olan Cüzdanı İçeri Aktarma
```shell
rebusd keys add $WALLET --recover
```

* BU AŞAMADAN SONRA NODE'UMUZUN EŞLEŞMESİNİ BEKLİYORUZ.

## Faucet  Musluk
Test token almak için Discord'da [#faucet](https://discord.gg/FScWfZqeWC) kanalından şu şekilde `$request CUZDAN_ADRESINIZ` mesaj atıyoruz.

## Cüzdan Bakiyesini Kontrol Etme
```shell
rebusd query bank balances CUZDAN_ADRESINIZ --chain-id $REBUS_CHAIN_ID
```  

## Senkronizasyonu Kontrol Etme
`false` çıktısı almaldıkça bir sonraki yani validator oluşturma adımına geçmiyoruz.
```shell
strided status 2>&1 | jq .SyncInfo
```

## Validator Oluşturma
 Aşağıdaki komutta aşağıda berlittiğim yerler dışında bir değişikli yapmanız gerekmez;
   'identity'  buraya `httpskeybase.io` sitesine üye olarak size verilen kimlik numaranızı yazıyorsunuz.
   'details'  kendiniz hakkında bilgiler verebilir ya da `Rues Community Supporter` yazabilirsiniz.
   'website'  Varsa bir siteniz yazabilirsiniz ya da `httpsforum.rues.info` olarak bırakabilirsiniz.
   'security-contact'  E-posta adresiniz.
```shell 
rebusd tx staking create-validator \
--commission-max-change-rate=0.01 \
--commission-max-rate=0.2 \
--commission-rate=0.05 \
--amount 9900000arebus \
--pubkey=$(rebusd  tendermint show-validator) \
--moniker=$NODENAME \
--chain-id=$REBUS_CHAIN_ID \
--details="Rues Community Supporter" \
--security-contact=E-POSTANIZ 
--website=http://forum.rues.info \
--identity=XXXX1111XXXX1111 \
--min-self-delegation=1 \
--fees=0.0025arebus \
--gas=300000 \
--from=$WALLET
 ```  


## Validator Linkinizi Paylaşma
Sei Discord [#testnet-apply](https://discord.gg/FScWfZqeWC) kanalından validatorumuze ait [explorer](https://stride.explorers.guru) linkini gönderiyoruz.

## DAHA FAZLA SORUNUZ VARSA REBUS TÜRKİYE TELEGRAM GRUBU

[Rebus Türkiye Telegram Sayfası](https://t.me/RebusTurkish)

## FAYDALI KOMUTLAR

### Logları Kontrol Etme 
```shell
journalctl -fu rebusd -o cat
```

### Sistemi Başlatma
```shell
systemctl start rebusd
```

### Sistemi Durdurma
```shell
systemctl stop rebusd
```

### Sistemi Yeniden Başlatma
```shell
systemctl restart rebusd
```

### Node Senkronizasyon Durumu
```shell
rebusd status 2>&1 | jq .SyncInfo
```

### Validator Bilgileri
```shell
rebusd status 2>&1 | jq .ValidatorInfo
```

### Node Bilgileri
```shell
rebusd status 2>&1 | jq .NodeInfo
```

### Node ID Öğrenme
```shell
rebusd tendermint show-node-id
```

### Node IP Adresini Öğrenme
```shell
curl icanhazip.com
```

### Peer Adresinizi Öğrenme
```shell
eecho $(strided tendermint show-node-id)@$(curl ifconfig.me):${REBUS_PORT}656
```

### Cüzdanların Listesine Bakma
```shell
rebusd keys list
```

### Cüzdanı İçeri Aktarma
```shell
rebusd keys add $WALLET --recover
```

### Cüzdanı Silme
```shell
rebusd keys delete CUZDAN_ADI
```

### Cüzdan Bakiyesine Bakma
```shell
rebusd query bank balances CUZDAN_ADRESI
```

### Bir Cüzdandan Diğer Bir Cüzdana Transfer Yapma
```shell
rebusd tx bank send CUZDAN_ADRESI GONDERILECEK_CUZDAN_ADRESI 100000000ustrd
```

### Proposal Oylamasına Katılma
```shell
rebusd tx gov vote 1 yes --from $WALLET --chain-id=$REBUS_CHAIN_ID
```

### Validatore Stake Etme  Delegate Etme
```shell
rebusd tx staking delegate $VALOPER_ADDRESS 100000000utoi --from=$WALLET --chain-id=$REBUS_CHAIN_ID --gas=auto
```

### Mevcut Validatorden Diğer Validatore Stake Etme  Redelegate Etme
```shell
rebusd tx staking redelegate MevcutValidatorAdresi StakeEdilecekYeniValidatorAdresi 100000000ustrd --from=WALLET --chain-id=$REBUS_CHAIN_ID --gas=auto
```

### Ödülleri Çekme
```shell
rebusd tx distribution withdraw-all-rewards --from=$WALLET --chain-id=$REBUS_CHAIN_ID --gas=auto
```

### Komisyon Ödüllerini Çekme
```shell
rebusd tx distribution withdraw-rewards VALIDATOR_ADRESI --from=$WALLET --commission --chain-id=$REBUS_CHAIN_ID 
```

### Validator İsmini Değiştirme
```shell
rebusd tx staking edit-validator 
--moniker=YENI_NODE_ADI 
--chain-id=$REBUS_CHAIN_ID  
--from=$WALLET
```

### Validatoru Jail Durumundan Kurtarma 
```shell
rebusd tx slashing unjail 
  --broadcast-mode=block 
  --from=$WALLET 
  --chain-id=$REBUS_CHAIN_ID  
  --gas=auto --fees 250arebus
```

### Node'u Tamamen Silme 
```shell
sudo systemctl stop rebusd
sudo systemctl disable rebusd
sudo rm /etc/systemd/system/rebusd* -rf
sudo rm $(which rebusd) -rf
sudo rm $HOME/.rebusd* -rf
sudo rm $HOME/rebus.core -rf
sed -i '/RBS_/d' ~/.bash_profile
```


### Hesaplar

[Linktree](https://linktr.ee.mehmetkoltigin)

[Twitter](https://twitter.com.mehmetkoltigin)

### Komunite 
[Forum Rues](https://forum.rues.infoindex.php)

[Telegram Rues Announcement](https://t.meRuesAnnouncement)

[Telegram Rues Chat](https://t.meRuesChat)

[Telegram Rues Node](https://t.meRuesNode)

[Telegram Rues Node Chat](https://t.meRuesNodeChat)


# Testnetler

This repo contains genesis files for the [Rebus](https://github.com/rebuschain/rebus.core) Testnets.

The current testnet is `rebus_3333-1`. You can find a list of seeds and peers to connect to in the respective directory.

For the full instructions on how to [join the testnet](https://docs.rebuschain.com/validators/joining-the-testnets/), please refer to the [official documentation](https://docs.rebuschain.com/).

Bu depo, [Rebus](https://github.com/rebuschain/rebus.core) Testnetleri için genesis dosyalarını içerir.

Mevcut test ağı `rebus_3333-1`dir. İlgili dizinde bağlanılacak seeds ve peers'lerin bir listesini bulabilirsiniz.

[Testnet](https://docs.rebuschain.com/validators/joining-the-testnets/)'e nasıl katılacağınızla ilgili tüm talimatlar için lütfen [resmi belgelere](https://docs.rebuschain.com/) bakın.
