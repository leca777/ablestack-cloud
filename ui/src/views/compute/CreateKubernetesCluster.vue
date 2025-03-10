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
        @submit="handleSubmit"
        layout="vertical">
        <a-form-item>
          <tooltip-label slot="label" :title="$t('label.name')" :tooltip="apiParams.name.description"/>
          <a-input
            v-decorator="['name', {
              rules: [{ required: true, message: $t('message.error.kubecluster.name') }]
            }]"
            :placeholder="apiParams.name.description"
            autoFocus />
        </a-form-item>
        <a-form-item>
          <tooltip-label slot="label" :title="$t('label.description')" :tooltip="apiParams.description.description"/>
          <a-input
            v-decorator="['description', {
              rules: [{ required: true, message: $t('message.error.cluster.description') }]
            }]"
            :placeholder="apiParams.description.description"/>
        </a-form-item>
        <a-form-item>
          <tooltip-label slot="label" :title="$t('label.zoneid')" :tooltip="apiParams.zoneid.description"/>
          <a-select
            id="zone-selection"
            v-decorator="['zoneid', {
              rules: [{ required: true, message: $t('message.error.zone.for.cluster') }]
            }]"
            showSearch
            optionFilterProp="children"
            :filterOption="(input, option) => {
              return option.componentOptions.propsData.label.toLowerCase().indexOf(input.toLowerCase()) >= 0
            }"
            :loading="zoneLoading"
            :placeholder="apiParams.zoneid.description"
            @change="val => { this.handleZoneChange(this.zones[val]) }">
            <a-select-option v-for="(opt, optIndex) in this.zones" :key="optIndex" :label="opt.name || opt.description">
              <span>
                <resource-icon v-if="opt.icon" :image="opt.icon.base64image" size="1x" style="margin-right: 5px"/>
                <a-icon v-else type="global" style="margin-right: 5px" />
                {{ opt.name || opt.description }}
              </span>
            </a-select-option>
          </a-select>
        </a-form-item>
        <a-form-item>
          <tooltip-label slot="label" :title="$t('label.kubernetesversionid')" :tooltip="apiParams.kubernetesversionid.description"/>
          <a-select
            id="version-selection"
            v-decorator="['kubernetesversionid', {
              rules: [{ required: true, message: $t('message.error.version.for.cluster') }]
            }]"
            showSearch
            optionFilterProp="children"
            :filterOption="(input, option) => {
              return option.componentOptions.children[0].text.toLowerCase().indexOf(input.toLowerCase()) >= 0
            }"
            :loading="kubernetesVersionLoading"
            :placeholder="apiParams.kubernetesversionid.description"
            @change="val => { this.handleKubernetesVersionChange(this.kubernetesVersions[val]) }">
            <a-select-option v-for="(opt, optIndex) in this.kubernetesVersions" :key="optIndex">
              {{ opt.name || opt.description }}
            </a-select-option>
          </a-select>
        </a-form-item>
        <a-form-item>
          <tooltip-label slot="label" :title="$t('label.serviceofferingid')" :tooltip="apiParams.serviceofferingid.description"/>
          <a-select
            id="offering-selection"
            v-decorator="['serviceofferingid', {
              rules: [{ required: true, message: $t('message.error.serviceoffering.for.cluster') }]
            }]"
            showSearch
            optionFilterProp="children"
            :filterOption="(input, option) => {
              return option.componentOptions.children[0].text.toLowerCase().indexOf(input.toLowerCase()) >= 0
            }"
            :loading="serviceOfferingLoading"
            :placeholder="apiParams.serviceofferingid.description">
            <a-select-option v-for="(opt, optIndex) in this.serviceOfferings" :key="optIndex">
              {{ opt.name || opt.description }}
            </a-select-option>
          </a-select>
        </a-form-item>
        <a-form-item>
          <tooltip-label slot="label" :title="$t('label.noderootdisksize')" :tooltip="apiParams.noderootdisksize.description"/>
          <a-input
            v-decorator="['noderootdisksize', {
              initialValue: '8',
              rules: [{
                validator: (rule, value, callback) => {
                  if (value && (isNaN(value) || value < 8)) {
                    callback(this.$t('messgae.validate.min').replace('{0}', '8GB'))
                  }
                  callback()
                }
              }]
            }]"
            :placeholder="apiParams.noderootdisksize.description"/>
        </a-form-item>
        <a-form-item>
          <tooltip-label slot="label" :title="$t('label.networkid')" :tooltip="apiParams.networkid.description"/>
          <a-select
            id="network-selection"
            v-decorator="['networkid', {}]"
            showSearch
            optionFilterProp="children"
            :filterOption="(input, option) => {
              return option.componentOptions.children[0].text.toLowerCase().indexOf(input.toLowerCase()) >= 0
            }"
            :loading="networkLoading"
            :placeholder="apiParams.networkid.description">
            <a-select-option v-for="(opt, optIndex) in this.networks" :key="optIndex">
              {{ opt.name || opt.description }}
            </a-select-option>
          </a-select>
        </a-form-item>
        <a-form-item :label="$t('label.haenable')" v-if="this.selectedKubernetesVersion != null && this.selectedKubernetesVersion != undefined && this.selectedKubernetesVersion.supportsha === true">
          <a-switch v-decorator="['haenable', {initialValue: this.haEnabled}]" :checked="this.haEnabled" @change="val => { this.haEnabled = val }" />
        </a-form-item>
        <a-form-item v-if="this.haEnabled">
          <tooltip-label slot="label" :title="$t('label.controlnodes')" :tooltip="apiParams.controlnodes.description"/>
          <a-input
            v-decorator="['controlnodes', {
              initialValue: '2',
              rules: [{ required: true, message: $t('message.error.input.value') },
                      {
                        validator: (rule, value, callback) => {
                          if (value && (isNaN(value) || value < 2)) {
                            callback(this.$t('message.validate.number'))
                          }
                          callback()
                        }
                      }
              ]
            }]"
            :placeholder="apiParams.controlnodes.description"/>
        </a-form-item>
        <a-form-item v-if="this.haEnabled">
          <tooltip-label slot="label" :title="$t('label.externalloadbalanceripaddress')" :tooltip="apiParams.externalloadbalanceripaddress.description"/>
          <a-input
            v-decorator="['externalloadbalanceripaddress', {}]"
            :placeholder="apiParams.externalloadbalanceripaddress.description"/>
        </a-form-item>
        <a-form-item>
          <tooltip-label slot="label" :title="$t('label.cks.cluster.size')" :tooltip="apiParams.size.description"/>
          <a-input
            v-decorator="['size', {
              initialValue: '1',
              rules: [{ required: true, message: $t('message.error.size.for.cluster') },
                      {
                        validator: (rule, value, callback) => {
                          if (value && (isNaN(value) || value <= 0)) {
                            callback(this.$t('message.validate.number'))
                          }
                          callback()
                        }
                      }
              ]
            }]"
            :placeholder="apiParams.size.description"/>
        </a-form-item>
        <a-form-item>
          <tooltip-label slot="label" :title="$t('label.keypair')" :tooltip="apiParams.keypair.description"/>
          <a-select
            id="keypair-selection"
            v-decorator="['keypair', {}]"
            showSearch
            optionFilterProp="children"
            :filterOption="(input, option) => {
              return option.componentOptions.children[0].text.toLowerCase().indexOf(input.toLowerCase()) >= 0
            }"
            :loading="keyPairLoading"
            :placeholder="apiParams.keypair.description">
            <a-select-option v-for="(opt, optIndex) in this.keyPairs" :key="optIndex">
              {{ opt.name || opt.description }}
            </a-select-option>
          </a-select>
        </a-form-item>
        <div v-if="$store.getters.features.kubernetesclusterexperimentalfeaturesenabled">
          <a-form-item :label="$t('label.private.registry')">
            <a-switch v-decorator="['privateregistry']" @change="checked => { this.usePrivateRegistry = checked }" />
          </a-form-item>
          <div v-if="usePrivateRegistry">
            <a-form-item>
              <tooltip-label slot="label" :title="$t('label.username')" :tooltip="apiParams.dockerregistryusername.description"/>
              <a-input
                v-decorator="['dockerregistryusername', {
                  rules: [{ required: true, message: $t('label.required') }]
                }]"
                :placeholder="apiParams.dockerregistryusername.description"/>
            </a-form-item>
            <a-form-item>
              <tooltip-label slot="label" :title="$t('label.password')" :tooltip="apiParams.dockerregistrypassword.description"/>
              <a-input-password
                v-decorator="['dockerregistrypassword', {
                  rules: [{ required: true, message: $t('label.required') }]
                }]"
                :placeholder="apiParams.dockerregistrypassword.description"/>
            </a-form-item>
            <a-form-item>
              <tooltip-label slot="label" :title="$t('label.url')" :tooltip="apiParams.dockerregistryurl.description"/>
              <a-input
                v-decorator="['dockerregistryurl', {
                  rules: [{ required: true, message: $t('label.required') }]
                }]"
                :placeholder="apiParams.dockerregistryurl.description"/>
            </a-form-item>
            <a-form-item>
              <tooltip-label slot="label" :title="$t('label.email')" :tooltip="apiParams.dockerregistryemail.description"/>
              <a-input
                v-decorator="['dockerregistryemail', {
                  rules: [{ required: true, message: $t('label.required') }]
                }]"
                :placeholder="apiParams.dockerregistryemail.description"/>
            </a-form-item>
          </div>
        </div>
        <div :span="24" class="action-button">
          <a-button @click="closeAction">{{ this.$t('label.cancel') }}</a-button>
          <a-button :loading="loading" ref="submit" type="primary" @click="handleSubmit">{{ this.$t('label.ok') }}</a-button>
        </div>
      </a-form>
    </a-spin>
  </div>
