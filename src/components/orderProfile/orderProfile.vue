<template>
      <a-spin :spinning="spinning">
            <a-page-header :title="$t('order.profile.work_id', { id: order.work_id })" @back="() => $router.go(-1)"
                  :ghost="false">
                  <template #extra>
                        <template v-if="route.params.tp === 'audit' && isCurrent > -1 && order.status === 2">
                              <a-button key="2" danger ghost @click="r.turnState()">{{ $t('order.reject') }}</a-button>
                              <a-popconfirm :title="$t('order.agree.tips')" @confirm="next" :visible="condition"
                                    @visibleChange="handleVisibleChange">
                                    <a-button key="1" type="primary" :disabled="enabled">{{
                                                $t('order.agree')
                                    }}</a-button>
                              </a-popconfirm>

                        </template>
                        <template v-if="route.params.tp !== 'audit' && order.status === 2">
                              <a-popconfirm :title="$t('order.undo.tips')" @confirm="undoNext">
                                    <a-button key="1" danger ghost>{{
                                                $t('order.undo')
                                    }}</a-button>
                              </a-popconfirm>
                        </template>
                  </template>
                  <a-row type="flex" justify="center" align="middle">
                        <a-col :span="16">
                              <a-descriptions :column="2">
                                    <a-descriptions-item :label="$t('common.table.type')">{{ order.type === 0 ? 'DDL' :
                                                'DML'
                                    }}</a-descriptions-item>
                                    <a-descriptions-item :label="$t('common.table.env')">{{ order.idc }}
                                    </a-descriptions-item>
                                    <a-descriptions-item :label="$t('common.table.source')">{{ order.source }}
                                    </a-descriptions-item>
                                    <a-descriptions-item :label="$t('common.table.schema')">{{ order.data_base }}
                                    </a-descriptions-item>
                                    <a-descriptions-item :label="$t('order.profile.roll')">{{ order.backup ?
                                                $t('common.yes') : $t('common.no')
                                    }}</a-descriptions-item>
                                    <a-descriptions-item :label="$t('order.profile.timing')">{{ order.delay === 'none' ?
                                                $t('order.table.delay') : order.delay
                                    }}</a-descriptions-item>
                                    <a-descriptions-item :label="$t('order.profile.auditor')">
                                          <template v-for="i in order.assigned.split(',')" :key="i">
                                                <a-tag v-if="i !== '提交人'" color="#408B9B">{{ i }}</a-tag>
                                          </template>
                                    </a-descriptions-item>
                              </a-descriptions>
                        </a-col>
                        <a-col :span="8">
                              <a-progress :percent="(order.current_step / orderProfileArch.timeline.length) * 100"
                                    :strokeWidth="5" :width="150" :stroke-color="StateUsege(order.status).color"
                                    type="circle" style="position: relative">
                                    <template #format="percent">
                                          <span class="state_color">{{ StateUsege(order.status).title
                                          }}</span>
                                    </template>
                              </a-progress>
                        </a-col>
                  </a-row>
                  <br>
            </a-page-header>
            <!-- <a-card> -->
            <a-tabs>
                  <a-tab-pane key="1" :tab="$t('order.profile.profile')">
                        <Step :current="order.current_step" :step="orderProfileArch.timeline" :status="order.status">
                        </Step>
                        <br />
                        <a-row :gutter="24">
                              <a-col :xs="24" :sm="5">
                                    <a-card style="height: 500px;overflow: auto;" :title="$t('order.profile.progress')"
                                          size="small">
                                          <a-timeline
                                                :pending="order.current_step === orderProfileArch.timeline.length ? false : 'Recording...'">
                                                <a-timeline-item v-for="i in usege" :key="i.id" color="green">{{
                                                            i.username
                                                }} {{ i.action }} {{ i.time }}</a-timeline-item>
                                          </a-timeline>
                                    </a-card>
                                    <br />
                                    <a-alert message="Info" type="info" show-icon>
                                          <template #description>{{ $t('order.profile.tips') }}</template>
                                    </a-alert>
                              </a-col>
                              <a-col :xs="24" :sm="19">
                                    <a-spin :spinning="spin" :delay="100">
                                          <a-card size="small">
                                                <div class="editor_border">
                                                      <Editor container-id="profile" ref="profile" readonly
                                                            @getValues="testResults"></Editor>
                                                </div>

                                                <br />
                                                <a-table :columns="col" bordered size="small" :dataSource="tData">
                                                </a-table>
                                          </a-card>
                                    </a-spin>
                              </a-col>
                        </a-row>
                  </a-tab-pane>

                  <a-tab-pane key="2" :tab="$t('order.profile.comment')" forceRender>
                        <a-card size="small">
                              <Comment :work_id="order.work_id"></Comment>
                        </a-card>
                  </a-tab-pane>

                  <a-tab-pane key="3" :tab="$t('order.profile.results')">
                        <a-card size="small">
                              <Results :work_id="order.work_id" :status="order.status"></Results>
                        </a-card>

                  </a-tab-pane>

                  <a-tab-pane key="4" :tab="$t('order.profile.osc')" v-if="order.type === 0">
                        <a-card size="small">
                              <OSC :work_id="order.work_id"></OSC>
                        </a-card>

                  </a-tab-pane>
            </a-tabs>
            <!-- </a-card> -->
      </a-spin>

      <RejectModal :work_id="order.work_id" ref="r" @spin="() => spinning = !spinning"></RejectModal>
