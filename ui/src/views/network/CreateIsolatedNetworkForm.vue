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
  <a-spin :spinning="loading">
    <div class="form-layout" v-ctrl-enter="handleSubmit">
      <div class="form">
        <a-form
          :form="form"
          layout="vertical"
          @submit="handleSubmit">
          <a-form-item>
            <tooltip-label slot="label" :title="$t('label.name')" :tooltip="apiParams.name.description"/>
            <a-input
              v-decorator="['name', {
                rules: [{ required: true, message: $t('message.error.name') }]
              }]"
              :placeholder="this.$t('label.name')"
              autoFocus/>
          </a-form-item>
          <a-form-item>
            <tooltip-label slot="label" :title="$t('label.displaytext')" :tooltip="apiParams.displaytext.description"/>
            <a-input
              v-decorator="['displaytext', {
                rules: [{ required: true, message: $t('message.error.display.text') }]
              }]"
              :placeholder="this.$t('label.display.text')"/>
          </a-form-item>
          <a-form-item>
            <tooltip-label slot="label" :title="$t('label.zoneid')" :tooltip="apiParams.zoneid.description"/>
            <a-select
              v-decorator="['zoneid', {
                rules: [
                  {
                    required: true,
                    message: `${this.$t('message.error.select')}`
                  }
                ]
              }]"
              showSearch
              optionFilterProp="children"
              :filterOption="(input, option) => {
                return option.componentOptions.propsData.label.toLowerCase().indexOf(input.toLowerCase()) >= 0
              }"
              :loading="zoneLoading"
              :placeholder="this.$t('label.zoneid')"
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
          <a-form-item v-if="isAdminOrDomainAdmin()">
            <tooltip-label slot="label" :title="$t('label.domainid')" :tooltip="apiParams.domainid.description"/>
            <a-select
              v-decorator="['domainid', {}]"
              showSearch
              optionFilterProp="children"
              :filterOption="(input, option) => {
                return option.componentOptions.children[0].text.toLowerCase().indexOf(input.toLowerCase()) >= 0
              }"
              :loading="domainLoading"
              :placeholder="this.$t('label.domainid')"
              @change="val => { this.handleDomainChange(this.domains[val]) }">
              <a-select-option v-for="(opt, optIndex) in this.domains" :key="optIndex">
                {{ opt.path || opt.name || opt.description }}
              </a-select-option>
            </a-select>
          </a-form-item>
          <a-form-item>
            <tooltip-label slot="label" :title="$t('label.networkofferingid')" :tooltip="apiParams.networkofferingid.description"/>
            <a-select
              v-decorator="['networkofferingid', {
                rules: [
                  {
                    required: true,
                    message: `${this.$t('message.error.select')}`
                  }
                ]
              }]"
              showSearch
              optionFilterProp="children"
              :filterOption="(input, option) => {
                return option.componentOptions.children[0].text.toLowerCase().indexOf(input.toLowerCase()) >= 0
              }"
              :loading="networkOfferingLoading"
              :placeholder="this.$t('label.networkofferingid')"
              @change="val => { this.handleNetworkOfferingChange(this.networkOfferings[val]) }">
              <a-select-option v-for="(opt, optIndex) in this.networkOfferings" :key="optIndex">
                {{ opt.displaytext || opt.name || opt.description }}
              </a-select-option>
            </a-select>
          </a-form-item>
          <a-form-item v-if="!this.isObjectEmpty(this.selectedNetworkOffering) && this.selectedNetworkOffering.specifyvlan">
            <tooltip-label slot="label" :title="$t('label.vlan')" :tooltip="apiParams.vlan.description"/>
            <a-input
              v-decorator="['vlanid', {
                rules: [{ required: true, message: $t('message.please.enter.value') }]
              }]"
              :placeholder="this.$t('label.vlanid')"/>
          </a-form-item>
          <a-form-item v-if="!this.isObjectEmpty(this.selectedNetworkOffering) && this.selectedNetworkOffering.forvpc">
            <tooltip-label slot="label" :title="$t('label.vpcid')" :tooltip="apiParams.vpcid.description"/>
            <a-select
              v-decorator="['vpcid', {
                rules: [
                  {
                    required: true,
                    message: `${this.$t('message.error.select')}`
                  }
                ]
              }]"
              showSearch
              optionFilterProp="children"
              :filterOption="(input, option) => {
                return option.componentOptions.children[0].text.toLowerCase().indexOf(input.toLowerCase()) >= 0
              }"
              :loading="vpcLoading"
              :placeholder="this.$t('label.vpcid')"
              @change="val => { this.selectedVpc = this.vpcs[val] }">
              <a-select-option v-for="(opt, optIndex) in this.vpcs" :key="optIndex">
                {{ opt.name || opt.description }}
              </a-select-option>
            </a-select>
          </a-form-item>
          <a-form-item>
            <tooltip-label slot="label" :title="$t('label.externalid')" :tooltip="apiParams.externalid.description"/>
            <a-input
              v-decorator="['externalid', {}]"
              :placeholder="$t('label.externalid')"/>
          </a-form-item>
          <a-form-item>
            <tooltip-label slot="label" :title="$t('label.gateway')" :tooltip="apiParams.gateway.description"/>
            <a-input
              v-decorator="['gateway', {}]"
              :placeholder="this.$t('label.gateway')"/>
          </a-form-item>
          <a-form-item>
            <tooltip-label slot="label" :title="$t('label.netmask')" :tooltip="apiParams.netmask.description"/>
            <a-input
              v-decorator="['netmask', {}]"
              :placeholder="this.$t('label.netmask')"/>
          </a-form-item>
          <a-form-item v-if="!this.isObjectEmpty(this.selectedNetworkOffering) && !this.selectedNetworkOffering.forvpc">
            <tooltip-label slot="label" :title="$t('label.networkdomain')" :tooltip="apiParams.networkdomain.description"/>
            <a-input
              v-decorator="['networkdomain', {}]"
              :placeholder="this.$t('label.networkdomain')"/>
          </a-form-item>
          <a-form-item v-if="this.accountVisible">
            <tooltip-label slot="label" :title="$t('label.account')" :tooltip="apiParams.account.description"/>
            <a-input
              v-decorator="['account']"
              :placeholder="this.$t('label.account')"/>
          </a-form-item>
          <div :span="24" class="action-button">
            <a-button
              :loading="actionLoading"
              @click="closeAction">
              {{ this.$t('label.cancel') }}
            </a-button>
            <a-button
              :loading="actionLoading"
              ref="submit"
              type="primary"
              @click="handleSubmit">
              {{ this.$t('label.ok') }}
            </a-button>
          </div>
        </a-form>
      </div>
    </div>
  </a-spin>
