<template>
  <div class="form-model-model" v-clickoutside="_handleClose">
    <div class="title-box">
      <span class="name">{{$t('当前节点设置')}}</span>
      <span class="go-subtask">
        <!-- Component can't pop up box to do component processing -->
        <m-log :item="backfillItem">
          <template slot="history"><a href="javascript:" @click="_seeHistory" ><i class="iconfont">&#xe6ee;</i><em>{{$t('查看历史')}}</em></a></template>
          <template slot="log"><a href="javascript:"><i class="iconfont">&#xe691;</i><em>{{$t('查看日志')}}</em></a></template>
        </m-log>
        <a href="javascript:" @click="_goSubProcess" v-if="_isGoSubProcess"><i class="iconfont">&#xe600;</i><em>{{$t('进入该子节点')}}</em></a>
      </span>
    </div>
    <div class="content-box" v-if="isContentBox">
      <div class="from-model">

        <!-- Node name -->
        <div class="clearfix list">
          <div class="text-box"><span>{{$t('节点名称')}}</span></div>
          <div class="cont-box">
            <label class="label-box">
              <x-input
                      type="text"
                      v-model="name"
                      :disabled="isDetails"
                      :placeholder="$t('请输入name(必填)')"
                      maxlength="100"
                      @on-blur="_verifName()"
                      autocomplete="off">
              </x-input>
            </label>
          </div>
        </div>

        <!-- Running sign -->
        <div class="clearfix list">
          <div class="text-box"><span>{{$t('运行标志')}}</span></div>
          <div class="cont-box">
            <label class="label-box">
              <x-radio-group v-model="runFlag" >
                <x-radio :label="'NORMAL'" :disabled="isDetails">{{$t('正常')}}</x-radio>
                <x-radio :label="'FORBIDDEN'" :disabled="isDetails">{{$t('禁止执行')}}</x-radio>
              </x-radio-group>
            </label>
          </div>
        </div>

        <!-- desc -->
        <div class="clearfix list">
          <div class="text-box">
            <span>{{$t('描述')}}</span>
          </div>
          <div class="cont-box">

            <label class="label-box">
              <x-input
                      resize
                      :autosize="{minRows:2}"
                      type="textarea"
                      :disabled="isDetails"
                      v-model="desc"
                      :placeholder="$t('请输入desc')"
                      autocomplete="off">
              </x-input>
            </label>
          </div>
        </div>

        <!-- Task priority -->
        <div class="clearfix list">
          <div class="text-box">
            <span>{{$t('任务优先级')}}</span>
          </div>
          <div class="cont-box">
            <label class="label-box">
              <m-priority v-model="taskInstancePriority" style="width: 180px;"></m-priority>
            </label>
          </div>
        </div>

        <!-- Number of failed retries -->
        <div class="clearfix list" v-if="taskType !== 'SUB_PROCESS'">
          <div class="text-box">
            <span>{{$t('失败重试次数')}}</span>
          </div>
          <div class="cont-box">
            <m-select-input v-model="maxRetryTimes" :list="[0,1,2,3,4]">
            </m-select-input>
            <span>({{$t('次')}})</span>
            <span class="text-b">{{$t('失败重试间隔')}}</span>
            <m-select-input v-model="retryInterval" :list="[1,10,30,60,120]">
            </m-select-input>
            <span>({{$t('分')}})</span>
          </div>
        </div>

        <!-- Task timeout alarm -->
        <m-timeout-alarm
                ref="timeout"
                :backfill-item="backfillItem"
                @on-timeout="_onTimeout">
        </m-timeout-alarm>

        <!-- shell node -->
        <m-shell
                v-if="taskType === 'SHELL'"
                @on-params="_onParams"
                ref="SHELL"
                :backfill-item="backfillItem">
        </m-shell>
        <!-- sub_process node -->
        <m-sub-process
                v-if="taskType === 'SUB_PROCESS'"
                @on-params="_onParams"
                @on-set-process-name="_onSetProcessName"
                ref="SUB_PROCESS"
                :backfill-item="backfillItem">
        </m-sub-process>
        <!-- procedure node -->
        <m-procedure
                v-if="taskType === 'PROCEDURE'"
                @on-params="_onParams"
                ref="PROCEDURE"
                :backfill-item="backfillItem">
        </m-procedure>
        <!-- sql node -->
        <m-sql
                v-if="taskType === 'SQL'"
                @on-params="_onParams"
                ref="SQL"
                :backfill-item="backfillItem">
        </m-sql>
        <!-- spark node -->
        <m-spark
                v-if="taskType === 'SPARK'"
                @on-params="_onParams"
                ref="SPARK"
                :backfill-item="backfillItem">
        </m-spark>
        <!-- mr node -->
        <m-mr
                v-if="taskType === 'MR'"
                @on-params="_onParams"
                ref="MR"
                :backfill-item="backfillItem">
        </m-mr>
        <!-- python node -->
        <m-python
                v-if="taskType === 'PYTHON'"
                @on-params="_onParams"
                ref="PYTHON"
                :backfill-item="backfillItem">
        </m-python>
        <!-- dependent node -->
        <m-dependent
                v-if="taskType === 'DEPENDENT'"
                @on-dependent="_onDependent"
                ref="DEPENDENT"
                :backfill-item="backfillItem">
        </m-dependent>

      </div>
    </div>
    <div class="bottom-box">
      <div class="submit" style="background: #fff;">
        <x-button type="text" @click="close()"> {{$t('取消')}} </x-button>
        <x-button type="primary" shape="circle" :loading="spinnerLoading" @click="ok()" :disabled="isDetails" v-ps="['GENERAL_USER']">{{spinnerLoading ? 'Loading...' : $t('确认添加')}} </x-button>
      </div>
    </div>
  </div>
