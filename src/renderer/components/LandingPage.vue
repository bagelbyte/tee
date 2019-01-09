<template>
  <div id="wrapper">
    <main>
        <ul class="list-group">
            <li v-for="(each, key) in lastPrice" :key="key" class="list-group-item" :id="key">{{key}}: {{each}}</li>
        </ul>
        <div class="row">
            <div class="column">
                <div class="input-group mb-3">
                    <div class="input-group-prepend">
                        <label class="input-group-text" for="inputGroupSelect01">Alarm (with sound)</label>
                    </div>
                    <select v-model="currentAlarmSound">
                        <option value="hero">hero</option>
                        <option value="basso">basso</option>
                        <option value="blow">blow</option>
                        <option value="bottle">bottle</option>
                        <option value="frog">frog</option>
                        <option value="funk">funk</option>
                        <option value="glass">glass</option>
                        <option value="morse">morse</option>
                        <option value="ping">ping</option>
                        <option value="pop">pop</option>
                        <option value="purr">purr</option>
                        <option value="sosumi">sosumi</option>
                        <option value="submarine">submarine</option>
                        <option value="tink">tink</option>
                    </select>
                    <div class="input-group-prepend">
                        <label class="input-group-text" for="inputGroupSelect01">when</label>
                    </div>
                    <select class="custom-select" v-model="currentAlarmPair">
                        <option v-for="(each, key) in lastPrice" :key="key">{{key}}</option>
                    </select>
                    <div class="input-group-prepend">
                        <label class="input-group-text" for="inputGroupSelect01">goes</label>
                    </div>
                    <select class="custom-select" v-model="currentAlarmWhen">
                        <option value=">">above</option>
                        <option value="<">below</option>
                        <option value="==">equals</option>
                    </select>
                    <input v-model="currentAlarmPrice" type="text" class="form-control">
                </div>
            </div>
        </div>


        <button @click="newAlarm" class="btn btn-success">Create</button>

        <h4>pending alerts</h4>
        <ul class="list-group">
            <li v-for="(each, key) in alarms" :key="key" class="list-group-item list-group-item-primary">
                {{each.message}} <button @click="removeAlarm(each)" class="btn btn-danger">Delete</button>
            </li>
        </ul>

        <h4>triggered alerts</h4>
        <ul class="list-group">
            <li v-for="(each, key) in triggeredAlarms" :key="key" class="list-group-item list-group-item-danger">
                {{each.message}} <button @click="removeTriggeredAlarm(each)" class="btn btn-danger">Delete</button>
            </li>
        </ul>

    </main>
  </div>
</template>

