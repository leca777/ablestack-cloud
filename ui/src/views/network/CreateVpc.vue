// Licensed to the Apache Software Foundation (ASF) under one
// or more contributor license agreements.  See the NOTICE file
// distributed with this work for additional information
// regarding copyright ownership.  The ASF licenses this file
// to you under the Apache License, Version 2.0 (the
// "License"); you may not use this file except in compliance
// with the License.  You may obtain a copy of the License at
//
//   http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing,
// software distributed under the License is distributed on an
// "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
// KIND, either express or implied.  See the License for the
// specific language governing permissions and limitations
// under the License.
<template>
  <div class="form-layout" v-ctrl-enter="handleSubmit">
    <a-spin :spinning="loading">
      <a-form
        :form="form"
        layout="vertical">
        <a-form-item>
          <tooltip-label slot="label" :title="$t('label.name')" :tooltip="apiParams.name.description"/>
          <a-input
            v-decorator="['name', {
              rules: [{ required: true, message: $t('message.error.required.input') }]
            }]"
            :placeholder="apiParams.name.description"
            autoFocus/>
        </a-form-item>
        <a-form-item>
          <tooltip-label slot="label" :title="$t('label.displaytext')" :tooltip="apiParams.displaytext.description"/>
          <a-input
            v-decorator="['displaytext', {
              rules: [{ required: true, message: $t('message.error.required.input') }]
            }]"
            :placeholder="apiParams.displaytext.description"/>
        </a-form-item>
        <a-form-item>
          <tooltip-label slot="label" :title="$t('label.zoneid')" :tooltip="apiParams.zoneid.description"/>
          <a-select
            :loading="loadingZone"
            v-decorator="['zoneid', {
              initialValue: this.selectedZone,
              rules: [{ required: true, message: `${this.$t('label.required')}`}]
            }]"
            @change="val => changeZone(val)"
            showSearch
            optionFilterProp="children"
            :filterOption="(input, option) => {
              return option.componentOptions.propsData.label.toLowerCase().indexOf(input.toLowerCase()) >= 0
            }" >
            <a-select-option v-for="zone in zones" :key="zone.id" :label="zone.name">
              <span>
                <resource-icon v-if="zone.icon" :image="zone.icon.base64image" size="1x" style="margin-right: 5px"/>
                <a-icon v-else type="global" style="margin-right: 5px" />
                {{ zone.name }}
              </span>
            </a-select-option>
          </a-select>
        </a-form-item>
        <a-form-item>
          <tooltip-label slot="label" :title="$t('label.cidr')" :tooltip="apiParams.cidr.description"/>
          <a-input
            v-decorator="['cidr', {
              rules: [{ required: true, message: $t('message.error.required.input') }]
            }]"
            :placeholder="apiParams.cidr.description"/>
        </a-form-item>
        <a-form-item>
          <tooltip-label slot="label" :title="$t('label.networkdomain')" :tooltip="apiParams.networkdomain.description"/>
          <a-input
            v-decorator="['networkdomain']"
            :placeholder="apiParams.networkdomain.description"/>
        </a-form-item>
        <a-form-item>
          <tooltip-label slot="label" :title="$t('label.vpcofferingid')" :tooltip="apiParams.vpcofferingid.description"/>
          <a-select
            :loading="loadingOffering"
            v-decorator="['vpcofferingid', {
              initialValue: this.selectedOffering,
              rules: [{ required: true, message: `${this.$t('label.required')}`}]}]"
            showSearch
            optionFilterProp="children"
            :filterOption="(input, option) => {
              return option.componentOptions.children[0].text.toLowerCase().indexOf(input.toLowerCase()) >= 0
            }" >
            <a-select-option :value="offering.id" v-for="offering in vpcOfferings" :key="offering.id">
              {{ offering.name }}
            </a-select-option>
          </a-select>
        </a-form-item>
        <a-form-item>
          <tooltip-label slot="label" :title="$t('label.start')" :tooltip="apiParams.start.description"/>
          <a-switch v-decorator="['start', {initialValue: true}]" defaultChecked />
        </a-form-item>
      </a-form>
      <div :span="24" class="action-button">
        <a-button @click="closeAction">{{ this.$t('label.cancel') }}</a-button>
        <a-button :loading="loading" ref="submit" type="primary" @click="handleSubmit">{{ this.$t('label.ok') }}</a-button>
      </div>
    </a-spin>
  </div>
</template>
<script>
import { api } from '@/api'
import ResourceIcon from '@/components/view/ResourceIcon'
import TooltipLabel from '@/components/widgets/TooltipLabel'

export default {
  name: 'CreateVpc',
  components: {
    ResourceIcon,
    TooltipLabel
  },
  data () {
    return {
      loading: false,
      loadingZone: false,
      loadingOffering: false,
      selectedZone: '',
      zones: [],
      vpcOfferings: [],
      selectedOffering: ''
    }
  },
  beforeCreate () {
    this.form = this.$form.createForm(this)
    this.apiParams = this.$getApiParams('createVPC')
  },
  created () {
    this.fetchData()
  },
  methods: {
    fetchData () {
      this.fetchZones()
    },
    fetchZones () {
      this.loadingZone = true
      api('listZones', { listAll: true, showicon: true }).then((response) => {
        const listZones = response.listzonesresponse.zone || []
        this.zones = listZones.filter(zone => !zone.securitygroupsenabled)
        this.selectedZone = ''
        if (this.zones.length > 0) {
          this.selectedZone = this.zones[0].id
          this.changeZone(this.selectedZone)
        }
      }).finally(() => {
        this.loadingZone = false
      })
    },
    changeZone (value) {
      this.selectedZone = value
      if (this.selectedZone === '') {
        this.selectedOffering = ''
        return
      }
      this.fetchOfferings()
    },
    fetchOfferings () {
      this.loadingOffering = true
      api('listVPCOfferings', { zoneid: this.selectedZone, state: 'Enabled' }).then((reponse) => {
        this.vpcOfferings = reponse.listvpcofferingsresponse.vpcoffering
        this.selectedOffering = this.vpcOfferings[0].id || ''
      }).finally(() => {
        this.loadingOffering = false
      })
    },
    closeAction () {
      this.$emit('close-action')
    },
    handleSubmit (e) {
      e.preventDefault()
      if (this.loading) return
      this.form.validateFieldsAndScroll((err, values) => {
        if (err) {
          return
        }
        const params = {}
        for (const key in values) {
          const input = values[key]
          if (input === '' || input === null || input === undefined) {
            continue
          }
          params[key] = input
        }
        this.loading = true
        const title = this.$t('label.add.vpc')
        const description = this.$t('message.success.add.vpc.network')
        api('createVPC', params).then(json => {
          const jobId = json.createvpcresponse.jobid
          if (jobId) {
            this.$pollJob({
              jobId,
              title,
              description,
              loadingMessage: `${title} ${this.$t('label.in.progress')}`,
              catchMessage: this.$t('error.fetching.async.job.result')
            })
          }
          this.closeAction()
        }).catch(error => {
          this.$notifyError(error)
        }).finally(() => {
          this.loading = false
        })
      })
    }
  }
}
</script>
<style lang="scss" scoped>
.form-layout {
  width: 80vw;
  @media (min-width: 700px) {
    width: 600px;
  }
}

.form {
  margin: 10px 0;
}
</style>
