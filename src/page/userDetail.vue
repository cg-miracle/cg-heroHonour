<template>
  <div class="container-wrap">
    <section id='user-detail' class='main'>
      <div class='user-info'>
        <i class='back-btn el-icon-arrow-left' @click='backfunc'></i>
        <img class='user-avatar' :src="userInfo.avatarmedium" alt="头像">
        <p class='user-name'>{{userInfo.personaname}}</p>
        <p class='dota-id'>ID: {{getDotaid(userInfo.steamid)}}</p>
      </div>
      <div>
        <section class='summaries'>
          <h2><i class="iconfont icon-iconfonthuangguan huangguan"></i>摘要</h2>
          <div class='summaries-content'>
            <div class='summaries-item'><p>2087</p><p>总场数</p></div>
            <div class='summaries-item'><p>3.20</p><p>KDA</p></div>
            <div class='summaries-item'><p>49.5%</p><p>总胜率</p></div>
            <div class='summaries-item'><p>3000</p><p>天梯匹配MMR</p></div>
          </div>
        </section>
        <section class='recent-match'>
          <h2><i class="iconfont icon-iconfonthuangguan huangguan"></i>最近比赛</h2>
          <section class='match-table'>
            <div class='table-header'>
                <span>英雄</span>
                <span>结果</span>
                <span>level</span>
                <span>结束时间</span>
                <span>KDA</span>
            </div>
            <loading v-if="!isLoad"></loading>
            <div class='table-body' v-else>
              <div class='cell' v-for='match in tableData'>
                <router-link class="cell-link" :to="{name:'matchDetail',params:{mid:match.match_id}}">
                  <span class="item"><img :src="match.hero" class="hero_avatar"></span>
                  <span class="item"><span class="resultTag" :class="match.result === '胜'?'winColor':'failColor'">{{match.result}}</span></span>
                  <span class="item col-C7CBCF">{{match.level}}</span>
                  <span class="item end-time">{{match.end_at}}</span>
                  <span class="item">{{match.kda}}<p class='k_d_a col-C7CBCF'>{{match.kdaInfo}}</p></span>
                </router-link>
              </div>
            </div>
          </section>
        </section>
      </div>
    </section>
    <tb type='好友'></tb>
  </div>
</template>

<script>
  import hd from '../components/header.vue'
  import tb from '../components/toolbar.vue'
  import api from '../http/apis'
  import util from '../lib/utils'
  import loading from '../components/loading.vue'
  import defaultLogo from '../assets/images/defaultuser.png'

  export default {
    data () {
      return {
        userInfo: {// 用户信息
          steamid: '0',
          personaname: '匿名玩家',
          avatarmedium: defaultLogo
        },
        isLoad: false,
        sid: '', // stemid
        did: '', // dota2 数字id
        imgurl: '',
        dotaid: '',
        tips: '请输入你要查找的玩家',
        tableData: [{
          end_at: '19小时前',
          result: '胜',
          hero: '斧王',
          level: 'High',
          kda: '40',
          kdaInfo: '1／2／3',
          mid: '111'
        }],
        matchIds: [],
        matchDetils: []
      }
    },
    mounted () {
      this.sid = this.$route.params.sid
      this.did = this.getDotaid(this.sid)
      this.getUsers()
      this.getmatchIds()
    },
    methods: {
      // 获取用户信息 头像，数字ID等
      getUsers () { // 221829218
        api.user.GetPlayerSummaries({
          steamids: this.sid
        }).then((data) => {
          this.userInfo = data.response.players[0]
        })
      },
      // 获取最近5场比赛的 比赛id GetMatchHistory
      getmatchIds () {
        let accountid = this.getDotaid(this.sid)
        api.match.GetMatchHistory({
          account_id: accountid,
          matches_requested: 5
        }).then((data) => {
          if (data) {
            let matches = data.result.matches
            this.matchIds = matches.map(function (value) {
              return value.match_id
            })
            this.getmatchDetail()
          }
        }).catch((err) => {
          console.error(err)
        })
      },
      getmatchDetail () {
        // 创建返回值为promise对象的函数数组
        let promisify = this.matchIds.map(value => {
          return () => {
            return this.getDetailAjax(value)
          }
        })
        // 利用reduce 进行循环promise操作 设置初始值initialValue 为Promise.resolve()
        promisify.reduce((pre, next) => {
          return pre.then(next)
        }, Promise.resolve()).then((result) => {
          this.getShowData()
        }).catch(err => {
          console.error(err)
        })
      },
      // 获取最近5场比赛的详情
      getDetailAjax (id) {
        return new Promise((resolve, reject) => {
          api.match.GetMatchDetails({
            match_id: id
          }).then((data) => {
            let details = data.result
            this.matchDetils.push(details)
            resolve('ok')
          }).catch((err) => {
            reject(err)
          })
        })
      },
      // 得到列表数据
      getShowData () {
        // radiant_win true Or false
        this.tableData = this.matchDetils.map(match => {
          let players = match.players
          let player = players.filter((value) => {
            if (value.account_id === this.did) {
              return value
            }
          })[0]
          let rtn = {}
          let winStatus = ''
          let engine = util.get8bitNumber(player.player_slot).substr(0, 1) // 阵营
          if (!parseInt(engine)) { // 天辉
            winStatus = match.radiant_win ? '胜' : '负'
          } else { // 夜宴
            winStatus = match.radiant_win ? '负' : '胜'
          }
          rtn.result = winStatus
          util.getHeroNameFromId(player.hero_id, (nameObj) => {
            rtn.hero = util.getHeroAvatar(nameObj.name, 'vert')
          })
          var diff = Date.now() - match.start_time * 1000
          rtn.end_at = util.getLastTimeStr(diff)
          rtn.level = 'Normal' // level 不知道怎么算 ＝ ＝！ Normal,High,Very High
          rtn.kda = util.getKDA(player.kills, player.deaths, player.assists)
          rtn.kdaInfo = player.kills + '／' + player.deaths + '／' + player.assists
          rtn.match_id = match.match_id
          return rtn
        })
        this.$nextTick(function () {
          this.isLoad = true
        })
      },
      getDotaid (steamid) {
        return util.steamidToDotaid(steamid)
      },
      backfunc () {
        util.backTo()
      }
    },
    components: {
      hd,
      tb,
      loading
    }
  }
</script>
