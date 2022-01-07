<template>
  <div id="root">
    <header>
      <Publicity v-show="!running" />
       <span>
      <el-button type="text" @click="download(resultList)">
        导出中奖名单
      </el-button>
      </span>
      <span style="margin-left: 120px;">
      <el-button type="text"  @click="downloadRemain(list,resultList)">
       导出未中奖名单
      </el-button>
      </span>
       <span style="margin-left: 120px;">
      <el-button type="text" @click="downloadAll(list)">
       导出全体名单
      </el-button>
      </span>
      <el-button class="res" type="text" @click="showResult = true">
        抽奖结果
      </el-button>
      <el-button class="con" type="text" @click="showConfig = true">
        抽奖配置
      </el-button>
    </header>
    <div id="main" :class="{ mask: showRes }"></div>
    <div id="tags">
      <ul v-for="item in datas" :key="item.key">
        <li>
          <a
            href="javascript:void(0);"
            :style="{
              color: '#fff'
            }"
          >
            {{ item.name ? item.name : item.key }}
            <img v-if="item.photo" :src="item.photo" :width="50" :height="50" />
          </a>
        </li>
      </ul>
    </div>
    <transition name="bounce">
      <div id="resbox" v-show="showRes">
        <p @click="showRes = false">{{ categoryName }}抽奖结果：</p>
        <div class="container">
          <span
            v-for="item in resArr"
            :key="item.key"
            class="itemres"
            :style="resCardStyle"
            :data-id="item"
            @click="showRes = false"
            :class="{
              numberOver:
                !!photos.find(d => d.id === item) ||
                !!list.find(d => d.key === item),
            }"
          >
            <span class="cont" v-if="!photos.find(d => d.id === item)">
              <span
                v-if="!!list.find(d => d.key === item)"
                :style="{
                  fontSize: '40px'
                }"
              >
                <div>{{ list.find(d => d.key === item).key }}</div>
                <div>{{ list.find(d => d.key === item).name }}</div>
                <div>{{ list.find(d => d.key === item).department }}</div>
              </span>
              <span class="else" v-else>
                <div>{{ item.key }}</div>
                <div>{{ item.name }}</div>
                <div>{{ item.department }}</div>
              </span>
            </span>
            <img
              v-if="photos.find(d => d.id === item)"
              :src="photos.find(d => d.id === item).value"
              alt="photo"
              :width="160"
              :height="160"
            />
          </span>
        </div>
      </div>
    </transition>

    <el-button
      class="audio"
      type="text"
      @click="
        () => {
          playAudio(!audioPlaying);
        }
      "
    >
      <i
        class="iconfont"
        :class="[audioPlaying ? 'iconstop' : 'iconplay1']"
      ></i>
    </el-button>

    <LotteryConfig :visible.sync="showConfig" @resetconfig="reloadTagCanvas" />
    <Tool
      @toggle="toggle"
      @resetConfig="reloadTagCanvas"
      @getPhoto="getPhoto"
      :running="running"
      :closeRes="closeRes"
    />
    <Result :visible.sync="showResult"></Result>

    <span class="copy-right"> 勿以“善”小而不为！ </span>
    <audio
      id="audiobg"
      preload="auto"
      controls
      autoplay
      loop
      @play="playHandler"
      @pause="pauseHandler"
    >
      <source :src="audioSrc" />
      你的浏览器不支持audio标签
    </audio>
  </div>
</template>
<script>
import LotteryConfig from '@/components/LotteryConfig';
import Publicity from '@/components/Publicity';
import Tool from '@/components/Tool';
import bgaudio from '@/assets/bg.mp3';
import beginaudio from '@/assets/begin.mp3';
import {
  getData,
  configField,
  resultField,
  newLotteryField,
  conversionCategoryName,
  listField
} from '@/helper/index';
import { luckydrawHandler } from '@/helper/algorithm';
import Result from '@/components/Result';
import { database, DB_STORE_NAME } from '@/helper/db';
import json2csv from 'json2csv';

