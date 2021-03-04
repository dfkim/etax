<template>
  <v-container
    fill-height
    fluid
    grid-list-xl
  >
    <v-layout wrap>
      <v-flex
      md12
      lg12>
      <material-card>
   <form @submit.prevent="checkForm">
          <v-btn
            type="submit"
            class="mx-0 font-weight-light"
            color="success"
            :disabled="processing"
          >
            送信
          </v-btn>
 <v-text-field label="Select File"   @click="$refs.inputUpload.click()" prepend-icon='attach_file'></v-text-field>
<input v-show="false" ref="inputUpload" type="file" @change="fileChange" >
        </form>             
      <article class="media" v-for="(message, index) in reverseMessages" :key="index">
       
        <material-notification
                  class="mb-3"
                  color="info"
                  dismissible
                >
                   {{ message.from }}  <small>{{ message.time }}</small>
                </material-notification>

   
      </article></material-card>
      </v-flex>
     
      
      <v-flex
        md12
        lg12
      >
        <material-card
          class="card-tabs"
          color="green">
          <v-flex
            slot="header"
          >
            <v-tabs
              v-model="tabs"
              color="transparent"
              slider-color="white"
            >
              
              <v-tab class="mr-3">
                <v-icon class="mr-2">mdi-ballot</v-icon>
                仕訳データ
              </v-tab>
              <v-tab class="mr-3">
                <v-icon class="mr-2">mdi-receipt</v-icon>
                勘定科目
              </v-tab>
              <v-tab>
                <v-icon class="mr-2">mdi-transform</v-icon>
                家事按分
              </v-tab>
            </v-tabs>
          </v-flex>

          <v-tabs-items v-model="tabs">
            <v-tab-item
             
            >
           
          <v-data-table
            :headers="headers"
            :items="items"
            hide-actions
          >
            <template
              slot="headerCell"
              slot-scope="{ header }"
            >
              <span
                class="font-weight-light text-warning text--darken-3"
                v-text="header.text"
              />
            </template>
            <template
              slot="items"
              slot-scope="{  item }"
            >
              <td>{{ item.date }}</td>
              <td class="text-xs-right">{{ item.amount }}</td>
              <td class="text-xs-left">{{ item.summary }}</td>
              <td class="text-xs-left">{{ item.left_account }}</td>
              <td class="text-xs-left">{{ item.right_account }}</td>
            </template>
          </v-data-table>


            </v-tab-item>
             <v-tab-item
             
            >

            <v-form>
            <v-container py-0>
              <v-layout wrap>
                <v-flex
                  xs12
                  md12
                >
                 <v-textarea
            name="input-7-1"
            :value="journal"
            rows="20"
            ></v-textarea>
                </v-flex></v-layout></v-container></v-form>
         
            </v-tab-item>
             <v-tab-item
             
            >

            <v-form>
            <v-container py-0>
              <v-layout wrap>
                <v-flex
                  xs12
                  md12
                >
                <v-textarea
            name="input-7-2"
            :value="apportionment"
            ></v-textarea>
           
                </v-flex></v-layout></v-container></v-form>
            
            </v-tab-item>
          </v-tabs-items>
        </material-card>
      </v-flex>
    </v-layout>
  </v-container>
</template>

