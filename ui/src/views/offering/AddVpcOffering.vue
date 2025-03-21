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
            autoFocus
            v-decorator="['name', {
              rules: [{ required: true, message: $t('message.error.name') }]
            }]"
            :placeholder="apiParams.name.description"/>
        </a-form-item>
        <a-form-item>
          <tooltip-label slot="label" :title="$t('label.displaytext')" :tooltip="apiParams.displaytext.description"/>
          <a-input
            v-decorator="['displaytext', {
              rules: [{ required: true, message: $t('message.error.description') }]
            }]"
            :placeholder="apiParams.displaytext.description"/>
        </a-form-item>
        <a-form-item>
          <tooltip-label slot="label" :title="$t('label.supportedservices')" :tooltip="apiParams.supportedservices.description"/>
          <div class="supported-services-container" scroll-to="last-child">
            <a-list itemLayout="horizontal" :dataSource="supportedServices">
              <a-list-item slot="renderItem" slot-scope="item">
                <CheckBoxSelectPair
                  v-decorator="['service.'+item.name, {}]"
                  :resourceKey="item.name"
                  :checkBoxLabel="item.description"
                  :selectOptions="item.provider"
                  @handle-checkselectpair-change="handleSupportedServiceChange"/>
              </a-list-item>
            </a-list>
          </div>
        </a-form-item>
        <a-form-item :label="$t('label.service.connectivity.regionlevelvpccapabilitycheckbox')" v-if="connectivityServiceChecked">
          <a-switch v-decorator="['regionlevelvpc', {initialValue: true}]" defaultChecked />
        </a-form-item>
        <a-form-item :label="$t('label.service.connectivity.distributedroutercapabilitycheckbox')" v-if="connectivityServiceChecked">
          <a-switch v-decorator="['distributedrouter', {initialValue: true}]" defaultChecked />
        </a-form-item>
        <a-form-item :label="$t('label.redundantrouter')" v-if="sourceNatServiceChecked">
          <a-switch v-decorator="['redundantrouter', {initialValue: false}]" />
        </a-form-item>
        <a-form-item :label="$t('label.ispublic')" v-if="isAdmin()">
          <a-switch v-decorator="['ispublic', {initialValue: isPublic}]" :defaultChecked="isPublic" @change="val => { isPublic = val }" />
        </a-form-item>
        <a-form-item v-if="!isPublic">
          <tooltip-label slot="label" :title="$t('label.domainid')" :tooltip="apiParams.domainid.description"/>
          <a-select
            mode="multiple"
            v-decorator="['domainid', {
              rules: [
                {
                  required: true,
                  message: $t('message.error.select')
                }
              ]
            }]"
            showSearch
            optionFilterProp="children"
            :filterOption="(input, option) => {
              return option.componentOptions.propsData.label.toLowerCase().indexOf(input.toLowerCase()) >= 0
            }"
            :loading="domainLoading"
            :placeholder="apiParams.domainid.description">
            <a-select-option v-for="(opt, optIndex) in domains" :key="optIndex" :label="opt.path || opt.name || opt.description">
              <resource-icon v-if="opt && opt.icon" :image="opt.icon.base64image" size="1x" style="margin-right: 5px"/>
              <a-icon v-else type="block" style="margin-right: 5px" />
              {{ opt.path || opt.name || opt.description }}
            </a-select-option>
          </a-select>
        </a-form-item>
        <a-form-item>
          <tooltip-label slot="label" :title="$t('label.zoneid')" :tooltip="apiParams.zoneid.description"/>
          <a-select
            id="zone-selection"
            mode="multiple"
            v-decorator="['zoneid', {
              rules: [
                {
                  validator: (rule, value, callback) => {
                    if (value && value.length > 1 && value.indexOf(0) !== -1) {
                      callback($t('message.error.zone.combined'))
                    }
                    callback()
                  }
                }
              ]
            }]"
            showSearch
            optionFilterProp="children"
            :filterOption="(input, option) => {
              return option.componentOptions.propsData.label.toLowerCase().indexOf(input.toLowerCase()) >= 0
            }"
            :loading="zoneLoading"
            :placeholder="apiParams.zoneid.description">
            <a-select-option v-for="(opt, optIndex) in zones" :key="optIndex" :label="opt.name || opt.description">
              <span>
                <resource-icon v-if="opt.icon" :image="opt.icon.base64image" size="1x" style="margin-right: 5px"/>
                <a-icon v-else type="global" style="margin-right: 5px"/>
                {{ opt.name || opt.description }}
              </span>
            </a-select-option>
          </a-select>
        </a-form-item>
        <a-form-item v-if="apiParams.enable">
          <tooltip-label slot="label" :title="$t('label.enable.vpc.offering')" :tooltip="apiParams.enable.description"/>
          <a-switch v-decorator="['enable', {initialValue: false}]" />
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
import { isAdmin } from '@/role'
import CheckBoxSelectPair from '@/components/CheckBoxSelectPair'
import ResourceIcon from '@/components/view/ResourceIcon'
import TooltipLabel from '@/components/widgets/TooltipLabel'

