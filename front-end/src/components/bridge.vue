<template>
    <section class="bg-white">
        <div class="row-container">
            <div class="intro text-center">
                <img src="../assets/images/smt.png">
                <div>
                    <h3> Bridge </h3>
                    <br>
                    <br>
                    <b> With bridge</b>
                    <br>
                    <b> You can exchange your assets</b>
                    <br>
                    <b>between public and consortium chain</b>
                </div>
            </div>

            <div class="wallet-container">
                <div class="bridge-item-container" style="margin-top: 50px;">
                    <div class="bridge-item-header"> ERC20 Token</div>
                    <div class="token-container">
                        <img class="token-icon" src="../assets/images/token.png">
                        <div class="token-content-container">
                            <div class="token-exchange-info" style="margin-bottom: 50px">
                                <div class="token-label">
                                    <div>Public chain tokens:</div>
                                    <div>
                                        {{tokenPublicChain}}
                                    </div>
                                </div>
                                <form>
                                    <div class="token-form">
                                        <label class="form-label label slot-label" for="pvcExInput">Amount: </label>
                                        <input class="form-input" id="pvcExInput" type="number" v-model.number="toPriAmount">
                                        <button type="button" class="form-button alpha-button alpha-button-primary"
                                                @click="tokenToPrivate">Exchange to consortium chain
                                        </button>
                                    </div>
                                </form>
                            </div>
                            <div class="token-exchange-info">
                                <div class="token-label">
                                    <div>Consortium chain tokens:</div>
                                    <div>
                                        {{tokenPrivateChain}}
                                    </div>
                                </div>
                                <form>
                                    <div class="token-form">
                                        <label class="form-label label slot-label" for="pbcExInput">Amount: </label>
                                        <input class="form-input" id="pbcExInput" type="number" v-model.number="toPubAmount">
                                        <button type="button" class="form-button alpha-button alpha-button-primary"
                                                @click="tokenToPublic">Exchange to public chain
                                        </button>
                                    </div>
                                </form>
                            </div>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </section>
</template>
<script>
    import {
        encodePrivateChainFunction, encodePublicChainFunction,
        getPrivateChainBaseTxObject,
        getPublicChainBaseTxObject,
        signTx
    } from "../assets/js/tx";

    export default {
        props: {
            tokenPrivateChain: Number,
            tokenPublicChain: Number,
            ws: WebSocket,
        },
        data() {
            return {
                socketConstants: undefined,
                account: undefined,
                toPriAmount: 0,
                toPubAmount: 0,
                wallet: {
                    publicBalance: undefined,
                    balance: undefined,
                    publicEther: undefined
                },
            }
        },
        methods: {
            tokenToPublic: function () {
                if (this.toPubAmount > this.tokenPrivateChain) {
                    this.$store.state.notifyError('Tokens not enough')
                    return
                }
                if (this.toPubAmount <= 0) {
                    this.$store.state.notifyInfo('Require token amount >0')
                    return
                }
                this.$store.state.notifyInfo('Request has been received')
                let p1 = this.axios.get(`${this.$store.state.config.httpPath}/${this.socketConstants.chainType.private}/${this.account.address}/nonce`);
                let p2 = this.axios.get(`${this.$store.state.config.httpPath}/${this.socketConstants.chainType.private}/chainId`);
                Promise.all([p1, p2]).then(([r1, r2]) => {
                    let nonce = r1.data;
                    let chainId = r2.data;
                    let data = encodePrivateChainFunction("exchange", this.account.address, this.toPubAmount);
                    let tx = getPrivateChainBaseTxObject();
                    console.log("sender:",this.account.address);
                    console.log("nonce:",nonce);
                    console.log("chainId:",chainId);
                    console.log("private key:",this.account.privateKey);
                    tx.from = this.account.address;
                    tx.nonce = nonce;
                    tx.data = data;
                    tx.chainId = chainId;
                    console.log(tx);
                    return signTx(tx, '0x' + this.account.privateKey)
                }).then((rawTx) => {
                    let payload = {
                        amount: this.toPubAmount,
                        gcuid: this.socketConstants.gcuid.exchange,
                        type: this.socketConstants.exchangeType.erc20Token,
                        source: this.socketConstants.sourceType.privateChain,
                        transaction: rawTx.slice(2),
                    };
                    this.ws.send(JSON.stringify(payload))
                }).catch(err => {
                    console.log(err)
                });
            },
            tokenToPrivate: function () {
                if (this.toPriAmount > this.tokenPublicChain) {
                    this.$store.state.notifyError('Tokens not enough')
                    return
                }
                if (this.toPriAmount <= 0) {
                    this.$store.state.notifyError('Require token amount>0')
                    return
                }
                this.$store.state.notifyInfo('Request has been received')
                let p1 = this.axios.get(`${this.$store.state.config.httpPath}/${this.socketConstants.chainType.public}/${this.account.address}/nonce`);
                let p2 = this.axios.get(`${this.$store.state.config.httpPath}/${this.socketConstants.chainType.public}/chainId`);
                Promise.all([p1, p2]).then(([r1, r2]) => {
                    let nonce = r1.data;
                    let chainId = r2.data;
                    let data = encodePublicChainFunction("exchange", this.account.address, this.toPriAmount);
                    let tx = getPublicChainBaseTxObject();
                    tx.from = this.account.address;
                    tx.nonce = nonce;
                    tx.data = data;
                    tx.chainId = chainId;
                    return signTx(tx, '0x' + this.account.privateKey)
                }).then((rawTx) => {
                    console.log("to private chain amount",this.toPriAmount);
                    let payload = {
                        amount: this.toPriAmount,
                        gcuid: this.socketConstants.gcuid.exchange,
                        type: this.socketConstants.exchangeType.erc20Token,
                        source: this.socketConstants.sourceType.publicChain,
                        transaction: rawTx.slice(2),
                    };
                    this.ws.send(JSON.stringify(payload))
                }).catch(err => {
                    console.log(err)
                });
            },
        },
        created: function () {
            this.account = this.$store.state.account;
            this.socketConstants = this.$store.state.socketConstants;
        },
    }
</script>