export default {
  name: 'App',

  components: { LotteryConfig, Publicity, Tool, Result },

  computed: {
    resCardStyle() {
      const style = { fontSize: '30px' };
      // const { number } = this.config;
      // if (number < 100) {
      //   style.fontSize = '100px';
      // } else if (number < 1000) {
      //   style.fontSize = '80px';
      // } else if (number < 10000) {
      //   style.fontSize = '60px';
      // }
      return style;
    },
    config: {
      get() {
        return this.$store.state.config;
      }
    },
    result: {
      get() {
        return this.$store.state.result;
      },
      set(val) {
        this.$store.commit('setResult', val);
      }
    },
    list() {
      return this.$store.state.list;
    },
    allresult() {
      let allresult = [];
      for (const key in this.result) {
        if (this.result.hasOwnProperty(key)) {
          const element = this.result[key];
          allresult = allresult.concat(element);
        }
      }
      return allresult;
    },
    datas() {
      const { number } = this.config;
      const nums = number >= 1500 ? 500 : this.config.number;
      const configNum = number > 1500 ? Math.floor(number / 3) : number;
      const randomShowNums = luckydrawHandler(configNum, [], nums);
      const randomShowDatas = randomShowNums.map((item) => {
        const listData = this.list[item - 1];
        const photo = this.photos.find((d) => d.id === item);
        return {
          key: item * (number > 1500 ? 3 : 1),
          name: listData ? listData.name : '',
          photo: photo ? photo.value : ''
        };
      });
      return randomShowDatas;
    },
    categoryName() {
      return conversionCategoryName(this.category);
    },
    photos() {
      return this.$store.state.photos;
    },
   resultList() {
      const list = [];
      for (const key in this.result) {
        if (this.result.hasOwnProperty(key)) {
          const element = this.result[key];
          let name = conversionCategoryName(key);
          list.push({
            label: key,
            name,
            value: element
          });
        }
      }
      return list;
    }
  },
  created() {
    const data = getData(configField);
    if (data) {
      this.$store.commit('setConfig', Object.assign({}, data));
    }
    const result = getData(resultField);
    if (result) {
      this.$store.commit('setResult', result);
    }

    const newLottery = getData(newLotteryField);
    if (newLottery) {
      const config = this.config;
      newLottery.forEach(item => {
        this.$store.commit('setNewLottery', item);
        if (!config[item.key]) {
          this.$set(config, item.key, 0);
        }
      });
      this.$store.commit('setConfig', config);
    }

    const list = getData(listField);
    if (list) {
      this.$store.commit('setList', list);
    }
  },
  data() {
    return {
      running: false,
      showRes: false,
      showConfig: false,
      showResult: false,
      resArr: [],
      category: '',
      audioPlaying: false,
      audioSrc: bgaudio,
      jsondatas:[{"key":"3","name":"小张","department":"FATP"},{"key":"4","name":"小赵","department":"FATP"},{"key":"5","name":"小周","department":"FATP"}],
      fields: ['key','name','department']
    };
  },
  watch: {
    photos: {
      deep: true,
      handler() {
        this.$nextTick(() => {
          this.reloadTagCanvas();
        });
      }
    }
  },
  mounted() {
    this.startTagCanvas();
    setTimeout(() => {
      this.getPhoto();
    }, 1000);
    window.addEventListener('resize', this.reportWindowSize);
  },
  beforeDestroy() {
    window.removeEventListener('resize', this.reportWindowSize);
  },
  methods: {
    reportWindowSize() {
      const AppCanvas = this.$el.querySelector('#rootcanvas');
      if (AppCanvas.parentElement) {
        AppCanvas.parentElement.removeChild(AppCanvas);
      }
      this.startTagCanvas();
    },
    playHandler() {
      this.audioPlaying = true;
    },
    pauseHandler() {
      this.audioPlaying = false;
    },
    playAudio(type) {
      if (type) {
        this.$el.querySelector('#audiobg').play();
      } else {
        this.$el.querySelector('#audiobg').pause();
      }
    },
    loadAudio() {
      this.$el.querySelector('#audiobg').load();
      this.$nextTick(() => {
        this.$el.querySelector('#audiobg').play();
      });
    },
    getPhoto() {
      database.getAll(DB_STORE_NAME).then(res => {
        if (res && res.length > 0) {
          this.$store.commit('setPhotos', res);
        }
      });
    },
    speed() {
      return [0.1 * Math.random() + 0.01, -(0.1 * Math.random() + 0.01)];
    },
    createCanvas() {
      const canvas = document.createElement('canvas');
      canvas.width = document.body.offsetWidth;
      canvas.height = document.body.offsetHeight;
      canvas.id = 'rootcanvas';
      this.$el.querySelector('#main').appendChild(canvas);
    },
    startTagCanvas() {
      this.createCanvas();
      const { speed } = this;
      window.TagCanvas.Start('rootcanvas', 'tags', {
        textColour: null,
        initial: speed(),
        dragControl: 1,
        textHeight: 20,
        noSelect: true,
        lock: 'xy'
      });
    },
    reloadTagCanvas() {
      window.TagCanvas.Reload('rootcanvas');
    },
    closeRes() {
      this.showRes = false;
    },
    toggle(form) {
      const { speed, config } = this;
      if (this.running) {
        this.audioSrc = bgaudio;
        this.loadAudio();

        window.TagCanvas.SetSpeed('rootcanvas', speed());
        this.showRes = true;
        this.running = !this.running;
        this.$nextTick(() => {
          this.reloadTagCanvas();
        });
      } else {
        this.showRes = false;
        if (!form) {
          return;
        }

        this.audioSrc = beginaudio;
        this.loadAudio();

        const { number } = config;
        const { category, mode, qty, remain, allin } = form;
        let num = 1;
        if (mode === 1 || mode === 5) {
          num = mode;
        } else if (mode === 0) {
          num = remain;
        } else if (mode === 99) {
          num = qty;
        }
        let inde = [];
        this.list.forEach((value, index) => {
          if (this.allresult.includes(value)) {
            inde.push(index + 1);
          }
        });
        const indexArr = luckydrawHandler(number, allin ? [] : inde, num);
        let resultArr = [];
        const _this = this;
        indexArr.forEach(index => {
          resultArr.push(_this.list[index - 1]);
        });
        this.resArr = resultArr;

        this.category = category;
        if (!this.result[category]) {
          this.$set(this.result, category, []);
        }
        const oldRes = this.result[category] || [];
        const data = Object.assign({}, this.result, {
          [category]: oldRes.concat(resultArr)
        });
        this.result = data;
        window.TagCanvas.SetSpeed('rootcanvas', [5, 1]);
        this.running = !this.running;
      }
    },
    download(UserList) {
      //datas：数据，fields：字段名
      try {
        console.log(JSON.stringify(UserList));
        console.log(JSON.stringify(UserList[0].value));
        if(UserList.length == 0 || (UserList.length == 1 && Array.prototype.isPrototypeOf(UserList[0].value) && UserList[0].value.length === 0)){
          alert("请先抽奖！");
        }else{
        //抓取中奖纪录名单，传进去
        for(var k =0 ;k<UserList.length;k++){
          var prizeName=UserList[k].name;
          var prizeResult= UserList[k].value;
          console.log("奖品名称："+prizeName);
          console.log("名单："+JSON.stringify(prizeResult));
          //传进去的为object类型的数据
          const result = json2csv.parse(prizeResult, {
          fields: this.fields
        });
        // 判断浏览器类型
        if ((navigator.userAgent.indexOf('compatible') > -1 &&
           navigator.userAgent.indexOf('MSIE') > -1) ||
          navigator.userAgent.indexOf('Edge') > -1 ) {
          //IE10或Edge浏览器
          var BOM = "\uFEFF";
          var csvData = new Blob([BOM + result], { type: "text/csv" });
          navigator.msSaveBlob(csvData, `test.csv`);
        } else {
          //非IE浏览器
          var csvContent = "data:text/csv;charset=utf-8,\uFEFF" + result;
          //使用a标签的download属性实现下载功能
          var link = document.createElement("a");
          link.href = encodeURI(csvContent);

          var d = new Date();
          var h = d.getHours();
          var mi = d.getMinutes();
          var s = d.getSeconds();
          console.log("当前时间："+[h,mi,s].join('-'));
        var fileName = prizeName + "中奖名单_" + [h,mi,s].join('-');
        link.download = fileName + `.csv`;
        document.body.appendChild(link);
        link.click();
          document.body.removeChild(link);
        }
        }
        }
      } catch (err) {
        alert(err);
      }
    },
    downloadAll(AllUserList) {
      //datas：数据，fields：字段名
      try {
        console.log(JSON.stringify());
        console.log("全体名单："+ JSON.stringify(AllUserList));
        if(AllUserList.length == 0 ){
          alert("请先导入参与人员名单！");
        }else{
          const result = json2csv.parse(AllUserList, {
          fields: this.fields
        });
       
        // 判断浏览器类型
        if ((navigator.userAgent.indexOf('compatible') > -1 &&
           navigator.userAgent.indexOf('MSIE') > -1) ||
          navigator.userAgent.indexOf('Edge') > -1 ) {
          //IE10或Edge浏览器
          var BOM = "\uFEFF";
          var csvData = new Blob([BOM + result], { type: "text/csv" });
          navigator.msSaveBlob(csvData, `test.csv`);
        } else {
          //非IE浏览器
          var csvContent = "data:text/csv;charset=utf-8,\uFEFF" + result;
          //使用a标签的download属性实现下载功能
          var link = document.createElement("a");
          link.href = encodeURI(csvContent);
          var fileName = "全体参与名单";
          link.download = fileName + `.csv`;
          document.body.appendChild(link);
          link.click();
          document.body.removeChild(link);
        }

        }
      } catch (err) {
        alert(err);
      }
    },
    downloadRemain(AllUserList, UserList) {
      try {
        var remainUserList=AllUserList;
        for(var n=0;n<UserList.length;n++){
          var prizeResult= UserList[n].value;
          console.log(UserList[n].name +JSON.stringify(prizeResult));
        for (let i = remainUserList.length - 1; i >= 0; i--) {
		for (let j = 0; j < prizeResult.length; j++) {
			if (remainUserList[i].key === prizeResult[j].key) {
	            remainUserList.splice(i, 1);
				break;
			}
		}
	}
     }
        // remainUserList =AllUserList;
        console.log("未中奖名单：" + JSON.stringify(remainUserList));
        if(remainUserList.length == 0 ){
          alert("请先导入参与人员名单！");
        }else{
          const result = json2csv.parse(remainUserList, {
          fields: this.fields
        });
       
        // 判断浏览器类型
        if ((navigator.userAgent.indexOf('compatible') > -1 &&
           navigator.userAgent.indexOf('MSIE') > -1) ||
          navigator.userAgent.indexOf('Edge') > -1 ) {
          //IE10或Edge浏览器
          var BOM = "\uFEFF";
          var csvData = new Blob([BOM + result], { type: "text/csv" });
          navigator.msSaveBlob(csvData, `test.csv`);
        } else {
          //非IE浏览器
          var csvContent = "data:text/csv;charset=utf-8,\uFEFF" + result;
          //使用a标签的download属性实现下载功能
          var link = document.createElement("a");
          link.href = encodeURI(csvContent);
          var fileName = "未中奖名单";
          link.download = fileName + `.csv`;
          document.body.appendChild(link);
          link.click();
          document.body.removeChild(link);
        }

        }
      } catch (err) {
        alert(err);
      }
    }
  }
};
</script>
<style lang="scss">
#root {
  height: 100%;
  position: relative;
  background-image: url('./assets/bg1.jpg');
  background-size: 100% 100%;
  background-position: center center;
  background-repeat: no-repeat;
  background-color: #121936;
  .mask {
    -webkit-filter: blur(5px);
    filter: blur(5px);
  }
  header {
    height: 50px;
    line-height: 50px;
    position: relative;
    .el-button {
      position: absolute;
      top: 17px;
      padding: 0;
      z-index: 9999;
      &.con {
        right: 20px;
      }
      &.res {
        right: 100px;
      }
    }
  }
  .audio {
    position: absolute;
    top: 100px;
    right: 30px;
    width: 40px;
    height: 40px;
    line-height: 40px;
    border: 1px solid #fff;
    border-radius: 50%;
    padding: 0;
    text-align: center;
    .iconfont {
      position: relative;
      left: 1px;
    }
  }
  .copy-right {
    position: absolute;
    right: 0;
    bottom: 0;
    color: #ccc;
    font-size: 30px;
  }
  .bounce-enter-active {
    animation: bounce-in 1.5s;
  }
  .bounce-leave-active {
    animation: bounce-in 0s reverse;
  }
}
#main {
  height: 100%;
}

#resbox {
  position: absolute;
  top: 50%;
  left: 50%;
  width: 1280px;
  transform: translateX(-50%) translateY(-50%);
  text-align: center;
  p {
    color: rgb(233, 236, 11);
    font-size: 50px;
    line-height: 120px;
  }
  .container {
    display: flex;
    justify-content: center;
    flex-wrap: wrap;
  }
  .itemres {
    background: #fff;
    width: 160px;
    height: 160px;
    border-radius: 4px;
    border: 1px solid #ccc;
    font-weight: bold;
    margin-right: 20px;
    margin-bottom: 20px;
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    position: relative;
    .cont {
      color: rgb(0, 87, 183);
      display: flex;
      justify-content: center;
      align-items: center;
    }
    &.numberOver::before {
      content: attr(data-id);
      width: 50px;
      height: 30px;
      line-height: 22px;
      background-color: #fff;
      position: absolute;
      bottom: 0;
      left: 0;
      font-size: 14px;
      // border-radius: 50%;
      z-index: 1;
    }
  }
}
</style>
