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
    <a-form
      :form="form"
      layout="vertical"
      @submit="handleSubmit">
      <a-row :gutter="12">
        <a-col :md="24" :lg="24">
          <a-form-item :label="$t('label.ip')">
            <a-input
              :placeholder="apiParams.url.description"
              autoFocus
              v-decorator="['ip', {
                rules: [{ required: true, message: $t('message.error.required.input') }]
              }]" />
          </a-form-item>
        </a-col>
      </a-row>
      <a-row :gutter="12">
        <a-col :md="24" :lg="24">
          <a-form-item :label="$t('label.username')">
            <a-input
              :placeholder="apiParams.username.description"
              v-decorator="['username', {
                rules: [{ required: true, message: $t('message.error.required.input') }]
              }]" />
          </a-form-item>
        </a-col>
      </a-row>
      <a-row :gutter="12">
        <a-col :md="24" :lg="24">
          <a-form-item :label="$t('label.password')">
            <a-input-password
              :placeholder="apiParams.password.description"
              v-decorator="['password', {
                rules: [{ required: true, message: $t('message.error.required.input') }]
              }]" />
          </a-form-item>
        </a-col>
      </a-row>
      <a-row :gutter="12">
        <a-col :md="24" :lg="24">
          <a-form-item :label="$t('label.networkdevicetype')">
            <a-select
              :placeholder="apiParams.networkdevicetype.description"
              v-decorator="['networkdevicetype', {
                rules: [{ required: true, message: $t('message.error.select') }]
              }]"
              showSearch
              optionFilterProp="children"
              :filterOption="(input, option) => {
                return option.componentOptions.children[0].text.toLowerCase().indexOf(input.toLowerCase()) >= 0
              }" >
              <a-select-option
                v-for="opt in networkDeviceType"
                :key="opt.id">{{ $t(opt.description) }}</a-select-option>
            </a-select>
          </a-form-item>
        </a-col>
      </a-row>
      <a-row :gutter="12">
        <a-col :md="24" :lg="24">
          <a-form-item :label="$t('label.publicinterface')">
            <a-input
              v-decorator="['publicinterface']" />
          </a-form-item>
        </a-col>
      </a-row>
      <a-row :gutter="12">
        <a-col :md="24" :lg="24">
          <a-form-item :label="$t('label.privateinterface')">
            <a-input
              v-decorator="['privateinterface']" />
          </a-form-item>
        </a-col>
      </a-row>
      <a-row :gutter="12">
        <a-col :md="24" :lg="24">
          <a-form-item :label="$t('label.usageinterface')">
            <a-input
              v-decorator="['usageinterface']" />
          </a-form-item>
        </a-col>
      </a-row>
      <a-row :gutter="12">
        <a-col :md="12" :lg="12">
          <a-form-item :label="$t('label.numretries')">
            <a-input-number
              style="width: 100%"
              v-decorator="['numretries', { initialValue: 2 }]" />
          </a-form-item>
        </a-col>
        <a-col :md="12" :lg="12">
          <a-form-item :label="$t('label.timeout')">
            <a-input-number
              style="width: 100%"
              v-decorator="['timeout', { initialValue: 300 }]" />
          </a-form-item>
        </a-col>
      </a-row>
      <a-row :gutter="12">
        <a-col :md="12" :lg="12">
          <a-form-item :label="$t('label.publicnetwork')">
            <a-input
              v-decorator="['publicnetwork', { initialValue: 'untrusted' }]"
              :disabled="true" />
          </a-form-item>
        </a-col>
        <a-col :md="12" :lg="12">
          <a-form-item :label="$t('label.privatenetwork')">
            <a-input
              v-decorator="['privatenetwork', { initialValue: 'trusted' }]"
              :disabled="true" />
          </a-form-item>
        </a-col>
      </a-row>
      <a-row :gutter="12">
        <a-col :md="24" :lg="24">
          <a-form-item :label="$t('label.capacity')">
            <a-input
              v-decorator="['capacity']" />
          </a-form-item>
        </a-col>
      </a-row>
      <div :span="24" class="action-button">
        <a-button :loading="loading" @click="onCloseAction">{{ this.$t('label.cancel') }}</a-button>
        <a-button :loading="loading" ref="submit" type="primary" @click="handleSubmit">{{ this.$t('label.ok') }}</a-button>
      </div>
    </a-form>
  </div>
</template>

<script>
import { api } from '@/api'