</template>

<script>
import { api } from '@/api'
import { isAdmin, isAdminOrDomainAdmin } from '@/role'
import ResourceIcon from '@/components/view/ResourceIcon'
import TooltipLabel from '@/components/widgets/TooltipLabel'

export default {
  name: 'CreateIsolatedNetworkForm',
  components: {
    TooltipLabel,
    ResourceIcon
  },
  props: {
    loading: {
      type: Boolean,
      default: false
    },
    vpc: {
      type: Object,
      default: null
    },
    resource: {
      type: Object,
      default: () => { return {} }
    }
  },
  data () {
    return {
      actionLoading: false,
      domains: [],
      domainLoading: false,
      selectedDomain: {},
      zones: [],
      zoneLoading: false,
      selectedZone: {},
      networkOfferings: [],
      networkOfferingLoading: false,
      selectedNetworkOffering: {},
      vpcs: [],
      vpcLoading: false,
      selectedVpc: {},
      accountVisible: isAdminOrDomainAdmin()
    }
  },
  watch: {
    resource (newItem, oldItem) {
      this.fetchData()
    }
  },
  beforeCreate () {
    this.form = this.$form.createForm(this)
    this.apiParams = this.$getApiParams('createNetwork')
  },
  created () {
    this.domains = [
      {
        id: '-1',
        name: ' '
      }
    ]
    this.fetchData()
  },
  methods: {
    fetchData () {
      this.fetchDomainData()
      this.fetchZoneData()
    },
    isAdminOrDomainAdmin () {
      return isAdminOrDomainAdmin()
    },
    isObjectEmpty (obj) {
      return !(obj !== null && obj !== undefined && Object.keys(obj).length > 0 && obj.constructor === Object)
    },
    arrayHasItems (array) {
      return array !== null && array !== undefined && Array.isArray(array) && array.length > 0
    },
    isValidTextValueForKey (obj, key) {
      return key in obj && obj[key] != null && String(obj[key]).length > 0
    },
    fetchZoneData () {
      this.zones = []
      const params = {}
      if (this.resource.zoneid && this.$route.name === 'deployVirtualMachine') {
        params.id = this.resource.zoneid
      }
      params.listAll = true
      params.showicon = true
      this.zoneLoading = true
      api('listZones', params).then(json => {
        for (const i in json.listzonesresponse.zone) {
          const zone = json.listzonesresponse.zone[i]
          if (zone.networktype === 'Advanced' && zone.securitygroupsenabled !== true) {
            this.zones.push(zone)
          }
        }
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
      this.updateVPCCheckAndFetchNetworkOfferingData()
    },
    fetchDomainData () {
      const params = {}
      params.listAll = true
      params.details = 'min'
      this.domainLoading = true
      api('listDomains', params).then(json => {
        const listDomains = json.listdomainsresponse.domain
        this.domains = this.domains.concat(listDomains)
      }).finally(() => {
        this.domainLoading = false
        this.form.setFieldsValue({
          domainid: 0
        })
        this.handleDomainChange(this.domains[0])
      })
    },
    handleDomainChange (domain) {
      this.selectedDomain = domain
      this.accountVisible = domain.id !== '-1'
      if (isAdminOrDomainAdmin()) {
        this.updateVPCCheckAndFetchNetworkOfferingData()
      }
    },
    updateVPCCheckAndFetchNetworkOfferingData () {
      if (this.vpc !== null) { // from VPC section
        this.fetchNetworkOfferingData(true)
      } else { // from guest network section
        var params = {}
        this.networkOfferingLoading = true
        api('listVPCs', params).then(json => {
          const listVPCs = json.listvpcsresponse.vpc
          var vpcAvailable = this.arrayHasItems(listVPCs)
          if (vpcAvailable === false) {
            this.fetchNetworkOfferingData(false)
          } else {
            this.fetchNetworkOfferingData()
          }
        })
      }
    },
    fetchNetworkOfferingData (forVpc) {
      this.networkOfferingLoading = true
      var params = {
        zoneid: this.selectedZone.id,
        guestiptype: 'Isolated',
        supportedServices: 'SourceNat',
        state: 'Enabled'
      }
      if (isAdminOrDomainAdmin() && this.selectedDomain.id !== '-1') { // domain is visible only for admins
        params.domainid = this.selectedDomain.id
      }
      if (!isAdmin()) { // normal user is not aware of the VLANs in the system, so normal user is not allowed to create network with network offerings whose specifyvlan = true
        params.specifyvlan = false
      }
      if (forVpc !== null) {
        params.forvpc = forVpc
      }
      this.networkOfferings = []
      this.selectedNetworkOffering = {}
      api('listNetworkOfferings', params).then(json => {
        this.networkOfferings = json.listnetworkofferingsresponse.networkoffering
      }).catch(error => {
        this.$notifyError(error)
      }).finally(() => {
        this.networkOfferingLoading = false
        if (this.arrayHasItems(this.networkOfferings)) {
          this.form.setFieldsValue({
            networkofferingid: 0
          })
          this.handleNetworkOfferingChange(this.networkOfferings[0])
        }
      })
    },
    handleNetworkOfferingChange (networkOffering) {
      this.selectedNetworkOffering = networkOffering
      if (networkOffering.forvpc) {
        this.fetchVpcData()
      }
    },
    fetchVpcData () {
      this.vpcLoading = true
      var params = {
        listAll: true,
        details: 'min'
      }
      if (this.vpc !== null) {
        params.id = this.vpc.id
      }
      api('listVPCs', params).then(json => {
        this.vpcs = json.listvpcsresponse.vpc
      }).finally(() => {
        this.vpcLoading = false
        if (this.arrayHasItems(this.vpcs)) {
          this.form.setFieldsValue({
            vpcid: 0
          })
          this.selectedVpc = this.vpcs[0]
        }
      })
    },
    handleSubmit (e) {
      if (this.actionLoading) return
      const options = {
        scroll: {
          offsetTop: 10
        }
      }
      this.form.validateFieldsAndScroll(options, (error, values) => {
        if (error) {
          return
        }
        this.actionLoading = true
        var params = {
          zoneId: this.selectedZone.id,
          name: values.name,
          displayText: values.displaytext,
          networkOfferingId: this.selectedNetworkOffering.id
        }
        if (this.isValidTextValueForKey(values, 'gateway')) {
          params.gateway = values.gateway
        }
        if (this.isValidTextValueForKey(values, 'netmask')) {
          params.netmask = values.netmask
        }
        if (this.isValidTextValueForKey(values, 'externalid')) {
          params.externalid = values.externalid
        }
        if (this.isValidTextValueForKey(values, 'vpcid')) {
          params.vpcid = this.selectedVpc.id
        }
        if (this.isValidTextValueForKey(values, 'vlanid')) {
          params.vlan = values.vlanid
        }
        if (this.isValidTextValueForKey(values, 'networkdomain')) {
          params.networkdomain = values.networkdomain
        }
        if ('domainid' in values && values.domainid > 0) {
          params.domainid = this.selectedDomain.id
          if (this.isValidTextValueForKey(values, 'account')) {
            params.account = values.account
          }
        }
        api('createNetwork', params).then(json => {
          this.$notification.success({
            message: 'Network',
            description: this.$t('message.success.create.isolated.network')
          })
          this.$emit('refresh-data')
          this.closeAction()
        }).catch(error => {
          this.$notifyError(error)
        }).finally(() => {
          this.actionLoading = false
        })
      })
    },
    showInput () {
      this.inputVisible = true
      this.$nextTick(function () {
        this.$refs.input.focus()
      })
    },
    resetForm () {
      this.form.setFieldsValue({
      })
      this.tags = []
    },
    closeAction () {
      this.$emit('close-action')
    }
  }
}
</script>

<style lang="less" scoped>
.form-layout {
  .ant-tag {
    margin-bottom: 10px;
  }

  /deep/.custom-time-select .ant-time-picker {
    width: 100%;
  }

  /deep/.ant-divider-horizontal {
    margin-top: 0;
  }
}

.form {
  margin: 10px 0;
}

.tagsTitle {
  font-weight: 500;
  margin-bottom: 12px;
}
</style>