<script>
  import SystemInformation from './LandingPage/SystemInformation'
  import $ from 'jquery'
  var Tone = require('tone')

  export default {
    name: 'landing-page',
    created(){
        //open up the websocket and whatnot
        // Create WebSocket connection.
        const socket = new WebSocket('wss://ws-feed.pro.coinbase.com');

        // Connection opened
        socket.addEventListener('open', function (event) {
          socket.send(JSON.stringify({
            "type": "subscribe",
            "channels": ["ticker"],
            "product_ids": [
                'BCH-USD',
                "BTC-USD",
                "ZRX-USD",
                "ETH-USD",
                "LTC-USD",
                "ETC-USD"
            ]
        }));
        });

        // Listen for messages
        socket.addEventListener('message', (event) => {
            var eventobj = JSON.parse(event.data)
            if(eventobj.type == 'ticker'){
                if(typeof(this.lastPrice) !== 'undefined'){
                    if(parseFloat(this.lastPrice[eventobj.product_id]) > parseFloat(eventobj.price)){
                        this.tickDown(eventobj.product_id)
                    }
                    else{
                        this.tickUp(eventobj.product_id)
                    }
                }
                //align it with the product id in the last price object
                this.$set(this.lastPrice, eventobj.product_id, eventobj.price)
                this.evalAlarms(eventobj.product_id, eventobj.price)

            }
        });
    },
    mounted(){
        $("body").on(
            "animationend",
            function(eve) {
                $(eve.target).removeClass("tickerUp");
                $(eve.target).removeClass("tickerDown");
            }
        );
    },
    data(){
        return {
            lastPrice: {},
            currentAlarmPair: null,
            currentAlarmWhen: null,
            currentAlarmPrice: null,
            currentAlarmSound: null,
            alarms: [],
            triggeredAlarms: [],
            sounding: false,
            sound: null,
            soundInterval: null
        }
    },
    components: { SystemInformation },
    methods: {
      open (link) {
            this.$electron.shell.openExternal(link)
        },
        newAlarm(){
            //oh body
            var tmp = function(currentAlarmPair, currentAlarmWhen, currentAlarmPrice){
                return function(pair, newPrice){
                    if(pair == currentAlarmPair){
                        if(eval(newPrice + ' ' + currentAlarmWhen + ' ' + currentAlarmPrice)){
                            return true
                        }
                    }
                    return false
                }
            }(this.currentAlarmPair, this.currentAlarmWhen, this.currentAlarmPrice)

            let message = 'when ' + this.currentAlarmPair + ' ' + this.currentAlarmWhen + ' ' + this.currentAlarmPrice
            this.alarms.push({'message': message, 'func': tmp, 'sound': this.currentAlarmSound})
        },
        evalAlarms(pair, newPrice){
            var tmpAlarms = []
            for(var each in this.alarms){
                var alarm = this.alarms[each]

                if(!alarm.func(pair, newPrice)){
                    //put the alarm back in there
                    tmpAlarms.push(alarm)
                }
                else{
                    this.soundInterval = setInterval(() => {
                        //create a synth and connect it to the master output (your speakers)
                        var synth = new Tone.Synth().toMaster();

                        //play a middle 'C' for the duration of an 8th note
                        synth.triggerAttackRelease("C4", "8n");
                    }, 1000)
                    this.triggeredAlarms.push(alarm)
                }
            }

            this.alarms = tmpAlarms
        },
        removeAlarm(alarm){
            for(var each in this.alarms){
                if(this.alarms[each] == alarm){
                    this.alarms.splice(each, 1)
                }
            }
        },
        removeTriggeredAlarm(alarm){
            //just gonna get rid of the sound thats happening right now

            clearInterval(this.soundInterval)

            for(var each in this.triggeredAlarms){
                if(this.triggeredAlarms[each] == alarm){
                    this.triggeredAlarms.splice(each, 1)
                }
            }
        },
        tickUp(pair){
            $('#' + pair).removeClass("tickerUp");
            $('#' + pair).removeClass("tickerDown");
            $('#' + pair).addClass('tickerUp')
        },
        tickDown(pair){
            $('#' + pair).removeClass("tickerUp");
            $('#' + pair).removeClass("tickerDown");
            $('#' + pair).addClass('tickerDown')
        }
    }
  }
</script>

<style>
  @import url('https://fonts.googleapis.com/css?family=Source+Sans+Pro');

  * {
    box-sizing: border-box;
    margin: 0;
    padding: 0;
  }

  body { font-family: 'Source Sans Pro', sans-serif; }

  #wrapper {
    background:
      radial-gradient(
        ellipse at top left,
        rgba(255, 255, 255, 1) 40%,
        rgba(229, 229, 229, .9) 100%
      );
    height: 100vh;
    padding: 60px 80px;
    width: 100vw;
  }

  #logo {
    height: auto;
    margin-bottom: 20px;
    width: 420px;
  }

  main {
    display: flex;
    justify-content: space-between;
  }

  main > div { flex-basis: 50%; }

  .left-side {
    display: flex;
    flex-direction: column;
  }

  .welcome {
    color: #555;
    font-size: 23px;
    margin-bottom: 10px;
  }

  .title {
    color: #2c3e50;
    font-size: 20px;
    font-weight: bold;
    margin-bottom: 6px;
  }

  .title.alt {
    font-size: 18px;
    margin-bottom: 10px;
  }

  .doc p {
    color: black;
    margin-bottom: 10px;
  }

  .doc button {
    font-size: .8em;
    cursor: pointer;
    outline: none;
    padding: 0.75em 2em;
    border-radius: 2em;
    display: inline-block;
    color: #fff;
    background-color: #4fc08d;
    transition: all 0.15s ease;
    box-sizing: border-box;
    border: 1px solid #4fc08d;
  }

  .doc button.alt {
    color: #42b983;
    background-color: transparent;
  }

  .tickerUp {
      -webkit-animation: color_change 1s;
	-moz-animation: color_change 1s;
	-ms-animation: color_change 1s;
	-o-animation: color_change 1s;
    	animation: color_change 1s;
    }
    @keyframes color_change {
    	from { background-color: green; }
    	to { background-color: white; }
    }

    .tickerDown {
        -webkit-animation: color_change_bad 1s;
  	-moz-animation: color_change_bad 1s;
  	-ms-animation: color_change_bad 1s;
  	-o-animation: color_change_bad 1s;
      	animation: color_change_bad 1s;
      }
      @keyframes color_change_bad {
      	from { background-color: red; }
      	to { background-color: white; }
      }
</style>
