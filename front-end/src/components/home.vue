<template>

    <body id="page-top">
    <nav class="navbar navbar-expand-lg navbar-dark fixed-top" id="mainNav">
        <div class="container">
            <a class="navbar-brand  js-scroll-trigger" href="#" v-scroll-to="'#page-top'">IEEE SecDev 2018</a>
            <button class="navbar-toggler" type="button" data-toggle="collapse" data-target="#navbarResponsive"
                    aria-controls="navbarResponsive" aria-expanded="false" aria-label="Toggle navigation">
                Menu
                <i class="fas fa-bars"></i>
            </button>
            <div class="collapse navbar-collapse" id="navbarResponsive">
                <ul class="navbar-nav  ml-auto">
                    <li class="nav-item text-uppercase">
                        <a class="nav-link" href="#" v-scroll-to="'#sm'">Game Machine</a>
                    </li>
                    <li class="nav-item text-uppercase">
                        <a class="nav-link" href="#" v-scroll-to="'#wallet'">Bridge</a>
                    </li>
                </ul>
            </div>
        </div>
    </nav>
    <header class="masthead">
        <div class="container">
            <div class="intro-text">
                <div class="intro-lead-in"></div>
            </div>
        </div>
    </header>
    <div>
        <a id="sm">
            <vue-slotmachine :ws="ws"
                             :tokenPrivateChain="tokenPrivateChain"
                             :tokenPublicChain="tokenPublicChain"
                             :renewDoubleRound="renewDoubleRound"
            >

            </vue-slotmachine>
        </a>
        <a id="wallet">
            <vue-bridge :ws="ws"
                        :tokenPrivateChain="tokenPrivateChain"
                        :tokenPublicChain="tokenPublicChain"
            >

            </vue-bridge>
        </a>

    </div>
    </body>