</template>
<script>
  import _ from 'lodash'
  import mLog from './log'
  import mMr from './tasks/mr'
  import mSql from './tasks/sql'
  import i18n from '@/module/i18n'
  import mShell from './tasks/shell'
  import mSpark from './tasks/spark'
  import mPython from './tasks/python'
  import { isNameExDag,rtBantpl } from './../plugIn/util'
  import JSP from './../plugIn/jsPlumbHandle'
  import mProcedure from './tasks/procedure'
  import mDependent from './tasks/dependent'
  import mSubProcess from './tasks/sub_process'
  import mSelectInput from './_source/selectInput'
  import mTimeoutAlarm from './_source/timeoutAlarm'
  import clickoutside from '@/module/util/clickoutside'
  import disabledState from '@/module/mixin/disabledState'
  import mPriority from '@/module/components/priority/priority'

  export default {
    name: 'form-model',
    data () {
      return {
        // loading
        spinnerLoading: false,
        // node name
        name: ``,
        // desc
        desc: '',
        // Node echo data
        backfillItem: {},
        // Resource(list)
        resourcesList: [],
        // dependence
        dependence: {},
        // Current node params data
        params: {},
        // Running sign
        runFlag: 'NORMAL',
        // The second echo problem caused by the node data is specifically which node hook caused the unfinished special treatment
        isContentBox: false,
        // Number of failed retries
        maxRetryTimes: '0',
        // Failure retry interval
        retryInterval: '1',
        // Task timeout alarm
        timeout: {},
        // Task priority
        taskInstancePriority: 'MEDIUM'
      }
    },
    /**
     * Click on events that are not generated internally by the component
     */
    directives: { clickoutside },
    mixins: [disabledState],
    props: {
      id: Number,
      taskType: String,
      self: Object
    },
    methods: {
      /**
       * depend
       */
      _onDependent (o) {
        this.dependence = Object.assign(this.dependence, {}, o)
      },
      /**
       * Task timeout alarm
       */
      _onTimeout (o) {
        this.timeout = Object.assign(this.timeout, {}, o)
      },
      /**
       * Click external to close the current component
       */
      _handleClose () {
        this.close()
      },
      /**
       * Jump to task instance
       */
      _seeHistory () {
        this.self.$router.push({
          name: 'task-instance-list',
          query: {
            processInstanceId: this.self.$route.params.id,
            taskName: this.backfillItem.name
          }
        })
        this.$modal.destroy()
      },
      /**
       * Enter the child node to judge the process instance or the process definition
       * @param  type = instance
       */
      _goSubProcess () {
        if (_.isEmpty(this.backfillItem)) {
          this.$message.warning(`${i18n.$t('新创建子工作流还未执行，不能进入子工作流')}`)
          return
        }
        if (this.router.history.current.name === 'projects-instance-details') {
          let stateId = $(`#${this.id}`).attr('data-state-id') || null
          if (!stateId) {
            this.$message.warning(`${i18n.$t('该任务还未执行，不能进入子工作流')}`)
            return
          }
          this.store.dispatch('dag/getSubProcessId', { taskId: stateId }).then(res => {
            this.$emit('onSubProcess', {
              subProcessId: res.data.subProcessInstanceId,
              fromThis: this
            })
          }).catch(e => {
            this.$message.error(e.msg || '')
          })
        } else {
          this.$emit('onSubProcess', {
            subProcessId: this.backfillItem.params.processDefinitionId,
            fromThis: this
          })
        }
      },
      /**
       * return params
       */
      _onParams (o) {
        this.params = Object.assign(this.params, {}, o)
      },
      /**
       * verification name
       */
      _verifName () {
        if (!_.trim(this.name)) {
          this.$message.warning(`${i18n.$t('请输入名称(必填)')}`)
          return false
        }
        if (this.name === this.backfillItem.name) {
          return true
        }
        // Name repeat depends on dom backfill dependent store
        if (isNameExDag(this.name, _.isEmpty(this.backfillItem) ? 'dom' : 'backfill')) {
          this.$message.warning(`${i18n.$t('名称已存在请重新输入')}`)
          return false
        }
        return true
      },
      /**
       * Global verification procedure
       */
      _verification () {
        // Verify name
        if (!this._verifName()) {
          return
        }
        // Verify task alarm parameters
        if (!this.$refs['timeout']._verification()) {
          return
        }
        // Verify node parameters
        if (!this.$refs[this.taskType]._verification()) {
          return
        }

        $(`#${this.id}`).find('span').text(this.name)

        // Store the corresponding node data structure
        this.$emit('addTaskInfo', {
          item: {
            type: this.taskType,
            id: this.id,
            name: this.name,
            params: this.params,
            desc: this.desc,
            runFlag: this.runFlag,
            dependence: this.dependence,
            maxRetryTimes: this.maxRetryTimes,
            retryInterval: this.retryInterval,
            timeout: this.timeout,
            taskInstancePriority: this.taskInstancePriority
          },
          fromThis: this
        })
      },
      /**
       * Sub-workflow selected node echo name
       */
      _onSetProcessName (name) {
        this.name = name
      },
      /**
       * Submit verification
       */
      ok () {
        this._verification()
      },
      /**
       * Close and destroy component and component internal events
       */
      close () {
        let flag = false
        // Delete node without storage
        if (!this.backfillItem.name) {
          flag = true
        }
        this.isContentBox = false
        // flag Whether to delete a node this.$destroy()
        this.$emit('close', {
          flag: flag,
          fromThis: this
        })
      }
    },
    watch: {
      runFlag(val){
        let dom = $(`#${this.id}`).find('.ban-p')
        dom.html('')
        if (val === 'FORBIDDEN') {
          dom.append(rtBantpl())
        }
      }
    },
    created () {
      // Unbind copy and paste events
      JSP.removePaste()
      // Backfill data
      let taskList = this.store.state.dag.tasks
      let o = {}
      if (taskList.length) {
        taskList.forEach(v => {
          if (v.id === this.id) {
            o = v
            this.backfillItem = v
          }
        })
        // Non-null objects represent backfill
        if (!_.isEmpty(o)) {
          this.name = o.name
          this.taskInstancePriority = o.taskInstancePriority
          this.runFlag = o.runFlag || 'NORMAL'
          this.desc = o.desc
          this.maxRetryTimes = o.maxRetryTimes
          this.retryInterval = o.retryInterval
        }
      }
      this.isContentBox = true
    },
    mounted () {},
    updated () {
    },
    beforeDestroy () {
    },
    destroyed () {
    },
    computed: {
      /**
       * Child workflow entry show/hide
       */
      _isGoSubProcess () {
        return this.taskType === 'SUB_PROCESS' && this.name
      }
    },
    components: {
      mMr,
      mShell,
      mSubProcess,
      mProcedure,
      mSql,
      mLog,
      mSpark,
      mPython,
      mDependent,
      mSelectInput,
      mTimeoutAlarm,
      mPriority
    }
  }
