<template>
  <div id="app">
    <button @click="isTestnet = !isTestnet">
      USING {{ isTestnet ? 'TESTNET' : 'MAINNET' }}. Click to use {{ isTestnet ? 'MAINNET' : 'TESTNET' }}
    </button>
    <h1>Welcome to FLUX multisig</h1>
    <hr>
    <h3>
      Keypair generation tool
    </h3>
    <p>
      This tool generates a Public Key and corresponding private key that can later be used for MultiSignature address.
    </p>
    <div>
      <button @click="generateKeypair">
        Generate Keypair
      </button>
      <br>
      Public Key: {{ keypair.publickey }}
      <br>
      Private Key: {{ keypair.privatekey }}
    </div>
    <hr>
    <h3>
      Multi Signature address generation tool
    </h3>
    <p>
      This tool generates a Multisignature address from given set of Public Keys and required signatures needed.
    </p>
    <div>
      <div
        v-for="n in inputs"
        :key="n"
      >
        Public Key {{ n }}: <input
          v-model="publickeys[n-1]"
          class="pubkey"
        >
      </div>
      <br>
      <button @click="addPubKey">
        Add Public Key Input
      </button>
      <br><br>
      <div>
        Signatures needed: <input v-model="reqsig">
        <br><br>
        <button @click="generateMultisig">
          Generate Multisig Address
        </button>
        <br><br>
        Address: {{ multisig.address }}
        <br>
        Redeem Script: {{ multisig.redeemScript }}
      </div>
    </div>
    <hr>
    <div>
      <h3>
        Coin Control
      </h3>
      <p>
        This tool lets you select which outputs you would like to spend
      </p>
      <br>
      Address: <input
        v-model="coincontrol.address"
        class="pubkey"
      >
      <p>
        <button @click="fetchUtxoSet">
          View Spendable Transactions
        </button>
        <br><br>
        {{ coincontrol.errorMsg }}
      </p>
      <div v-if="coincontrol.show">
        <table
          id="coincontroltable"
          class="center"
        >
          <thead>
            <tr>
              <th />
              <th />
              <th>Confirmations</th>
              <th>Txid</th>
              <th>Vout</th>
              <th>Amount</th>
            </tr>
          </thead>
          <tbody>
            <tr
              v-for="(item, index) in coincontrol.getrows"
              :key="index"
            >
              <td>{{ (coincontrol.currentPage - 1) * 10 + index }}</td>
              <td>
                <input
                  v-model="coincontrol.selected[(coincontrol.currentPage - 1) * 10 + index]"
                  aria-labelledby="coinControl"
                  type="checkbox"
                  class="checkbox"
                  @change="checkboxClicked($event.target.checked, (coincontrol.currentPage - 1) * 10 + index);"
                >
              </td>
              <td>{{ item.confirmations }}</td>
              <td>{{ item.txid }}</td>
              <td>{{ item.vout }}</td>
              <td>{{ item.amount }}</td>
            </tr>
          </tbody>
        </table>
        <br>
        <div class="pagination">
          <div
            v-for="(i) in coincontrol.numpages"
            :key="i"
            class="number"
            :class="[i == coincontrol.currentPage ? 'active' : '']"
            @click="change_page(i)"
          >
            {{ i }}
          </div>
        </div>
      </div>
    </div>
    <hr>
    <div>
      <h3>
        Build Unsigned Transaction
      </h3>
      <p>
        This tool helps you build an unsigned transaction.
      </p>

      <p>
        Avoid Flux Node Collateral Amounts: <input
          id="checkbox"
          v-model="avoidFluxNodeAmounts"
          aria-labelledby="avoindFluxNodeAmounts"
          type="checkbox"
        >
        <br>
        Select All Flux (Ignores the Amount - Max 2000 inputs): <input
          id="checkbox"
          v-model="sendAllFlux"
          aria-labelledby="sendAll"
          type="checkbox"
        >
        <br>
      </p>

      <div
        v-show="coincontrol.selectedValueSats"
        style="color:green;"
      >
        Coin Control Active: <b>{{ coincontrol.selectedValueAmount }}</b>
      </div>
      <br>
      My Address: <input
        v-model="unsignedTx.myAddress"
        class="pubkey"
      >
      <br>
      Receiver Address: <input
        v-model="unsignedTx.receiver"
        class="pubkey"
      >
      <br>
      Amount to Send: <input
        v-model="unsignedTx.amount"
        class="pubkey"
        :disabled="sendAllFlux"
      >
      <br>
      Message to Send: <input
        v-model="unsignedTx.message"
        class="pubkey"
      >
      <br>
      <br>
      <button @click="buildUnsignedRawTx">
        Build!
      </button>
      <br><br>
      Raw Transaction: {{ unsignedTx.hex }}
    </div>
    <hr>
    <div>
      <h3>
        Sign Transaction
      </h3>
      <p>
        This tool signs a transaction that is being performed from a multisig address with your private key. Fee is 0 satoshis.
      </p>
      My Private Key: <input
        v-model="signedTx.privatekey"
        class="pubkey"
      >
      <br>
      My Multisig address Redeem Script: <textarea
        v-model="signedTx.redeemScript"
        aria-labelledby="redeemScript"
        class="pubkey"
      />
      <br>
      Transaction to sign: <textarea
        v-model="signedTx.rawtx"
        aria-labelledby="transactionToSign"
        class="pubkey"
      />
      <br>
      <br>
      <button @click="signTransaction">
        Sign!
      </button>
      <br><br>
      Raw Transaction: {{ signedTx.hex }}
    </div>
    <hr>
    <div>
      <h3>
        Finalise Transaction
      </h3>
      <p>
        This tool finalises a transaction that was previously signed.
      </p>
      Transaction to finalise: <textarea
        v-model="finalisedTx.rawtx"
        aria-labelledby="transactionToFinalise"
        class="pubkey"
      />
      <br>
      <br>
      <button @click="finaliseTransaction">
        Finalise!
      </button>
      <br><br>
      Finalised Transaction: {{ finalisedTx.hex }}
    </div>
  </div>