</template>

<script>
import { api } from '@/api'
import ResourceIcon from '@/components/view/ResourceIcon'
import TooltipLabel from '@/components/widgets/TooltipLabel'

export default {
  name: 'CreateKubernetesCluster',
  components: {
    TooltipLabel,
    ResourceIcon
  },
  props: {},
  data () {
    return {
      zones: [],
      zoneLoading: false,
      selectedZone: {},
      kubernetesVersions: [],
      kubernetesVersionLoading: false,
      selectedKubernetesVersion: {},
      serviceOfferings: [],
      serviceOfferingLoading: false,
      networks: [],
      networkLoading: false,
      keyPairs: [],
      keyPairLoading: false,
      haEnabled: false,
      usePrivateRegistry: false,
      loading: false
    }
  },
  beforeCreate () {
    this.form = this.$form.createForm(this)
    this.apiParams = this.$getApiParams('createKubernetesCluster')
  },
  created () {
    this.networks = [
      {
        id: null,
        name: ''
      }
    ]
    this.keyPairs = [
      {
        id: '',
        name: ''
      }
    ]
    this.fetchData()
  },
  methods: {
    fetchData () {
      this.fetchZoneData()
      this.fetchNetworkData()
      this.fetchKeyPairData()
    },
    isValidValueForKey (obj, key) {
      return key in obj && obj[key] != null
    },
    arrayHasItems (array) {
      return array !== null && array !== undefined && Array.isArray(array) && array.length > 0
    },
    isObjectEmpty (obj) {
      return !(obj !== null && obj !== undefined && Object.keys(obj).length > 0 && obj.constructor === Object)
    },
    fetchZoneData () {
      const params = {}
      this.zoneLoading = true
      params.showicon = true
      api('listZones', params).then(json => {
        const listZones = json.listzonesresponse.zone
        this.zones = this.zones.concat(listZones)
      }).finally(() => {
        this.zoneLoading = false
        if (this.arrayHasItems(this.zones)) {
          this.form.setFieldsValue({
            zoneid: 0
          })
          this.handleZoneChange(this.zones[0])
        }
      })
    },
    handleZoneChange (zone) {
      this.selectedZone = zone
      this.fetchKubernetesVersionData()
    },
    fetchKubernetesVersionData () {
      this.kubernetesVersions = []
      const params = {}
      if (!this.isObjectEmpty(this.selectedZone)) {
        params.zoneid = this.selectedZone.id
      }
      this.kubernetesVersionLoading = true
      api('listKubernetesSupportedVersions', params).then(json => {
        const versionObjs = json.listkubernetessupportedversionsresponse.kubernetessupportedversion
        if (this.arrayHasItems(versionObjs)) {
          for (var i = 0; i < versionObjs.length; i++) {
            if (versionObjs[i].state === 'Enabled' && versionObjs[i].isostate === 'Ready') {
              this.kubernetesVersions.push(versionObjs[i])
            }
          }
        }
      }).finally(() => {
        this.kubernetesVersionLoading = false
        if (this.arrayHasItems(this.kubernetesVersions)) {
          this.form.setFieldsValue({
            kubernetesversionid: 0
          })
          this.handleKubernetesVersionChange(this.kubernetesVersions[0])
        }
      })
    },
    handleKubernetesVersionChange (version) {
      this.selectedKubernetesVersion = version
      this.fetchServiceOfferingData()
    },
    fetchServiceOfferingData () {
      this.serviceOfferings = []
      const params = {}
      this.serviceOfferingLoading = true
      api('listServiceOfferings', params).then(json => {
        var items = json.listserviceofferingsresponse.serviceoffering
        var minCpu = 2
        var minMemory = 2048
        if (!this.isObjectEmpty(this.selectedKubernetesVersion)) {
          minCpu = this.selectedKubernetesVersion.mincpunumber
          minMemory = this.selectedKubernetesVersion.minmemory
        }
        if (items != null) {
          for (var i = 0; i < items.length; i++) {
            if (items[i].iscustomized === false &&
                items[i].cpunumber >= minCpu && items[i].memory >= minMemory) {
              this.serviceOfferings.push(items[i])
            }
          }
        }
      }).finally(() => {
        this.serviceOfferingLoading = false
        if (this.arrayHasItems(this.serviceOfferings)) {
          this.form.setFieldsValue({
            serviceofferingid: 0
          })
        }
      })
    },
    fetchNetworkData () {
      const params = {}
      this.networkLoading = true
      api('listNetworks', params).then(json => {
        const listNetworks = json.listnetworksresponse.network
        if (this.arrayHasItems(listNetworks)) {
          this.networks = this.networks.concat(listNetworks)
        }
      }).finally(() => {
        this.networkLoading = false
        if (this.arrayHasItems(this.networks)) {
          this.form.setFieldsValue({
            networkid: 0
          })
        }
      })
    },
    fetchKeyPairData () {
      const params = {}
      this.keyPairLoading = true
      api('listSSHKeyPairs', params).then(json => {
        const listKeyPairs = json.listsshkeypairsresponse.sshkeypair
        if (this.arrayHasItems(listKeyPairs)) {
          for (var i = 0; i < listKeyPairs.length; i++) {
            this.keyPairs.push({
              id: listKeyPairs[i].id,
              name: listKeyPairs[i].name,
              description: listKeyPairs[i].name
            })
          }
        }
      }).finally(() => {
        this.keyPairLoading = false
        if (this.arrayHasItems(this.keyPairs)) {
          this.form.setFieldsValue({
            keypair: 0
          })
        }
      })
    },
    handleSubmit (e) {
      e.preventDefault()
      if (this.loading) return
      const options = {
        scroll: {
          offsetTop: 10
        }
      }
      this.form.validateFieldsAndScroll(options, (err, values) => {
        if (err) {
          return
        }
        this.loading = true
        const params = {
          name: values.name,
          description: values.description,
          zoneid: this.zones[values.zoneid].id,
          kubernetesversionid: this.kubernetesVersions[values.kubernetesversionid].id,
          serviceofferingid: this.serviceOfferings[values.serviceofferingid].id,
          size: values.size
        }
        if (this.isValidValueForKey(values, 'noderootdisksize') && values.noderootdisksize > 0) {
          params.noderootdisksize = values.noderootdisksize
        }
        if (this.isValidValueForKey(values, 'controlnodes') && values.controlnodes > 0) {
          params.controlnodes = values.controlnodes
        }
        if (this.isValidValueForKey(values, 'externalloadbalanceripaddress') && values.externalloadbalanceripaddress !== '') {
          params.externalloadbalanceripaddress = values.externalloadbalanceripaddress
        }
        if (this.isValidValueForKey(values, 'networkid') && this.arrayHasItems(this.networks) && this.networks[values.networkid].id != null) {
          params.networkid = this.networks[values.networkid].id
        }
        if (this.isValidValueForKey(values, 'keypair') && this.arrayHasItems(this.keyPairs) && this.keyPairs[values.keypair].id != null && this.keyPairs[values.keypair].id !== '') {
          params.keypairid = this.keyPairs[values.keypair].id
          params.keypair = this.keyPairs[values.keypair].name
        }
        if (this.usePrivateRegistry) {
          params.dockerregistryusername = values.dockerregistryusername
          params.dockerregistrypassword = values.dockerregistrypassword
          params.dockerregistryurl = values.dockerregistryurl
          params.dockerregistryemail = values.dockerregistryemail
        }

        api('createKubernetesCluster', params).then(json => {
          const jobId = json.createkubernetesclusterresponse.jobid
          this.$pollJob({
            jobId,
            title: this.$t('label.kubernetes.cluster.create'),
            description: values.name,
            loadingMessage: `${this.$t('label.kubernetes.cluster.create')} ${values.name} ${this.$t('label.in.progress')}`,
            catchMessage: this.$t('error.fetching.async.job.result'),
            successMessage: this.$t('message.success.create.kubernetes.cluter') + ' ' + values.name
          })
          this.closeAction()
        }).catch(error => {
          this.$notifyError(error)
        }).finally(() => {
          this.loading = false
        })
      })
    },
    closeAction () {
      this.$emit('close-action')
    }
  }
}
</script>

<style scoped lang="less">
  .form-layout {
    width: 80vw;

    @media (min-width: 700px) {
      width: 550px;
    }
  }
</style>