</template>
<script lang="ts"  setup>
import RejectModal from "./rejectModal.vue";
import Editor from "@/components/editor/editor.vue";
import Comment from "./comment.vue";
import Results from "./results.vue";
import Step from '@/components/steps/steps.vue'
import router from "@/router";
import OSC from "./osc.vue";
import { useStore } from "@/store";
import { computed, ref } from "@vue/reactivity";
import { onMounted } from "@vue/runtime-core";
import FetchMixins from "@/mixins/fetch"
import { AxiosResponse } from "axios";
import { Res } from "@/config/request";
import { StateUsege } from "@/lib"
import { Request, SQLTestParams } from "@/apis/orderPostApis";
import { SQLTesting } from "@/types";
import { useRoute } from "vue-router";
import { Timeline } from "@/apis/fetchSchema";
import JunoMixin from '@/mixins/juno'
import WsSocket from "@/socket";
import { onUnmounted } from "vue";
import { debounce } from "lodash-es"

interface stepUsege {
      action: string
      time: string
      username: string
      id: number
}

const spinning = ref(false)

const { col } = JunoMixin()

const r = ref()

const store = useStore()

const profile = ref()

const tData = ref()

const enabled = ref(store.state.order.order.type !== 2)

const spin = ref(false)

const request = new Request

const route = useRoute()

const order = computed(() => store.state.order.order)

const sock = new WsSocket(`/fetch/order_state?work_id=${order.value.work_id}`)

const { FetchStepUsage, FetchProfileSQL, orderProfileArch, fetchRequest } = FetchMixins()

const usege = ref([] as stepUsege[])

const isCurrent = ref(-1)

const condition = ref(false)

const handleVisibleChange = (bool: boolean) => {
      if (enabled.value) {
            condition.value = false
            return
      }
      bool ? condition.value = true : condition.value = false
      // !enabled.value ? condition.value = true : null
}

const testResults = debounce((sql: string) => {
      spin.value = !spin.value
      request.Test({
            source_id: order.value.source_id as string,
            kind: order.value.type,
            data_base: order.value.data_base,
            sql: sql
      } as SQLTestParams)
            .then((res: AxiosResponse<Res<SQLTesting[]>>) => {
                  let counter = 0
                  tData.value = res.data.payload
                  tData.value.forEach((item: SQLTesting) => {
                        if (item.level !== 0) {
                              counter++
                        }
                  })
                  enabled.value = counter !== 0
            })
            .finally(() => spin.value = !spin.value)
}, 200)

const next = () => {
      spinning.value = !spinning.value
      request.Next({
            work_id: order.value.work_id as string,
            flag: order.value.current_step as number,
            tp: 'agree',
            source_id: order.value.source_id
      }).then(() => {
            router.go(-1)
      }).finally(() => spinning.value = !spinning.value)
}

const undoNext = () => {
      spinning.value = !spinning.value
      request.Undo({
            work_id: order.value.work_id as string,
            tp: 'undo',
      }).then(() => {
            router.go(-1)
      }).finally(() => spinning.value = !spinning.value)
}

const recv = (e: any) => {
      store.commit("order/SET_ORDER_STATUS", JSON.parse(e.data))
}

onMounted(() => {
      sock.create()
      sock.ping()
      sock.race(recv)
      fetchRequest.TimeLine(order.value.source_id as string).then((res: AxiosResponse<Res<Timeline[]>>) => {
            if (res.data.code === 5555) {
                  router.go(-1)
            } else {
                  orderProfileArch.timeline = res.data.payload
                  orderProfileArch.timeline[order.value.current_step] !== undefined ? isCurrent.value = orderProfileArch.timeline[order.value.current_step].auditor.indexOf(store.state.user.account.user) : null
            }


      })

      FetchStepUsage(order.value.work_id).then((res: AxiosResponse<Res<stepUsege[]>>) => {
            usege.value = res.data.payload
      })

      FetchProfileSQL(order.value.work_id).then((res: AxiosResponse<Res<{ [key: string]: string }>>) => {
            profile.value.ChangeEditorText(res.data.payload.sqls)
            store.commit("common/ORDER_SET_SQL", res.data.payload.sqls)
      })
})

onUnmounted(() => {
      sock.send("1")
      sock.close()
})


</script>