</template>

<script>
const bitgotx = require('bitgo-utxo-lib');
const axios = require('axios');

export default {
  name: 'App',
  data() {
    return {
      keypair: {
        publickey: '',
        privatekey: '',
      },
      multisig: {
        address: '',
        redeemScript: '',
      },
      publickeys: [],
      inputs: 1,
      reqsig: 1,
      unsignedTx: {
        myAddress: '',
        receiver: '',
        amount: 0,
        message: '',
        hex: '',
      },
      signedTx: {
        rawtx: '',
        privatekey: '',
        redeemScript: '',
        hex: '',
      },
      finalisedTx: {
        rawtx: '',
        hex: '',
      },
      coincontrol: {
        address: '',
        utxos: [],
        selected: [],
        errorMsg: '',
        currentPage: 1,
        elementsPerPage: 10,
        getrows: 0,
        numpages: 0,
        show: false,
        selectedValueSats: 0,
        selectedValueAmount: 0,
      },
      avoidFluxNodeAmounts: true,
      sendAllFlux: false,
      isTestnet: false,
      mainnetExplorer: 'https://explorer.runonflux.io',
      testnetExplorer: 'https://testnet.runonflux.io',
    };
  },
  methods: {
    generateKeypair() {
      const network = this.isTestnet ? bitgotx.networks.fluxtestnet : bitgotx.networks.zelcash;
      const keyPair = bitgotx.ECPair.makeRandom({ network });
      // console.log(keyPair);
      const pubKey = keyPair.getPublicKeyBuffer().toString('hex');
      // const address = keyPair.getAddress();
      // console.log(address);

      this.keypair.publickey = pubKey;
      this.keypair.privatekey = keyPair.toWIF();
    },
    generateMultisig() {
      try {
        const filteredPK = this.publickeys.filter((el) => el != null && el !== '' && el !== undefined);

        const pubKeysBuffer = filteredPK.map((hex) => Buffer.from(hex, 'hex'));

        const redeemScript = bitgotx.script.multisig.output.encode(Number(this.reqsig), pubKeysBuffer);
        const redeemScriptHex = redeemScript.toString('hex');
        const scriptPubKey = bitgotx.script.scriptHash.output.encode(bitgotx.crypto.hash160(redeemScript));

        const network = this.isTestnet ? bitgotx.networks.fluxtestnet : bitgotx.networks.zelcash;
        const address = bitgotx.address.fromOutputScript(scriptPubKey, network);
        this.multisig.address = address;
        this.multisig.redeemScript = redeemScriptHex;
      } catch (e) {
        console.log(e);
        this.multisig.address = e.message;
        this.multisig.redeemScript = e.message;
      }
    },
    addPubKey() {
      this.inputs += 1;
    },
    num_pages() {
      this.coincontrol.numpages = Math.ceil(this.coincontrol.utxos.length / this.coincontrol.elementsPerPage);
    },
    get_rows() {
      const start = (this.coincontrol.currentPage - 1) * this.coincontrol.elementsPerPage;
      const end = start + this.coincontrol.elementsPerPage;

      this.coincontrol.getrows = this.coincontrol.utxos.slice(start, end);
    },
    change_page(page) {
      this.coincontrol.currentPage = page;
      this.get_rows();
    },
    checkboxClicked(cb, index) {
      if (cb) {
        this.coincontrol.selectedValueSats += this.coincontrol.utxos[index].satoshis;
      } else {
        this.coincontrol.selectedValueSats -= this.coincontrol.utxos[index].satoshis;
      }
      this.coincontrol.selectedValueAmount = Number(this.coincontrol.selectedValueSats * 1e-8).toFixed(8);
    },
    async fetchUtxoSet() {
      try {
        this.coincontrol.selectedValueSats = 0;
        this.coincontrol.selectedValueAmount = 0;
        this.coincontrol.errorMsg = '';
        this.coincontrol.currentPage = 1;
        this.coincontrol.selected = [];
        const explorer = this.isTestnet ? this.testnetExplorer : this.mainnetExplorer;
        const utx = await axios.get(`${explorer}/api/addr/${this.coincontrol.address}/utxo`);

        this.coincontrol.utxos = utx.data;
        this.num_pages();
        this.get_rows();
        this.coincontrol.show = true;
      } catch (e) {
        console.log(e);
        this.coincontrol.errorMsg = e.message;
      }
    },
    async buildUnsignedRawTx() {
      try {
        const network = this.isTestnet ? bitgotx.networks.fluxtestnet : bitgotx.networks.zelcash;
        const explorer = this.isTestnet ? this.testnetExplorer : this.mainnetExplorer;
        const utx = await axios.get(`${explorer}/api/addr/${this.unsignedTx.myAddress}/utxo`);
        const utxos = utx.data;
        let satoshisSoFar = 0;
        let history = [];
        const satoshisToSend = Math.round(Number(this.unsignedTx.amount) * 1e8);
        const satoshisfeesToSend = 0;
        let recipients = [{
          address: this.unsignedTx.receiver,
          satoshis: satoshisToSend,
        }];
        let count = 0;

        const selectedCoins = new Set();

        if (this.coincontrol.selectedValueSats > 0) {
          for (let j = 0; j < this.coincontrol.selected.length; j += 1) {
            if (this.coincontrol.selected[j] === true) {
              selectedCoins.add(this.coincontrol.utxos[j].txid + this.coincontrol.utxos[j].vout);
            }
          }
        }

        console.log(selectedCoins);

        for (let i = 0; i < utxos.length; i += 1) {
          if (utxos[i].height !== 0) {
            if (this.avoidFluxNodeAmounts && (+utxos[i].satoshis === 4000000000000 || +utxos[i].satoshis === 1250000000000 || +utxos[i].satoshis === 100000000000)) {
              // eslint-disable-next-line no-continue
              continue;
            }

            if (this.coincontrol.selectedValueSats > 0) {
              if (!selectedCoins.has(utxos[i].txid + utxos[i].vout)) {
                // eslint-disable-next-line no-continue
                continue;
              }
            }

            history = history.concat({
              txid: utxos[i].txid,
              vout: utxos[i].vout,
              scriptPubKey: utxos[i].scriptPubKey,
              satoshis: utxos[i].satoshis,
            });

            satoshisSoFar += utxos[i].satoshis;
            count += 1;
            if (this.sendAllFlux) {
              if (count >= 2000) {
                break;
              }
              // eslint-disable-next-line no-continue
              continue;
            } else if (satoshisSoFar >= satoshisToSend + satoshisfeesToSend) {
              break;
            }
          }
        }

        if (this.sendAllFlux) {
          // Update the recipient to the full flux amount
          // Overrides the amount that was put in the Amount to Send textbox
          recipients[0].satoshis = satoshisSoFar;

          // We don't have any change when sendAllFlux is true

          // All txs have fee 0
        } else {
          const refundSatoshis = satoshisSoFar - satoshisToSend - satoshisfeesToSend;
          if (refundSatoshis > 0) {
            recipients = recipients.concat({
              address: this.unsignedTx.myAddress,
              satoshis: refundSatoshis,
            });
          }
          if (refundSatoshis < 0) {
            this.unsignedTx.hex = 'Insufficient amount';
            return;
          }
        }
        const txb = new bitgotx.TransactionBuilder(network, satoshisfeesToSend);
        txb.setVersion(4);
        txb.setVersionGroupId(0x892F2085);
        history.forEach((x) => txb.addInput(x.txid, x.vout));
        recipients.forEach((x) => txb.addOutput(x.address, x.satoshis));
        if (this.unsignedTx.message !== '') {
          const data = Buffer.from(this.unsignedTx.message, 'utf8');
          const dataScript = bitgotx.script.nullData.output.encode(data);
          txb.addOutput(dataScript, 0);
        }

        const tx = txb.buildIncomplete();
        this.unsignedTx.hex = tx.toHex();
      } catch (e) {
        console.log(e);
        this.unsignedTx.hex = e.message;
      }
    },
    async signTransaction() {
      try {
        const network = this.isTestnet ? bitgotx.networks.fluxtestnet : bitgotx.networks.zelcash;
        const hashType = bitgotx.Transaction.SIGHASH_ALL;
        const txhex = this.signedTx.rawtx;
        const keyPair = bitgotx.ECPair.fromWIF(this.signedTx.privatekey, network);
        const txb = bitgotx.TransactionBuilder.fromTransaction(bitgotx.Transaction.fromHex(txhex, network), network);
        console.log(txb);
        let i = 0;
        // eslint-disable-next-line no-unused-vars
        for (const input of txb.inputs) {
          const hash = this.getValueHexBuffer(txb.tx.ins[i].hash.toString('hex'));
          const { index } = txb.tx.ins[i];
          console.log(txb.tx);
          console.log(hash);
          const explorer = this.isTestnet ? this.testnetExplorer : this.mainnetExplorer;
          // eslint-disable-next-line no-await-in-loop
          const tx = await axios.get(`${explorer}/api/tx/${hash}`);
          const value = Math.round(Number(tx.data.vout[index].value) * 1e8);
          txb.sign(i, keyPair, Buffer.from(this.signedTx.redeemScript, 'hex'), hashType, value);
          i += 1;
        }
        const tx = txb.buildIncomplete();
        this.signedTx.hex = tx.toHex();
      } catch (e) {
        console.log(e);
        this.signedTx.hex = e.message;
      }
    },
    finaliseTransaction() {
      try {
        const network = this.isTestnet ? bitgotx.networks.fluxtestnet : bitgotx.networks.zelcash;
        const txhex = this.finalisedTx.rawtx;
        const txb = bitgotx.TransactionBuilder.fromTransaction(bitgotx.Transaction.fromHex(txhex, network), network);
        const tx = txb.build();
        this.finalisedTx.hex = tx.toHex();
      } catch (e) {
        console.log(e);
        this.signedTx.hex = e.message;
      }
    },
    getValueHexBuffer(hex) {
      const buf = Buffer.from(hex, 'hex').reverse();
      return buf.toString('hex');
    },
  },
};
</script>

<style scoped>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}

.pubkey {
  width: 80%;
}
.pubkey {
  width: 80%;
}

.number {
  display: inline-block;
  padding: 4px 10px;
  color: #fff;
  border-radius: 4px;
  background: #44475c;
  margin: 0px 5px;
  cursor: pointer;
}

table tbody tr:nth-child(2n) td {
  background: #d4d8f9;
}

table th {
  text-transform: uppercase;
  text-align: left;
  background: #44475c;
  color: #fff;
  cursor: pointer;
  padding: 8px;
  min-width: 30px;
}

table td {
  text-align: left;
  padding: 8px;
  border-right: 2px solid #7d82a8;
}
table td:last-child {
  border-right: none;
}

table {
  font-family: "Open Sans", sans-serif;
  width: 750px;
  border-collapse: collapse;
  border: 3px solid #44475c;
  margin: 10px 10px 0 10px;
}

table.center {
  margin-left: auto;
  margin-right: auto;
}
</style>
