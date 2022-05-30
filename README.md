# Обзор

## Обзор по запуску нод (узлов сети) на Celestia

Существует множество способов участия в сети Celestia.

Операторы нод Celestia могут запускаться в сети различным образом:

* В качестве бридж-ноды (эта нода соединяет сеть доступности данных и сеть консенсуса)
* В качестве валидатор-ноды (операторы, управляющие бридж-нодами, имеют возможность также участвовать в консенсусе, став валидатором)
* В качестве лайт-клиента (проводят сэмплирование доступности данных в сети Data Availability)

Вы можете узнать больше о том, как настроить каждую из нод, изучив это руководство.

О бридж-нодах и валидатор-нодах написано в рамках одного и того же раздела. Лайт-клиентам посвящен свой собственный раздел.

Пожалуйста, оставляйте свои отзывы об этих туториалах и гайдах. Если вы заметили какую-то ошибку или проблему, не стесняйтесь сделать Pull Request или написать Github Issue ticket!

# Настройка бридж ноды и валидатор ноды в сети Celestia

В этом гайде мы рассмотрим настройку бридж-ноды Celestia.

Бридж-ноды соединяют уровень доступности данных и уровень консенсуса, а также имеют возможность стать валидатором сети.

Если вы читаете это руководство для того, чтобы настроить валидатор в сети Celestia, следуйте по всем разделам, пока не дойдете до руководства по настройке валидатора. Если вы хотите запустить только бридж-ноду, вам не нужно выполнять шаги по настройке валидатора в конце.

# Обзор бридж-ноды

Бридж-нода Celestia обладает следующими свойствами:

1. Импорт и обработка "сырых" заголовков и блоков из надежного Core-процесса (имеется в виду доверенное RPC-соединение с узлом celestia-core) в сети Consensus. Бридж-ноды могут выполнять этот процесс Core самостоятельно (встроенным образом) или просто подключаться к удаленному эндпоинту (конечной точке). Бридж-ноды также могут быть активными валидаторами в консенсус-сети
2. Валидирование и стирание кода [erasure code] «сырых» блоков
3. Передача лайт-нодам частей блоков вместе с заголовками доступности данных