export default {
  name: 'AddSrxFirewall',
  props: {
    resource: {
      type: Object,
      default: () => {}
    },
    action: {
      type: Object,
      default: () => {}
    }
  },
  data () {
    return {
      apiParams: {},
      loading: false,
      nsp: {}
    }
  },
  computed: {
    networkDeviceType () {
      const items = []
      items.push({
        id: 'JuniperSRXFirewall',
        description: 'label.srx.firewall'
      })

      return items
    }
  },
  beforeCreate () {
    this.form = this.$form.createForm(this)
  },
  created () {
    this.apiParams = this.$getApiParams('addSrxFirewall')
  },
  mounted () {
    if (this.resource && Object.keys(this.resource).length > 0) {
      this.nsp = this.resource
    }
  },
  inject: ['provideCloseAction', 'provideReload', 'provideCloseAction', 'parentPollActionCompletion'],
  methods: {
    onCloseAction () {
      this.provideCloseAction()
    },
    handleSubmit (e) {
      e.preventDefault()
      if (this.loading) return
      this.form.validateFieldsAndScroll(async (err, values) => {
        if (err) {
          return
        }
        const params = {}
        params.physicalnetworkid = this.resource.physicalnetworkid
        params.username = values.username
        params.password = values.password
        params.networkdevicetype = values.networkdevicetype

        const url = []
        const ip = values.ip
        url.push('https://' + ip)
        let isQuestionMarkAdded = false

        const publicInterface = values.publicinterface ? values.publicinterface : null
        if (publicInterface != null && publicInterface.length > 0) {
          if (!isQuestionMarkAdded) {
            url.push('?')
            isQuestionMarkAdded = true
          } else {
            url.push('&')
          }
          url.push('publicinterface=' + publicInterface)
        }

        const privateInterface = values.privateinterface ? values.privateinterface : null
        if (privateInterface != null && privateInterface.length > 0) {
          if (!isQuestionMarkAdded) {
            url.push('?')
            isQuestionMarkAdded = true
          } else {
            url.push('&')
          }
          url.push('privateinterface=' + publicInterface)
        }

        const usageInterface = values.usageinterface ? values.usageinterface : null
        if (usageInterface != null && usageInterface.length > 0) {
          if (!isQuestionMarkAdded) {
            url.push('?')
            isQuestionMarkAdded = true
          } else {
            url.push('&')
          }
          url.push('usageinterface=' + usageInterface)
        }

        const numretries = values.numretries ? values.numretries : null
        if (numretries != null && numretries.length > 0) {
          if (!isQuestionMarkAdded) {
            url.push('?')
            isQuestionMarkAdded = true
          } else {
            url.push('&')
          }
          url.push('numretries=' + numretries)
        }

        const timeout = values.timeout ? values.timeout : null
        if (timeout != null && timeout.length > 0) {
          if (!isQuestionMarkAdded) {
            url.push('?')
            isQuestionMarkAdded = true
          } else {
            url.push('&')
          }
          url.push('timeout=' + timeout)
        }

        const publicNetwork = values.publicnetwork ? values.publicnetwork : null
        if (publicNetwork != null && publicNetwork.length > 0) {
          if (!isQuestionMarkAdded) {
            url.push('?')
            isQuestionMarkAdded = true
          } else {
            url.push('&')
          }
          url.push('publicnetwork=' + publicNetwork)
        }

        const privateNetwork = values.privatenetwork ? values.privatenetwork : null
        if (privateNetwork != null && privateNetwork.length > 0) {
          if (!isQuestionMarkAdded) {
            url.push('?')
            isQuestionMarkAdded = true
          } else {
            url.push('&')
          }
          url.push('privatenetwork=' + privateNetwork)
        }

        const capacity = values.capacity ? values.capacity : null
        if (capacity != null && capacity.length > 0) {
          if (!isQuestionMarkAdded) {
            url.push('?')
            isQuestionMarkAdded = true
          } else {
            url.push('&')
          }
          url.push('lbdevicecapacity=' + capacity)
        }

        const dedicated = values.dedicated ? values.dedicated : false
        if (!isQuestionMarkAdded) {
          url.push('?')
          isQuestionMarkAdded = true
        } else {
          url.push('&')
        }
        url.push('lbdevicededicated=' + dedicated)

        params.url = url.join('')

        this.loading = true

        try {
          if (!this.nsp.id) {
            const addParams = {}
            addParams.name = this.resource.name
            addParams.physicalnetworkid = this.resource.physicalnetworkid
            const networkServiceProvider = await this.addNetworkServiceProvider(addParams)
            this.nsp = { ...this.resource, ...networkServiceProvider }
          }
          params.id = this.nsp.id
          const jobId = await this.addSrxFirewall(params)
          this.parentPollActionCompletion(jobId, this.action, this.$t(this.nsp.name))
          this.provideCloseAction()
          this.loading = false
        } catch (error) {
          this.loading = false
          this.$notification.error({
            message: this.$t('message.request.failed'),
            description: (error.response && error.response.headers && error.response.headers['x-description']) || error.message
          })
        }
      })
    },
    addNetworkServiceProvider (args) {
      return new Promise((resolve, reject) => {
        api('addNetworkServiceProvider', args).then(async json => {
          this.$pollJob({
            jobId: json.addnetworkserviceproviderresponse.jobid,
            successMethod: (result) => {
              resolve(result.jobresult.networkserviceprovider)
            },
            errorMethod: (result) => {
              reject(result.jobresult.errortext)
            },
            catchMessage: this.$t('error.fetching.async.job.result'),
            action: {
              isFetchData: false
            }
          })
        }).catch(error => {
          reject(error)
        })
      })
    },
    addSrxFirewall (args) {
      return new Promise((resolve, reject) => {
        api('addSrxFirewall', args).then(json => {
          const jobId = json.addsrxfirewallresponse.jobid || null
          resolve(jobId)
        }).catch(error => {
          reject(error)
        })
      })
    }
  }
}
</script>
