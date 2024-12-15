<template>
  <div class="container flex baise">
    <div class="nav column" style="text-align: center;">
      <div class="add flex mcenter">
        <button class="addButton button w100" v-on:click="popShow"> 添加币种</button>
      </div>

      <div class="coinsList flex column" v-for="(coin, index) in coins" :key="index">
        <div class="coin flex w100 flex2">
          <div class="coinName flex2" v-on:click="selectCoin(coin)">{{ coin }}</div>
          <div class="coinfullName flex2" v-on:click="selectCoin(coin)">{{ coinsfullName[index] }}</div>

          <div class="coinDelete flex1">
            <button class="coinDelete button" v-on:click="deleteCoin(index)">Delete</button>
          </div>
        </div>
      </div>
    </div>

    <div class="content flex column baise">
      <div class="header flex1"></div>
      <div class="main-content flex8 flex" style="justify-content: center; align-items: center;" :style="{ backgroundColor: backgroundColor }">
        <!-- 显示当前选择的币种价格 -->
        <div v-if="currentPrice">
          当前价格: {{ currentPrice }}
        </div>
      </div>
    </div>

    <!-- 弹出框 -->
    <div v-if="showModal" class="modal-overlay">
      <div class="modal-content flex column" style="text-align: center;">
        <h2>添加币种</h2>
        <input type="text" placeholder="请输入币种的大写字母 如 BTC" v-model="newcoin">
        <input type="text" placeholder="请输入该币种的中文名或者全程 如 sol solana 索拉纳" v-model="newcoinName">

        <button class="Addcoin button" v-on:click="addcoin">添加币种</button>
        <button class="closeButton button" v-on:click="popClose">close</button>
      </div>
    </div>
  </div>
</template>

<script>
import axios from "axios"; // 用于请求币安API