</script>

<style lang="scss" rel="stylesheet/scss">
  .form-model-model {
    width: 720px;
    position: relative;
    .title-box {
      height: 61px;
      border-bottom: 1px solid #DCDEDC;
      position: relative;
      .name {
        position: absolute;
        left: 24px;
        top: 18px;
        font-size: 16px;
      }
      .go-subtask {
        position: absolute;
        right: 30px;
        top: 17px;
        a {
          font-size: 14px;
          color: #0097e0;
          margin-left: 10px;
          i.iconfont {
            font-size: 18px;
            vertical-align: middle;
          }
          em {
            color: #333;
            vertical-align: middle;
            font-style: normal;
            vertical-align: middle;
            padding-left: 2px;
          }
          &:hover {
            em {
              text-decoration: underline;
            }
          }
        }
      }
    }
    .bottom-box {
      position: absolute;
      bottom: 0;
      left: 0;
      width: 100%;
      text-align: right;
      height: 60px;
      line-height: 60px;
      border-top: 1px solid #DCDEDC;
      background: #fff;
      .submit {
        padding-right: 20px;
        position: relative;
        z-index: 9;
      }
    }
    .content-box {
      overflow-y: scroll;
      height: calc(100vh - 61px);
      padding-bottom: 60px;
    }
  }
  .from-model {
    padding-top: 26px;
    >div {
      clear: both;
    }
    .list {
      position: relative;
      margin-bottom: 10px;
      .text-box {
        width: 110px;
        float: left;
        text-align: right;
        margin-right: 10px;
        >span {
          font-size: 14px;
          color: #777;
          display: inline-block;
          padding-top: 6px;
        }
      }
      .cont-box {
        width: 580px;
        float: left;
        .label-box {
          width: 100%;
        }
        .text-b {
          font-size: 14px;
          color: #777;
          display: inline-block;
          padding:0 6px 0 20px;
        }
      }
      .add {
        line-height: 32px;
        a {
          color: #0097e0;
        }
      }
      &:hover {
      }
      .list-t {
        width: 50%;
        float: left;
      }
    }
  }
</style>