</template>
<script>
    export default {
        components: {
            'vue-slotmachine': () => import('@/components/SlotMachine.vue'),
            'vue-bridge': () => import('@/components/bridge.vue'),
        },
        data() {
            return {
                ws: undefined,
                renewDoubleRound: 0,
                tokenPrivateChain: undefined,
                tokenPublicChain: undefined,
                account: undefined,
                socketConstants: undefined,
            }
        },
        methods: {
            initialWS: function () {
                this.ws = new WebSocket(this.$store.state.config.wsPath);
                this.ws.onopen = e => {
                    console.log("websocket open");
                    this.reconnecting = false;
                    this.login()
                };
                this.ws.onmessage = e => {
                    let res = JSON.parse(e.data);
                    let socketConstants = this.socketConstants;
                    console.log(res);
                    switch (res.gcuid) {
                        case socketConstants.gcuid.login:
                            if (res.status === 0) {
                                console.log("login successs");
                                this.initializeChainInfo();
                            } else {
                                console.log("login fail");
                                console.log(res.reason);
                            }
                            break;
                        case socketConstants.gcuid.logout:  //TODO
                            break;
                        case socketConstants.gcuid.reward:
                            if (res.status === 0) {
                                console.log(`reward successs,type: ${res.type}`)
                                if (res.type === socketConstants.rewardType.erc20Token) {
                                    this.tokenPrivateChain += res.amount;
                                } else {
                                }
                            } else {
                                this.$store.state.notifyError("reward fail");
                                console.log("reward fail");
                                console.log(res.reason);
                            }
                            break;
                        case socketConstants.gcuid.consume:
                            if (res.status === 0) {
                                console.log(`consume successs, type:${res.type}`);
                                switch (res.type) {
                                    case socketConstants.consumeType.erc20Token:
                                        this.tokenPrivateChain -= res.amount;
                                        this.renewDoubleRound += 1;
                                        break;
                                }
                            } else {
                                this.$store.state.notifyError("consume fail");
                                console.log("consume fail");
                                console.log(res.reason);
                            }
                            break;
                        case socketConstants.gcuid.exchange:
                            if (res.status === 0) {
                                console.log(`exchange success, type:${res.type}, source: ${res.source}`);
                                if (res.type === socketConstants.exchangeType.erc20Token) {  //exchange token
                                    if (res.state === socketConstants.exchangeState.pending) {
                                        console.log("exchange pending")
                                        if (res.source === socketConstants.sourceType.publicChain) {
                                            this.tokenPublicChain -= res.amount;
                                            this.$store.state.notifySuccess('Pending, exchanging token to private chain');
                                        } else {
                                            this.tokenPrivateChain -= res.amount;
                                            this.$store.state.notifySuccess('Pending, exchanging token to public chain');
                                        }
                                    } else {
                                        console.log("exchange finish");
                                        if (res.source === socketConstants.sourceType.publicChain) {
                                            this.tokenPrivateChain += res.amount;
                                            this.$store.state.notifySuccess('Finish, exchanged token to private chain');
                                        } else {
                                            this.tokenPublicChain += res.amount;
                                            this.$store.state.notifySuccess('Finish, exchanged token to public chain');
                                        }
                                    }
                                } else {

                                }
                            } else {
                                this.$store.state.notifyError("exchange fail");
                                console.log("exchange fail");
                                console.log(res.reason);
                            }
                            break;
                        case socketConstants.gcuid.getBalance:
                            if (res.status === 0) {
                                console.log(`get balance success, amount:${res.amount}, from:${res.source}`);
                                if (res.source === socketConstants.sourceType.publicChain) {
                                    this.tokenPublicChain = res.amount;
                                } else {
                                    this.tokenPrivateChain = res.amount;
                                }
                            } else {
                                this.$store.state.notifyError("get balance fail");
                                console.log("get balance fail");
                                console.log(res.reason);
                            }
                            break;
                        default:
                            console.log("unknown response");
                    }
                };
                this.ws.onclose = e => {
                    console.log("websocket close");
                    this.reconnecting = true;
                    this.reconnect();
                    // TODO ERROR WARNING
                };
            },
            reconnect: function () {
                this.wsReconnect = setTimeout(this.initialWS, 2000);
            },
            login: function () {
                console.log("login")
                let payload = {
                    gcuid: this.socketConstants.gcuid.login,
                    address: this.account.address,
                }
                this.ws.send(JSON.stringify(payload))
            },
            getBalance: function (source) {
                console.log(`get balance from ${source}`);
                let payload = {
                    gcuid: this.socketConstants.gcuid.getBalance,
                    source: source,
                    address: this.account.address,
                };
                this.ws.send(JSON.stringify(payload));
            },
            initializeChainInfo: function () {
                this.getBalance(this.socketConstants.sourceType.publicChain);
                this.getBalance(this.socketConstants.sourceType.privateChain);
            },
        },
        mounted: function () {
            var el = document.getElementById('mainNav');

            function scrollHandle() {
                if (document.documentElement.scrollTop > 100) {
                    el.classList.add("navbar-shrink")
                } else {
                    el.classList.remove("navbar-shrink")
                }
            }

            window.addEventListener('scroll', scrollHandle);
        },
        created: function () {
            this.account = this.$store.state.account;
            this.socketConstants = this.$store.state.socketConstants;
            this.initialWS();
        },
        beforeCreate: function () {
            let socketConstants = {
                gcuid: {
                    login: 0,
                    logout: 1,
                    reward: 2,
                    consume: 3,
                    exchange: 4,
                    getBalance: 5,
                },
                responseStatus: {
                    success: 0,
                    fail: 1,
                    pending: 2,
                },
                rewardType: {
                    erc20Token: 0,
                },
                consumeType: {
                    erc20Token: 0,
                    weapon: 1,
                    armor: 2,
                },
                exchangeType: {
                    erc20Token: 0,
                },
                sourceType: {
                    privateChain: 0,
                    publicChain: 1,
                },
                exchangeState: {
                    pending: 0,
                    finish: 1,
                },
                chainType: {
                    public: "public",
                    private: "private",
                },
                geneMap: {
                    0: "Agumon",
                    1: "Charmander"
                },
                levelMap: {
                    0: "Level0",
                    1: "Level1",
                    2: "Level2",
                },
                weaponedMap: {
                    true: 'Weaponed',
                    false: '',
                },
            };
            this.$store.commit('setSocketConstants', socketConstants);
        }
    }
</script>