export default {
  data() {
    return {
      coins: [], // 初始为空数组
      coinsfullName: [], // 存储币种的全名
      showModal: false, // 控制弹出框是否显示
      newcoin: "", // 用来绑定输入框的值
      newcoinName: "", // 用来绑定币种中文名输入框的值
      currentCoin: '', // 当前选择的币种
      coinPrices: [], // 存储当前币种的币种价格
      currentPrice: null, // 当前选择的币种价格
      previousPrices: {}, // 存储每个币种的前一秒价格
      backgroundColor: "white", // 背景颜色，初始为白色
      priceUpdateInterval: null, // 定时器
      priceSpeakerInterval: null, // 语音播报定时器
    };
  },
  mounted() {
    // 页面加载时从 localStorage 获取 coins 数据
    const savedCoins = localStorage.getItem("coins");
    const savedCoinsname = localStorage.getItem("coinsfullName");

    if (savedCoins && savedCoinsname) {
      this.coins = JSON.parse(savedCoins); // 将 JSON 字符串转回数组
      this.coinsfullName = JSON.parse(savedCoinsname);
    } else {
      // 默认币种列表
      this.coins = ["SOL", "SUI", "BTC", "ETH", "DOGE"];
      this.coinsfullName = ["Solana", "Sui", "Bitcoin", "Ethereum", "Dogecoin"];
    }

    // 启动定时器，每秒更新当前币种的价格
    this.startPriceUpdate();
    // 启动语音播报定时器，每10秒播报一次当前选择的币种价格
    this.startPriceSpeakerUpdate();
  },
  beforeDestroy() {
    // 清除定时器
    if (this.priceUpdateInterval) {
      clearInterval(this.priceUpdateInterval);
    }
    if (this.priceSpeakerInterval) {
      clearInterval(this.priceSpeakerInterval);
    }
  },
  methods: {
    popShow() {
      this.showModal = true;
    },
    popClose() {
      this.showModal = false;
    },
    deleteCoin(index) {
      this.coins.splice(index, 1); // 删除币种
      this.coinsfullName.splice(index, 1); // 删除币种
      this.updateLocalStorage(); // 更新 localStorage
    },
    addcoin() {
      const newcoin = this.newcoin.trim(); // 获取输入的币种
      const newcoinName = this.newcoinName.trim(); // 获取输入的币种名称

      if (newcoin && newcoinName) { // 确保输入了币种及其名称
        this.coins.push(newcoin);
        this.coinsfullName.push(newcoinName);
        this.updateLocalStorage(); // 更新 localStorage
        this.newcoin = ""; // 清空输入框
        this.newcoinName = ""; // 清空输入框
        this.showModal = false;
      }
    },
    updateLocalStorage() {
      // 每次修改 coins 数组时都更新 localStorage
      localStorage.setItem("coins", JSON.stringify(this.coins));
      localStorage.setItem("coinsfullName", JSON.stringify(this.coinsfullName)); // 更新币种的全名
    },
    // 选择币种时，设置当前币种并获取价格
    selectCoin(coin) {
      this.currentCoin = coin;  // 设置当前选择的币种
      this.fetchPrice(coin);     // 获取当前币种的价格
    },
    // 获取当前币种的价格
    async fetchPrice(coin) {
      try {
        const symbol = `${coin}USDT`; // 构建币种与 USDT 的交易对
        const response = await axios.get(
          `https://api.binance.com/api/v3/ticker/price?symbol=${symbol}`
        );
        const price = parseFloat(response.data.price); // 获取价格

        // 获取当前选择币种的前一个价格
        const previousPrice = this.previousPrices[this.currentCoin];

        // 比较当前价格与之前的价格，更新背景颜色
        if (previousPrice !== undefined) {
          if (price > previousPrice) {
            this.backgroundColor = "green"; // 当前价格大于上次价格，背景为绿色
          } else if (price < previousPrice) {
            this.backgroundColor = "red"; // 当前价格小于上次价格，背景为红色
          } else {
            this.backgroundColor = "white"; // 当前价格与上次价格相同，背景为白色
          }
        }

        // 更新当前币种的价格和前一秒的价格
        this.currentPrice = price;
        this.previousPrices[this.currentCoin] = price; // 更新前一秒价格

        // console.log(`Current price of ${coin}: $${price}, Previous price: $${previousPrice}`);
      } catch (error) {
        console.error("获取币种价格失败:", error);
        this.currentPrice = "价格获取失败";
      }
    },
    // 定时更新当前选择的币种的价格
    startPriceUpdate() {
      this.priceUpdateInterval = setInterval(() => {
        if (this.currentCoin) {
          this.fetchPrice(this.currentCoin);  // 每秒更新一次当前选中的币种的价格
        }
      }, 1000); // 每秒调用一次 fetchPrice 更新价格
    },
    // 启动语音播报定时器，每10秒播报一次价格
    startPriceSpeakerUpdate() {
      this.priceSpeakerInterval = setInterval(() => {
        if (this.currentCoin && this.currentPrice) { // 确保有币种和价格
          this.speakPrice(this.currentCoin);
        }
      }, 10000); // 每10秒调用一次 speakPrice 方法
    },
    // 播报当前币种价格
    speakPrice(coin) {
      // console.log(`正在播报：${coin} 当前价格是 ${this.currentPrice}`);
      const index = this.coins.indexOf(coin);
      const fullCoinName = this.coinsfullName[index] || coin; // 获取币种全名
      const message = `${fullCoinName} 当前价格是 ${this.currentPrice} 美元`; // 播报的内容

      // 创建语音对象
      const speech = new SpeechSynthesisUtterance(message);
      speech.lang = 'zh-CN'; // 设置语言为中文

      // 确保语音引擎加载完成后获取语音列表
      if (typeof speechSynthesis !== 'undefined') {
        speechSynthesis.onvoiceschanged = () => {
          const voices = speechSynthesis.getVoices();
          speech.voice = voices.find(voice => voice.name === "Google 普通话（中国大陆）"); // 设置声音为中文普通话
        };
      }

      // 播放语音
      window.speechSynthesis.speak(speech);
      // console.log('播报中');
    }
  },
};
</script>

<style>

body {
  margin: 0;
  padding: 0;
  background-color: gray;
}

* {
  border: 1px solid rgb(0, 13, 255);
}

.flex {
  display: flex;
}

.mcenter {
  justify-content: center;
}

.column {
  flex-direction: column;
}

.container {
  height: 100vh;
  width: 100%;
}

.h100 {
  height: 100%;
}

.w100 {
  width: 100%;
}

.flex1 {
  flex: 1;
}

.flex8 {
  flex: 8;
}

.flex2 {
  flex: 2;
}

.grid {
  display: grid;
}

.nav {
  width: 15%;
  overflow-y: auto;
  height: 100vh;
}

.content {
  width: 85%;
  height: 100vh;
}

.button:hover {
  background-color: red;
}

.coin {
  width: 100%;
  display: flex;
  height: 60px;
}

.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 999;
}

.modal-content {
  background: white;
  padding: 20px;
  border-radius: 8px;
  text-align: center;
  width: 300px;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
}

.popButton {
  padding: 10px 20px;
  background-color: #007bff;
  color: white;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  font-size: 16px;
}

.popButton:hover {
  background-color: #0056b3;
}
.baise {
  background-color:white;
}
</style>