![](https://docs.celestia.org/assets/images/BridgeNodes-c9d5799bf588d3becaefb313bd03b0c2.png)

Бридж-ноды, с точки зрения имплементации, выполняют два отдельных процесса:

1. Celestia App с Celestia Core (см. [репозитарий](https://github.com/celestiaorg/celestia-app))
* Celestia App – это конечный автомат (state machine), в котором выполняется логика приложения и proof-of-stake. Celestia App построена на Cosmos SDK, а также включает в себя Celestia Core.
* Celestia Core – это уровень взаимодействия состояний, консенсуса и производства блоков. Celestia Core построено на основе Tendermint Core, модифицированного для хранения корней данных блоков со стирающим кодом, а также других изменений (см. [ADRs](https://github.com/celestiaorg/celestia-core/tree/master/docs/celestia-architecture)).
2. Celestia Node (см. [репозиторий](https://github.com/celestiaorg/celestia-node))
*  Celestia Node дополняет вышеперечисленное отдельной libp2p сетью, которая обслуживает запросы на сэмплирование доступности данных. Команда иногда называет эту сеть "halo".

# Требования к оборудованию

Для работы бридж-ноды рекомендуются следующие минимальные требования к аппаратному обеспечению:

* Оперативная память: 8 GB RAM
* CPU: 4 ядра
* Жесткий диск: 250 GB SSD Storage
* Пропускная способность: 1 Gbps на скачивание/100 Mbps на загрузку

# Установка бридж-ноды

Этот гайд был написан на примере сервера с ОС Ubuntu Linux 20.04 (LTS) x64.

## Установка зависимостей

После того, как вы настроили свой сервер, подключитесь к нему по ssh, чтобы начать настройку вместе со всеми необходимыми зависимостями для работы бридж-ноды.

Во-первых, обязательно обновите ОС:

`sudo apt update && sudo apt upgrade -y`

Эта команда установит важные пакеты, которые необходимы для выполнения многих задач, таких как загрузка файлов, компиляция и мониторинг ноды:

```
sudo apt install curl tar wget clang pkg-config libssl-dev jq build-essential \
git make ncdu -y
```

## Установите Golang

Установим Golang на этой машине для того, чтобы мы могли собрать необходимые бинарные файлы для работы бридж-ноды. В частности, Golang необходим для компиляции Celestia Application.

```
ver="1.17.2"
cd $HOME
wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz"
rm "go$ver.linux-amd64.tar.gz"
```

Теперь нам нужно добавить каталог ```/usr/local/go/bin``` в ```$PATH```:
```
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> $HOME/.bash_profile
source $HOME/.bash_profile
```

Чтобы проверить корректность установки Go, выполните команду:

```go version```

В выводе должна быть появится установленная версия:

```go version go1.17.2 linux/amd64```

# Развертывание Celestia App:

В этом разделе описана часть 1 настройки бридж-ноды Celestia: запуск демона Celestia App с внутренним узлом Celestia Core.
 
> Примечание: Убедитесь, что у вас есть не менее 100+ Гб свободного места для безопасной установки и запуска бридж-ноды.

## Установите Celestia App

Следующие шаги создадут бинарный файл с именем ```celestia-appd``` в папке ```$HOME/go/bin```, который будет использоваться позже для запуска ноды.

```
cd $HOME
rm -rf celestia-app
git clone https://github.com/celestiaorg/celestia-app.git
cd celestia-app/
git checkout tags/v0.1.0 -b v0.1.0
make install
```

Чтобы проверить, успешно ли скомпилирован бинарный файл, вы можете запустить его, используя флаг ```--help```:

```celestia-appd --help```

Вы должны увидеть примерно такой вывод (с полезными примерами команд):

```
Stargate CosmosHub App
 
Usage:
  celestia-appd [command]
 
Use "celestia-appd [command] --help" for more information about a command.
```

## Настройка P2P-сети

Теперь мы настроим P2P-сети путем клонирования репозитория:

```
cd $HOME
rm -rf networks
git clone https://github.com/celestiaorg/networks.git
```

Для инициализации сети выберите "node-name", которое характеризует вашу ноду. Параметр --chain-id, который мы используем здесь, - ```
devnet-2```. Имейте в виду, что он может измениться, если будет развернута новая тестовая сеть.

```celestia-appd init "node-name" --chain-id devnet-2```


Скопируйте файл ```genesis.json```. Для devnet-2 мы используем:

```cp $HOME/networks/devnet-2/genesis.json $HOME/.celestia-app/config```

Установите сиды и пиров:
```
SEEDS="74c0c793db07edd9b9ec17b076cea1a02dca511f@46.101.28.34:26656"
PEERS="34d4bfec8998a8fac6393a14c5ae151cf6a5762f@194.163.191.41:26656"
sed -i.bak -e "s/^seeds *=.*/seeds = \"$SEEDS\"/; s/^persistent_peers \
*=.*/persistent_peers = \"$PEERS\"/" $HOME/.celestia-app/config/config.toml
```

## Настройте прунинг

Для меньшего использования дискового пространства мы рекомендуем настроить прунинг с помощью приведенных ниже конфигураций. При желании вы можете изменить их на свои собственные конфигурации:

```
pruning="custom"
pruning_keep_recent="100"
pruning_keep_every="5000"
pruning_interval="10"
 
sed -i -e "s/^pruning *=.*/pruning = \"$pruning\"/" $HOME/.celestia-app/config/app.toml
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \
\"$pruning_keep_recent\"/" $HOME/.celestia-app/config/app.toml
sed -i -e "s/^pruning-keep-every *=.*/pruning-keep-every = \
\"$pruning_keep_every\"/" $HOME/.celestia-app/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \
\"$pruning_interval\"/" $HOME/.celestia-app/config/app.toml
```

## Сброс настроек сети

Это приведет к удалению всех папок с данными, и мы сможем начать все с чистого листа:

```celestia-appd unsafe-reset-all```

## Необязательно: быстрая синхронизация по снэпшоту
 
Синхронизация с Genesis может занять много времени, в зависимости от вашего оборудования. Используя этот метод, вы можете синхронизировать свою ноду Celestia очень быстро, загрузив свежий снимок блокчейна. Если вы хотите синхронизироваться с Genesis, то можете пропустить эту часть.

```
cd $HOME
rm -rf ~/.celestia-app/data; \
mkdir -p ~/.celestia-app/data; 
SNAP_NAME=$(curl -s https://snaps.qubelabs.io/celestia/ | \
egrep -o ">devnet-2.*tar" | tr -d ">"); wget -O - \
https://snaps.qubelabs.io/celestia/${SNAP_NAME} | tar xf - \
-C ~/.celestia-app/data/
```
 
## Запустите Celestia-App с помощью SystemD

SystemD - это служба-демон, полезная для запуска приложений в качестве фоновых процессов.

Создайте системный файл Celestia-App:

```
sudo tee <<EOF >/dev/null /etc/systemd/system/celestia-appd.service
[Unit]
Description=celestia-appd Cosmos daemon
After=network-online.target
 
[Service]
User=$USER
ExecStart=$HOME/go/bin/celestia-appd start
Restart=on-failure
RestartSec=3
LimitNOFILE=4096
 
[Install]
WantedBy=multi-user.target
EOF
```

Если файл был создан успешно, вы сможете увидеть его содержимое:

```cat /etc/systemd/system/celestia-appd.service```

Теперь загрузите адресную книгу. Есть 2 разных способа:

```wget -O $HOME/.celestia-app/config/addrbook.json "https://raw.githubusercontent.com/maxzonder/celestia/main/addrbook.json"```

ИЛИ

```wget -O $HOME/.celestia-app/config/addrbook.json "https://raw.githubusercontent.com/qubelabsio/celestia/main/addrbook.json"```

Включите и запустите демон celestia-appd:

```
sudo systemctl enable celestia-appd
sudo systemctl start celestia-appd
```

Проверьте, правильно ли запущен демон:

```sudo systemctl status celestia-appd```

Проверьте логи демона в режиме реального времени:

```sudo journalctl -u celestia-appd.service -f```

Проверьте, синхронизирована ли ваша нода, прежде чем двигаться дальше:

```curl -s localhost:26657/status | jq .result | jq .sync_info```

Убедитесь, что параметр ```"catching_up": false```, иначе оставьте процесс запущенным, пока он не будет синхронизирован.

## Кошелек

**Создайте кошелек**

Вы можете выбрать любое название для своего кошелька. В нашем примере мы использовали "validator" в качестве имени кошелька:

```celestia-appd keys add validator```

Сохраните мнемоник (сид-фразу), так как это единственный способ восстановить кошелек валидатора в случае его потери!

**Пополните кошелек**

Вы можете пополнить созданный кошелек через Discord, отправив это сообщение в канале #faucet:

```!faucet celes1xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx```

Дождитесь подтверждения об успешной отправке токенов. Чтобы проверить, успешно ли токены прибыли на кошелек получателя, выполните приведенную ниже команду, заменив публичный адрес на свой собственный:

```celestia-appd q bank balances celes1xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx```

## Делегирование стейка валидатору

Если вы хотите заделегировать больше стейка любому из валидаторов, включая свой собственный, вам понадобится ```celesvaloper``` адрес данного валидатора. Вы можете либо проверить его с помощью вышеупомянутого блок-эксплорера, либо выполнить приведенную ниже команду, чтобы получить ```celesvaloper``` адрес вашего локального кошелька валидатора на случай, если вы захотите заделегировать ему больше токенов:

```celestia-appd keys show $VALIDATOR_WALLET --bech val -a```

После ввода кодовой фразы (passphrase) для вашего кошелька вы должны увидеть аналогичный результат:

```
Enter keyring passphrase: 
celesvaloper1q3v5cugc8cdpud87u4zwy0a74uxkk6u43cv6hd
```

Чтобы заделигировать токены валидатору ```celesvaloper```, например, вы можете выполнить:

```
celestia-appd tx staking delegate \
celesvaloper1q3v5cugc8cdpud87u4zwy0a74uxkk6u43cv6hd 1000000celes \
--from=$VALIDATOR_WALLET --chain-id=devnet-2
```

Если все прошло успешно, вы должны видеть примерно такой вывод:

```
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
txhash: <tx-hash>
```

Вы можете убедится, прошел ли хэш TX, используя блок-эскплорер, введя идентификатор ```txhash```, который появился ранее.

# Разверните ноду Celestia

В этом разделе описана вторая часть настройки бридж-ноды Celestia: запуск демона Celestia Node.

## Установите Celestia Node

Установите бинарный файл Celestia Node, который будет использоваться для запуска бридж-ноды.

```
cd $HOME
rm -rf celestia-node
git clone https://github.com/celestiaorg/celestia-node.git
cd celestia-node/
git checkout tags/v0.2.0 -b v0.2.0
make install
```

Убедитесь, что бинарный файл работает, и проверьте версию с помощью команды ```celestia version```:

```
$ celestia version
Semantic version: v0.2.0
Commit: 1fcf0c0bb5d5a4e18b51cf12440ce86a84cf7a72
Build Date: Fri 04 Mar 2022 01:15:07 AM CET
System version: amd64/linux
Golang version: go1.17.5
```

## Получите доверенный хэш

> Предостережение: Для продолжения этого руководства вам понадобится запущенная celestia-app. Пожалуйста, обратитесь к [celestia-app.md](https://github.com/celestiaorg/networks/celestia-app.md) для установки.

Вам необходимо иметь проверенный сервер для инициализации бридж-ноды. Вы можете использовать ```http://localhost:26657``` для локального запуска ```celestia-app```. Доверенный хэш является необязательным флагом и необязательно должен использоваться. Если вы не передаете его, бридж-нода будет просто синхронизироваться с самого начала, что также является предпочтительным вариантом запуска.

Пример запроса к локальному приложению celestia-app для получения доверенного хэша:

```curl -s http://localhost:26657/block?height=1 | grep -A1 block_id | grep hash```

## Инициализация бридж-ноды

```celestia bridge init --core.remote <ip:port of celestia-app>```

Если вы все равно хотите использовать доверенный хэш, вот как его инициализировать:

```
celestia bridge init --core.remote <ip:port of celestia-app> \
--headers.trusted-hash <hash_from_celestia_app>
```

Например:

```celestia bridge init --core.remote tcp://127.0.0.1:26657 --headers.trusted-hash 4632277C441CA6155C4374AC56048CF4CFE3CBB2476E07A548644435980D5E17```

## Настройка бридж-ноды

Для того, чтобы ваша бридж-нода Celestia могла взаимодействовать с другими бридж-нодами, необходимо добавить их в качестве ```взаимных пиров (mutual peers)``` в файл ```config.toml``` и разрешить обмен пирами. Пожалуйста, перейдите по адресу ```networks/devnet-2/celestia-node/mutual_peers.txt```, чтобы найти список взаимных пиров.

Для получения более подробной информации о ```config.toml```, пожалуйста, перейдите по этой [ссылке](https://github.com/celestiaorg/networks/blob/master/config-toml.md).
 
```nano ~/.celestia-bridge/config.toml```
 
```
...
[P2P]
  ...
  #add multiaddresses of other celestia bridge nodes
  
  MutualPeers = [
    "/ip4/46.101.22.123/tcp/2121/p2p/12D3KooWD5wCBJXKQuDjhXFjTFMrZoysGVLtVht5hMoVbSLCbV22",
    "/ip4/x.x.x.x/tcp/yyy/p2p/abc"] 
    # the /ip4/x.x.x.x is only for example.
    # Don't add it! 
  PeerExchange = true #change this line to true. By default it's false
  ...
...
```
 
## Запуск бридж-ноды с помощью SystemD

SystemD - это служба-демон, удобная для запуска приложений в качестве фоновых процессов.

Создайте systemd файл Celestia Bridge:

```
sudo tee <<EOF >/dev/null /etc/systemd/system/celestia-bridge.service
[Unit]
Description=celestia-bridge Cosmos daemon
After=network-online.target
 
[Service]
User=$USER
ExecStart=$HOME/go/bin/celestia bridge start
Restart=on-failure
RestartSec=3
LimitNOFILE=4096
 
[Install]
WantedBy=multi-user.target
EOF
```

Если файл был создан успешно, вы сможете увидеть его содержимое:

```cat /etc/systemd/system/celestia-bridge.service```

Включите и запустите демон celestia-bridge:

```
sudo systemctl enable celestia-bridge
sudo systemctl start celestia-bridge && sudo journalctl -u \
celestia-bridge.service -f
```

Теперь бридж-нода Celestia начнет синхронизировать заголовки и хранить блоки из приложения Celestia.
 
> Примечание: При запуске мы можем увидеть мультиадрес бридж-ноды Celestia. Это необходимо для будущих подключений лайт-ноды и связи между бридж-нодами Celestia.

Например:

```/ip4/46.101.22.123/tcp/2121/p2p/12D3KooWD5wCBJXKQuDjhXFjTFMrZoysGVLtVht5hMoVbSLCbV22```

Вы должны видеть логи того, как бридж-нода синхронизируется.
 
Вы успешно настроили бридж-ноду, которая синхронизируется с сетью. Читайте дальше, если вас интересует настройка валидатор-ноды.
 
# Запустите валидатор бридж-ноду

По желанию, если вы хотите присоединиться к списку активных валидаторов, вы можете создать своего собственного валидатора, следуя инструкциям ниже. Помните, что эти шаги необходимы ТОЛЬКО в том случае, если вы хотите участвовать в консенсусе.
 
Выберите имя MONIKER по своему вкусу! Это имя валидатора, которое будет отображаться на публичных дэшбордах и в эксплорерах. ```VALIDATOR_WALLET``` должен быть тем же самым, который вы определили ранее. Параметр ```--min-self-delegation=1000000``` определяет количество токенов, которые будут самоделегироваться с вашего кошелька валидатора.

```
MONIKER="your_moniker"
VALIDATOR_WALLET="validator"
 
celestia-appd tx staking create-validator \
 --amount=1000000celes \
 --pubkey=$(celestia-appd tendermint show-validator) \
 --moniker=$MONIKER \
 --chain-id=devnet-2 \
 --commission-rate=0.1 \
 --commission-max-rate=0.2 \
 --commission-max-change-rate=0.01 \
 --min-self-delegation=1000000 \
 --from=$VALIDATOR_WALLET
```

Вам будет предложено подтвердить транзакцию:

```confirm transaction before signing and broadcasting [y/N]: y```

При вводе ```y``` должен появится примерно следующий результат:

```
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
txhash: <tx-hash>
```

Теперь вы сможете найти своего валидатора в блок-эсклорере. Например, [здесь](https://celestia.observer/validators).

# Установка лайт ноды Celestia

Это гайд по настройке лайт-ноды Celestia, которая сможет позволить вам выполнять сэмплирование данных в сети доступности данных (Data Availability, DA).

## Обзор лайт-ноды

Лайт-ноды (Celestia Light Nodes, CLN) обеспечивают доступность данных. Это наиболее распространенный способ взаимодействия с сетью Celestia.

> Примечание: В будущих имплементациях лайт-ноды смогут также публиковать транзакции (см. [ADR](https://github.com/celestiaorg/celestia-node/blob/main/docs/adr/adr-004-state-interaction.md)), хотя в Devnet’e транзакциями занимаются бридж-ноды.

![](https://docs.celestia.org/assets/images/LightNodes-6e065ce02c37e36a01707b9b1edd36b3.png)

Лайт-ноды обладают следующими свойствами:

1. Подключение к бридж-нодам в сети доступности данных (DA). Примечание: лайт-ноды не общаются друг с другом, а только с бридж-нодами.
2. Получают ExtendedHeaders, т.е. обернутые "сырые" заголовки, которые уведомляют ноды Celestia о новых заголовках блоков и соответствующих метаданных DA.
3. Выполняют сэмплирование доступности данных (Data Availability Sampling, DAS) над полученными заголовками.

## Требования к оборудованию

Для работы лайт-ноды рекомендуются следующие минимальные требования к аппаратному обеспечению:

* Оперативная память: 2 GB RAM
* CPU: 1 ядро
* Жесткий диск: 5 GB SSD Storage
* Пропускная способность: 56 Kbps на скачивание/56 Kbps на загрузку

## Установка лайт-ноды

Этот гайд был написан на примере сервера с ОС Ubuntu Linux 20.04 (LTS) x64.

### Установка зависимостей

После того, как вы настроили свой сервер, подключитесь к нему по ssh, чтобы начать настройку вместе со всеми необходимыми зависимостями для работы лайт-ноды.

Во-первых, обязательно обновите ОС:

```sudo apt update && sudo apt upgrade -y```

Эта команда установит важные пакеты, которые необходимы для выполнения многих задач, таких как загрузка файлов, компиляция и мониторинг ноды:

```
sudo apt install curl tar wget clang pkg-config libssl-dev jq \
build-essential git make ncdu -y
```

### Установите Golang:

Установим Golang на этой машине для того, чтобы мы могли собрать необходимые бинарные файлы для работы лайт-ноды. В частности, Golang необходим для компиляции Celestia Light Node.

```
ver="1.17.2"
cd $HOME
wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz"
rm "go$ver.linux-amd64.tar.gz"
Теперь нам нужно добавить каталог /usr/local/go/bin в $PATH:
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> $HOME/.bash_profile
source $HOME/.bash_profile
```

Чтобы проверить правильность установки Go, выполните команду:

```go version```

В выводе должна быть указана установленная версия:

```go version go1.17.2 linux/amd64```

## Установите Celestia Node

> Примечание: Убедитесь, что у вас есть не менее 5+ Гб свободного места для установки лайт-ноды Celestia.

Установите бинарный файл Celestia Node. Убедитесь, что у вас установлены ```git``` и ```golang```.

```
git clone https://github.com/celestiaorg/celestia-node.git
cd celestia-node/
git checkout tags/v0.2.0 -b v0.2.0
make install
```

### Запустите лайт-ноду

> Если вы хотите подключиться к бридж-ноде Celestia и начать синхронизацию лайт-ноды Celestia с не-genesis хэша, то рассмотрите возможность редактирования файла ```config.toml```.

Более подробную информацию о ```config.toml``` можно найти [здесь](https://github.com/celestiaorg/networks/blob/master/config-toml.md).

Инициализируйте лайт-ноду

Выполните следующую команду:

```celestia light init```

Запустите лайт-ноду

Запустите лайт-ноду в качестве процесса-демона в фоновом режиме.

```
sudo tee <<EOF >/dev/null /etc/systemd/system/celestia-lightd.service
[Unit]
Description=celestia-lightd Light Node
After=network-online.target

[Service]
User=$USER
ExecStart=$HOME/go/bin/celestia light start
Restart=on-failure
RestartSec=3
LimitNOFILE=4096

[Install]
WantedBy=multi-user.target
EOF
```

Если файл был создан успешно, вы сможете увидеть его содержимое:

```cat /etc/systemd/system/celestia-lightd.service```

Запустите демона

Включите и запустите демон celestia-lightd:

```
sudo systemctl enable celestia-lightd
sudo systemctl start celestia-lightd
```

Проверьте состояние демона

Проверьте, правильно ли запущен демон:

```sudo systemctl status celestia-lightd```

Проверье логи демона

Проверяйте логи демона в режиме реального времени:

```sudo journalctl -u celestia-lightd.service -f```

Теперь лайт-нода Celestia начнет процесс синхронизации заголовков. После завершения синхронизации лайт-ноду выполнит сэмплирование доступности данных (DAS) с бридж-ноды.

## Сэмплирование доступности данных (DAS)

### Предварительные условия:

Для продолжения, вам понадобятся:
* Лайт-нода Celestia, подключенная к бридж-ноде
* Кошелек Celestia

Откройте 2 окнатерминала, чтобы посмотреть на то, как работает сэмплирование доступности данных:

1. В 1 окне терминала откройте журнал логов лайт-ноды
2. В 2 окне терминала используйте CLI приложения Celestia для отправки в сеть платной транзакции payForMessage

### Создайте кошелек

Во-первых, вам нужен кошелек, чтобы оплатить транзакцию.

Вариант 1: Используйте кошелек Keplr, который имеет бета-поддержку Celestia.

Ознакомьтесь с эксплорером [здесь](https://staking.celestia.observer/)

Вариант 2: Загрузите бинарный файл Celestia App, в котором есть CLI для создания кошельков. 

В случае выбора 2 варианта проделайте следующие шаги:

Скачайте бинарный файл Celestia-App

Скачайте бинарный файл celestia-appd в папку ```$HOME/go/bin```, который будет использоваться для создания кошельков.

```
git clone https://github.com/celestiaorg/celestia-app.git
cd celestia-app/
git checkout tags/v0.1.0 -b v0.1.0
make install
```

Проверьте корректность комплияции бинарного файла

Чтобы проверить, корректно ли скомпилировался бинарный файл, вы можете запустить его, используя флаг ```--help```:

```celestia-appd –help```

Создайте кошелек

Создайте кошелек с любым произвольным именем кошелька, например mywallet:
 
```celestia-appd keys add mywallet```

Сохраните мнемоник (сид-фразу), так как это единственный способ восстановить кошелек валидатора в случае его потери!

### Пополните кошелек

Вы можете пополнить созданный кошелек через Discord, отправив это сообщение в канале #faucet:

```!faucet celes1xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx```

Дождитесь подтверждения об успешной отправке токенов. Чтобы проверить, успешно ли токены прибыли на кошелек получателя, выполните приведенную ниже команду, заменив публичный адрес на свой собственный:

```celestia-appd q bank balances celes1xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx```

### Отправьте транзакцию

Во втором окне терминала отправьте транзакцию ```payForMessage``` с помощью ```celestia-appd``` (или сделайте это в кошельке):

```
celestia-appd tx payment payForMessage <hex_namespace> <hex_message> \
--from <wallet_name> --keyring-backend <keyring-name> \
--chain-id <chain_name>
```

Например:

```
celestia-appd tx payment payForMessage 0102030405060708 \
68656c6c6f43656c6573746961444153 --from myWallet --keyring-backend test \
--chain-id devnet-2
```

### Наблюдайте за работой сэмплирования доступности данных (DAS) в действии

В логах лайт-ноды вы должны будете увидеть, как работает сэмплирование доступности данных:
Например:

```
INFO das das/daser.go:96 sampling successful {“height”: 81547, “hash”: \
“DE0B0EB63193FC34225BD55CCD3841C701BE841F29523C428CE3685F72246D94”, \
“square width”: 2, “finished (s)”: 0.000117466}
```
# Ресурсы для DevOps

В этом разделе будет представлен обзор различных доступных ресурсов для разработчиков и операторов узлов (нод), желающих протестировать архитектуру Celestia.

## Docker Compose

Одним из ресурсов для ознакомления с инфраструктурой Docker Compose для Celestia является репозиторий [ephemeral-cluster](https://github.com/celestiaorg/ephemeral-cluster). Этот репозиторий предоставляет инфраструктуру для проведения интеграционных тестов для разработки путем запуска экземпляра Docker Compose в Celestia devnet. Его не рекомендуется использовать как таковой для запуска нод по отдельности.