export default {
  name: 'AddVpcOffering',
  components: {
    CheckBoxSelectPair,
    ResourceIcon,
    TooltipLabel
  },
  data () {
    return {
      selectedDomains: [],
      selectedZones: [],
      isConserveMode: true,
      isPublic: true,
      domains: [],
      domainLoading: false,
      zones: [],
      zoneLoading: false,
      loading: false,
      supportedServices: [],
      supportedServiceLoading: false,
      connectivityServiceChecked: false,
      sourceNatServiceChecked: false,
      selectedServiceProviderMap: {}
    }
  },
  beforeCreate () {
    this.form = this.$form.createForm(this)
    this.apiParams = this.$getApiParams('createVPCOffering')
  },
  created () {
    this.zones = [
      {
        id: null,
        name: this.$t('label.all.zone')
      }
    ]
    this.fetchData()
  },
  methods: {
    fetchData () {
      this.fetchDomainData()
      this.fetchZoneData()
      this.fetchSupportedServiceData()
    },
    isAdmin () {
      return isAdmin()
    },
    fetchDomainData () {
      const params = {}
      params.listAll = true
      params.showicon = true
      params.details = 'min'
      this.domainLoading = true
      api('listDomains', params).then(json => {
        const listDomains = json.listdomainsresponse.domain
        this.domains = this.domains.concat(listDomains)
      }).finally(() => {
        this.domainLoading = false
      })
    },
    fetchZoneData () {
      const params = {}
      params.listAll = true
      params.showicon = true
      this.zoneLoading = true
      api('listZones', params).then(json => {
        const listZones = json.listzonesresponse.zone
        this.zones = this.zones.concat(listZones)
      }).finally(() => {
        this.zoneLoading = false
      })
    },
    fetchSupportedServiceData () {
      this.supportedServices = []
      this.supportedServices.push({
        name: 'Dhcp',
        provider: [
          { name: 'VpcVirtualRouter' }
        ]
      })
      this.supportedServices.push({
        name: 'Dns',
        provider: [{ name: 'VpcVirtualRouter' }]
      })
      this.supportedServices.push({
        name: 'Lb',
        provider: [
          { name: 'VpcVirtualRouter' },
          { name: 'InternalLbVm' }
        ]
      })
      this.supportedServices.push({
        name: 'Gateway',
        provider: [
          { name: 'VpcVirtualRouter' },
          { name: 'BigSwitchBcf' }
        ]
      })
      this.supportedServices.push({
        name: 'StaticNat',
        provider: [
          { name: 'VpcVirtualRouter' },
          { name: 'BigSwitchBcf' }
        ]
      })
      this.supportedServices.push({
        name: 'SourceNat',
        provider: [
          { name: 'VpcVirtualRouter' },
          { name: 'BigSwitchBcf' }
        ]
      })
      this.supportedServices.push({
        name: 'NetworkACL',
        provider: [
          { name: 'VpcVirtualRouter' },
          { name: 'BigSwitchBcf' }
        ]
      })
      this.supportedServices.push({
        name: 'PortForwarding',
        provider: [{ name: 'VpcVirtualRouter' }]
      })
      this.supportedServices.push({
        name: 'UserData',
        provider: [
          { name: 'VpcVirtualRouter' },
          { name: 'ConfigDrive' }
        ]
      })
      this.supportedServices.push({
        name: 'Vpn',
        provider: [
          { name: 'VpcVirtualRouter' },
          { name: 'BigSwitchBcf' }
        ]
      })
      this.supportedServices.push({
        name: 'Connectivity',
        provider: [
          { name: 'BigSwitchBcf' },
          { name: 'NiciraNvp' },
          { name: 'Ovs' },
          { name: 'JuniperContrailVpcRouter' }
        ]
      })
      for (var i in this.supportedServices) {
        var serviceName = this.supportedServices[i].name
        var serviceDisplayName = serviceName
        // Sanitize names
        this.supportedServices[i].description = serviceDisplayName
      }
    },
    handleSupportedServiceChange (service, checked, provider) {
      if (service === 'Connectivity') {
        this.connectivityServiceChecked = checked
      }
      if (service === 'SourceNat') {
        this.sourceNatServiceChecked = checked
      }
      if (checked && provider != null & provider !== undefined) {
        this.selectedServiceProviderMap[service] = provider
      } else {
        delete this.selectedServiceProviderMap[service]
      }
    },
    handleSubmit (e) {
      e.preventDefault()
      if (this.loading) return
      this.form.validateFieldsAndScroll((err, values) => {
        if (err) {
          return
        }
        this.loading = true
        var params = {}
        params.name = values.name
        params.displaytext = values.displaytext
        if (values.ispublic !== true) {
          var domainIndexes = values.domainid
          var domainId = null
          if (domainIndexes && domainIndexes.length > 0) {
            var domainIds = []
            for (var i = 0; i < domainIndexes.length; i++) {
              domainIds = domainIds.concat(this.domains[domainIndexes[i]].id)
            }
            domainId = domainIds.join(',')
          }
          if (domainId) {
            params.domainid = domainId
          }
        }
        var zoneIndexes = values.zoneid
        var zoneId = null
        if (zoneIndexes && zoneIndexes.length > 0) {
          var zoneIds = []
          for (var j = 0; j < zoneIndexes.length; j++) {
            zoneIds = zoneIds.concat(this.zones[zoneIndexes[j]].id)
          }
          zoneId = zoneIds.join(',')
        }
        if (zoneId) {
          params.zoneid = zoneId
        }
        if (this.selectedServiceProviderMap != null) {
          var supportedServices = Object.keys(this.selectedServiceProviderMap)
          params.supportedservices = supportedServices.join(',')
          for (var k in supportedServices) {
            params['serviceProviderList[' + k + '].service'] = supportedServices[k]
            params['serviceProviderList[' + k + '].provider'] = this.selectedServiceProviderMap[supportedServices[k]]
          }
          var serviceCapabilityIndex = 0
          if (supportedServices.includes('Connectivity')) {
            if (values.regionlevelvpc === true) {
              params['serviceCapabilityList[' + serviceCapabilityIndex + '].service'] = 'Connectivity'
              params['serviceCapabilityList[' + serviceCapabilityIndex + '].capabilitytype'] = 'RegionLevelVpc'
              params['serviceCapabilityList[' + serviceCapabilityIndex + '].capabilityvalue'] = true
              serviceCapabilityIndex++
            }
            if (values.distributedrouter === true) {
              params['serviceCapabilityList[' + serviceCapabilityIndex + '].service'] = 'Connectivity'
              params['serviceCapabilityList[' + serviceCapabilityIndex + '].capabilitytype'] = 'DistributedRouter'
              params['serviceCapabilityList[' + serviceCapabilityIndex + '].capabilityvalue'] = true
              serviceCapabilityIndex++
            }
          }
          if (supportedServices.includes('SourceNat') && values.redundantrouter === true) {
            params['serviceCapabilityList[' + serviceCapabilityIndex + '].service'] = 'SourceNat'
            params['serviceCapabilityList[' + serviceCapabilityIndex + '].capabilitytype'] = 'RedundantRouter'
            params['serviceCapabilityList[' + serviceCapabilityIndex + '].capabilityvalue'] = true
            serviceCapabilityIndex++
          }
        } else {
          params.supportedservices = ''
        }
        if (values.enable) {
          params.enable = values.enable
        }
        api('createVPCOffering', params).then(json => {
          this.$message.success(`${this.$t('message.create.vpc.offering')}: ` + values.name)
          this.$emit('refresh-data')
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

<style scoped lang="scss">
  .form-layout {
    width: 80vw;

    @media (min-width: 800px) {
      width: 500px;
    }
  }

  .supported-services-container {
    height: 250px;
    overflow: auto;
  }
</style>