<script>
  import SockJS from "sockjs-client";
  import Stomp from "webstomp-client";

  import materialCard from '~/components/material/AppCard'
  import materialChartCard from '~/components/material/AppChartCard'
  import materialStatsCard from '~/components/material/AppStatsCard'
  
  import materialNotification from '~/components/material/AppNotification'

  export default {
    layout: 'dashboard',
    middleware: 'authentication',
    components: {
      materialCard,
      materialChartCard,
      materialStatsCard,
      materialNotification
    }
    ,data() {
      return {
        processing: false,
        apportionment: '',
        journal: '',
        messages: [],
        connected: false,
        snackbar: {
          visible: false,
          text: '',
        },
        errors: [],
        headers: [
          {
            sortable: false,
            text: '日付',
            value: 'date'
          },
          {
            sortable: false,
            text: '金額',
            value: 'amount',
            align: 'right'
          },
          {
            sortable: false,
            text: '概要',
            value: 'summary',
            align: 'left'
          },
          {
            sortable: false,
            text: '借方: 勘定科目',
            value: 'left_account',
            align: 'left'
          },
          {
            sortable: false,
            text: '貸方:  勘定科目',
            value: 'right_account',
            align: 'left'
          }
        ],
        items: [],
        tabs: 0,
        list: {
          0: false,
          1: false,
          2: false
        }
      }
    },
    computed: {
      // 配列の後ろ（新しいもの）から順に表示させたいので反転させる
      reverseMessages: function() {
        return this.messages.slice().reverse()
      },
    },
    
    methods: {
      pickFile() {
      this.$refs.fileUp.click()
      },
      startProcessing: function () {
        this.processing = true
      },
      endProcessing: function () {
        this.processing = false

      },
      isProcessing: function () {
        return this.processing
      },
      connect() {
        this.socket = new SockJS("http://localhost:8080/gs-guide-websocket");
        this.stompClient = Stomp.over(this.socket);
        this.stompClient.connect(
          {},
          frame => {
            this.connected = true;
            console.log(frame);
            this.stompClient.subscribe("/topic/greetings", tick => {
              
              let tickObj = JSON.parse(tick.body);
              console.log(tickObj);
              if(tickObj.from == "すべての処理が終了しました。" || tickObj.from == "処理が失敗しました。"){
                this.endProcessing();
              }
              this.messages.push(tickObj);
            });
          },
          error => {
            console.log(error);
            this.connected = false;
          }
        );
      },
      disconnect() {
        if (this.stompClient) {
          this.stompClient.disconnect();
        }
        this.connected = false;
      },
      tickleConnection() {
        this.connected ? this.disconnect() : this.connect();
      },
      complete (index) {
        this.list[index] = !this.list[index]
      },
      checkForm:function(e) {
        this.startProcessing();
        this.messages = [];
        console.log("aaaa")
        e.preventDefault();
        
        this.errors = [];
        if(this.items.length==0) this.errors.push("please upload file");
        let options = { emulateJSON: true, emulateHTTP: true }
        var data = {
          fname: JSON.stringify(this.items),
          skipSettlement : false,
          showMonthlyTotal : false,
          isSoloProprietorship : true,
          apportionment : this.apportionment,
          journal : this.journal
        }
        this.$axios.post('http://localhost:8080/entry', data, options)
        .then(function(){
            console.log('Saved!');
        });
      },

      fileChange: function(e) {
        alert(0)
        this.connect();
        console.log("====change====")
        const file = e.target.files[0];
        const reader = new FileReader();
        const workers = [];

        const loadFunc = () => {
          console.log("loadFunc")
          const lines = reader.result.split("\n");
          
          lines.forEach(element => {
            const workerData = element.split(",");
            if (workerData.length != 5) return;
            const worker = {
              date: workerData[0],
              amount: workerData[1],
              summary: workerData[2],
              left_account: workerData[3],
              right_account: workerData[4]
            };
            workers.push(worker);
          });
          this.items = workers;
         
        };

        // onloadはreadAsBinaryStringでファイルを読み込んだ後に実行されます.
        reader.onload = loadFunc;

        reader.readAsText(file);
      }
    },
    async asyncData({ $axios }){},
    fetchOnServer: false,
    mounted() {
      let apportionmentData = ['- {勘定科目: 水道光熱費, 事業割合: 70.0}',
'- {勘定科目: 地代家賃,   事業割合: 30.0}',
'- {勘定科目: 通信費,     事業割合: 80.0}',
'- {勘定科目: 利子割引料,     事業割合: 30.0}'
].join("\n");

      this.apportionment = apportionmentData;
      let journalData = ['仕訳:',
'  資産: [現金, 当座預金, 定期預金, その他の預金,',
'   普通預金,',
'    受取手形, 売掛金, 有価証券,',
'    前払金, 貸付金,',
'    商品,',
'    建物, 建物附属設備, 機械装置, 車両運搬具, 工具器具備品, 土地,',
'    事業主貸,',
'    開業費, 仮払金,',
'    仮払消費税等, 未収消費税等]',
'  負債: [支払手形, 買掛金, 借入金, 未払金, 前受金, 預り金,',
'    貸倒引当金,',
'    仮受消費税等, 未払消費税等]',
'  資本: [事業主借, 元入金, 事業主勘定]',
'  収益: [売上, 家事消費等, 雑収入]',
'  費用: [仕入,',
'    租税公課, 荷造運賃, 水道光熱費, 旅費交通費, 通信費,',
'    広告宣伝費, 接待交際費, 交際費, 損害保険料, 修繕費, 消耗品費, 減価償却費,',
'    福利厚生費, 給料賃金, 外注工賃, 利子割引料, 地代家賃, 貸倒金,',
'    支払手数料, 会議費, 繰延資産償却費, 開業費償却,',
'    雑費, 車両費, パソコン, 事務所賃料,',
'    期首商品棚卸高, 期末商品棚卸高]',
'',
'損益計算書:',
'  青色申告特別控除前の所得金額:',
'    売上金額（雑収入を含む）: [売上, 家事消費等, 雑収入]',
'    売上原価:',
'    期首商品棚卸高: [期首商品棚卸高]',
'    仕入金額: [仕入]',
'    期末商品棚卸高: [期末商品棚卸高]',
'    経費:',
'    租税公課: [租税公課]',
'    荷造運賃: [荷造運賃]',
'    水道光熱費: [水道光熱費]',
'    旅費交通費: [旅費交通費]',
'    通信費: [通信費]',
'    広告宣伝費: [広告宣伝費]',
'    接待交際費: [接待交際費]',
'    交際費: [交際費]',
'    損害保険料: [損害保険料]',
'    修繕費: [修繕費]',
'    消耗品費: [消耗品費]',
'    減価償却費: [減価償却費]',
'    福利厚生費: [福利厚生費]',
'    給料賃金: [給料賃金]',
'    外注工賃: [外注工賃]',
'    利子割引料: [利子割引料]',
'    地代家賃: [地代家賃]',
'    貸倒金: [貸倒金]',
'    繰延資産償却費: [繰延資産償却費, 開業費償却]',
'    支払手数料: [支払手数料]',
'    会議費: [会議費]',
'    車両費: [車両費]',
'    パソコン: [パソコン]',
'    雑費: [雑費]',
'',
'損益計算書の表示制御:',
'  符号を反転して表示する見出し: [期末商品棚卸高]',
'  常に表示する見出し: [売上金額（雑収入を含む）,',
'    売上原価, 期首商品棚卸高, 仕入金額, 期末商品棚卸高,',
'    経費, 租税公課, 荷造運賃, 水道光熱費, 旅費交通費, 通信費, 広告宣伝費, 接待交際費, 交際費, 損害保険料, 修繕費,',
'    消耗品費, 減価償却費, 福利厚生費, 給料賃金, 外注工賃, 利子割引料, 地代家賃, 貸倒金,',
'    雑費]',
'  ゼロなら表示しない見出し: []',
'',
'貸借対照表:',
'  資産:',
'    現金: [現金]',
'    当座預金: [当座預金]',
'    定期預金: [定期預金]',
'    その他の預金: [その他の預金, 普通預金]',
'    受取手形: [受取手形]',
'    売掛金: [売掛金]',
'    有価証券: [有価証券]',
'    棚卸資産: [商品]',
'    前払金: [前払金]',
'    貸付金: [貸付金]',
'    建物: [建物]',
'    建物附属設備: [建物附属設備]',
'    機械装置: [機械装置]',
'    車両運搬具: [車両運搬具]',
'    工具器具備品: [工具器具備品]',
'    土地: [土地]',
'    仮払消費税等: [仮払消費税等]',
'    未収消費税等: [未収消費税等]',
'    繰延資産: [開業費]',
'    事業主貸: [事業主貸]',
'  負債:',
'    支払手形: [支払手形]',
'    買掛金: [買掛金]',
'    借入金: [借入金]',
'    未払金: [未払金]',
'    前受金: [前受金]',
'    預り金: [預り金]',
'    仮受消費税等: [仮受消費税等]',
'    未払消費税等: [未払消費税等]',
'    貸倒引当金: [貸倒引当金]',
'  資本:',
'    事業主借: [事業主借]',
'    事業主勘定: [事業主勘定]',
'    元入金: [元入金]',
'    控除前の所得金額: [控除前の所得金額]',
'',
'貸借対照表の表示制御: ',
'  符号を反転して表示する見出し: []',
'  常に表示する見出し: [現金, 当座預金, 定期預金, その他の預金, 受取手形, 売掛金, 有価証券, 棚卸資産,',
'    前払金, 貸付金, 建物, 建物附属設備, 機械装置, 車両運搬具, 工具器具備品, 土地, 事業主貸,',
'    支払手形, 買掛金, 借入金, 未払金, 前受金, 預り金, 貸倒引当金,',
'    事業主借, 事業主勘定, 元入金, 控除前の所得金額]',
'  ゼロなら表示しない見出し: [仮払消費税等, 仮受消費税等]'
].join("\n");
      this.journal = journalData;
      
    }
  }
</script>
