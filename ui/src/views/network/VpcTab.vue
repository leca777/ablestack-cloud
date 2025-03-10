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
  <a-spin :spinning="fetchLoading">
    <a-tabs
      :activeKey="currentTab"
      :tabPosition="device === 'mobile' ? 'top' : 'left'"
      :animated="false"
      @change="handleChangeTab">
      <a-tab-pane :tab="$t('label.details')" key="details">
        <DetailsTab :resource="resource" :loading="loading" />
      </a-tab-pane>
      <a-tab-pane :tab="$t('label.networks')" key="tier">
        <VpcTiersTab :resource="resource" :loading="loading" />
      </a-tab-pane>
      <a-tab-pane :tab="$t('label.public.ips')" key="ip" v-if="'listPublicIpAddresses' in $store.getters.apis">
        <IpAddressesTab :resource="resource" :loading="loading" />
      </a-tab-pane>
      <a-tab-pane :tab="$t('label.network.acl.lists')" key="acl" v-if="'listNetworkACLLists' in $store.getters.apis">
        <a-button
          type="dashed"
          icon="plus"
          style="width: 100%"
          :disabled="!('createNetworkACLList' in $store.getters.apis)"
          @click="() => handleOpenModals('networkAcl')">
          {{ $t('label.add.network.acl.list') }}
        </a-button>
        <a-table
          class="table"
          size="small"
          :columns="networkAclsColumns"
          :dataSource="networkAcls"
          :rowKey="item => item.id"
          :pagination="false"
        >
          <template slot="name" slot-scope="text, item">
            <router-link :to="{ path: '/acllist/' + item.id }">
              {{ text }}
            </router-link>
          </template>
        </a-table>
        <a-pagination
          class="row-element pagination"
          size="small"
          :current="page"
          :pageSize="pageSize"
          :total="itemCounts.networkAcls"
          :showTotal="total => `${$t('label.total')} ${total} ${$t('label.items')}`"
          :pageSizeOptions="['10', '20', '40', '80', '100']"
          @change="changePage"
          @showSizeChange="changePageSize"
          showSizeChanger>
          <template slot="buildOptionText" slot-scope="props">
            <span>{{ props.value }} / {{ $t('label.page') }}</span>
          </template>
        </a-pagination>
        <a-modal
          v-model="modals.networkAcl"
          :title="$t('label.add.acl.list')"
          :footer="null"
          :maskClosable="false"
          :closable="true"
          @cancel="modals.networkAcl = fetchAclList"
          v-ctrl-enter="handleNetworkAclFormSubmit">
          <a-form @submit.prevent="handleNetworkAclFormSubmit" :form="networkAclForm">
            <a-form-item :label="$t('label.add.list.name')">
              <a-input
                v-decorator="['name', {rules: [{ required: true, message: `${$t('label.required')}` }]}]"
                autoFocus></a-input>
            </a-form-item>
            <a-form-item :label="$t('label.description')">
              <a-input v-decorator="['description', {rules: [{ required: true, message: `${$t('label.required')}` }]}]"></a-input>
            </a-form-item>

            <div :span="24" class="action-button">
              <a-button @click="modals.networkAcl = false">{{ $t('label.cancel') }}</a-button>
              <a-button type="primary" @click="handleNetworkAclFormSubmit">{{ $t('label.ok') }}</a-button>
            </div>
          </a-form>
        </a-modal>
      </a-tab-pane>
      <a-tab-pane :tab="$t('label.private.gateway')" key="pgw" v-if="'listPrivateGateways' in $store.getters.apis">
        <a-button
          type="dashed"
          icon="plus"
          style="width: 100%"
          :disabled="!('createPrivateGateway' in $store.getters.apis)"
          @click="() => handleOpenModals('privateGateways')">{{ $t('label.add.private.gateway') }}</a-button>
        <a-table
          class="table"
          size="small"
          :columns="privateGatewaysColumns"
          :dataSource="privateGateways"
          :rowKey="item => item.id"
          :pagination="false"
        >
          <template slot="ipaddress" slot-scope="text, item">
            <router-link :to="{ path: '/privategw/' + item.id }">{{ text }}</router-link>
          </template>
          <template slot="state" slot-scope="text, item">
            <status :text="item.state" displayText></status>
          </template>
        </a-table>
        <a-pagination
          class="row-element pagination"
          size="small"
          :current="page"
          :pageSize="pageSize"
          :total="itemCounts.privateGateways"
          :showTotal="total => `${$t('label.total')} ${total} ${$t('label.items')}`"
          :pageSizeOptions="['10', '20', '40', '80', '100']"
          @change="changePage"
          @showSizeChange="changePageSize"
          showSizeChanger>
          <template slot="buildOptionText" slot-scope="props">
            <span>{{ props.value }} / {{ $t('label.page') }}</span>
          </template>
        </a-pagination>

        <a-modal
          v-model="modals.gateway"
          :title="$t('label.add.new.gateway')"
          :maskClosable="false"
          :closable="true"
          :footer="null"
          @cancel="modals.gateway = false"
          v-ctrl-enter="handleGatewayFormSubmit">
          <a-spin :spinning="modals.gatewayLoading">
            <p>{{ $t('message.add.new.gateway.to.vpc') }}</p>
            <a-form @submit.prevent="handleGatewayFormSubmit" :form="gatewayForm">
              <a-form-item :label="$t('label.physicalnetworkid')">
                <a-select
                  v-decorator="['physicalnetwork']"
                  autoFocus
                  showSearch
                  optionFilterProp="children"
                  :filterOption="(input, option) => {
                    return option.componentOptions.children[0].text.toLowerCase().indexOf(input.toLowerCase()) >= 0
                  }" >
                  <a-select-option v-for="item in physicalnetworks" :key="item.id" :value="item.id">
                    {{ item.name }}
                  </a-select-option>
                </a-select>
              </a-form-item>
              <a-form-item :label="$t('label.vlan')" :required="true">
                <a-input
                  :placeholder="placeholders.vlan"
                  v-decorator="['vlan', {rules: [{ required: true, message: `${$t('label.required')}` }]}]"
                ></a-input>
              </a-form-item>
              <a-form-item
                :label="$t('label.bypassvlanoverlapcheck')"
                v-if="$store.getters.apis.createPrivateGateway && $store.getters.apis.createPrivateGateway.params.filter(x => x.name === 'bypassvlanoverlapcheck').length > 0" >
                <a-checkbox
                  v-decorator="['bypassvlanoverlapcheck']"
                ></a-checkbox>
              </a-form-item>
              <a-form-item :label="$t('label.publicip')" :required="true">
                <a-input
                  :placeholder="placeholders.ipaddress"
                  v-decorator="['ipaddress', {rules: [{ required: true, message: `${$t('label.required')}` }]}]"
                ></a-input>
              </a-form-item>
              <a-form-item :label="$t('label.gateway')" :required="true">
                <a-input
                  :placeholder="placeholders.gateway"
                  v-decorator="['gateway', {rules: [{ required: true, message: `${$t('label.required')}` }]}]"
                ></a-input>
              </a-form-item>
              <a-form-item :label="$t('label.netmask')" :required="true">
                <a-input
                  :placeholder="placeholders.netmask"
                  v-decorator="['netmask', {rules: [{ required: true, message: `${$t('label.required')}` }]}]"
                ></a-input>
              </a-form-item>
              <a-form-item :label="$t('label.sourcenat')">
                <a-checkbox v-decorator="['nat']"></a-checkbox>
              </a-form-item>
              <a-form-item :label="$t('label.aclid')">
                <a-select
                  v-decorator="['acl']"
                  showSearch
                  optionFilterProp="children"
                  :filterOption="(input, option) => {
                    return option.componentOptions.children[0].text.toLowerCase().indexOf(input.toLowerCase()) >= 0
                  }" >
                  <a-select-option v-for="item in networkAcls" :key="item.id" :value="item.id">
                    <strong>{{ item.name }}</strong> ({{ item.description }})
                  </a-select-option>
                </a-select>
              </a-form-item>

              <div :span="24" class="action-button">
                <a-button @click="modals.gateway = false">{{ $t('label.cancel') }}</a-button>
                <a-button type="primary" @click="handleGatewayFormSubmit">{{ $t('label.ok') }}</a-button>
              </div>
            </a-form>
          </a-spin>
        </a-modal>
      </a-tab-pane>
      <a-tab-pane :tab="$t('label.vpn.gateway')" key="vpngw" v-if="'listVpnGateways' in $store.getters.apis">
        <a-button
          v-if="vpnGateways.length === 0"
          type="dashed"
          icon="plus"
          style="width: 100%"
          :disabled="!('createVpnGateway' in $store.getters.apis)"
          @click="handleCreateVpnGateway">
          {{ $t('label.create.site.vpn.gateway') }}
        </a-button>
        <a-list class="list">
          <a-list-item v-for="item in vpnGateways" :key="item.id">
            <div class="list__item">
              <div class="list__col">
                <div class="list__label">{{ $t('label.ip') }}</div>
                <div>
                  <router-link :to="{ path: '/s2svpn/' + item.id }">
                    {{ item.publicip }}
                  </router-link>
                </div>
              </div>
            </div>
          </a-list-item>
        </a-list>
      </a-tab-pane>
      <a-tab-pane :tab="$t('label.vpn.connection')" key="vpnc" v-if="'listVpnConnections' in $store.getters.apis">
        <a-button
          type="dashed"
          icon="plus"
          style="width: 100%"
          :disabled="!('createVpnConnection' in $store.getters.apis)"
          @click="handleOpenModals('vpnConnection')">
          {{ $t('label.create.site.vpn.connection') }}
        </a-button>
        <a-table
          class="table"
          size="small"
          :columns="vpnConnectionsColumns"
          :dataSource="vpnConnections"
          :pagination="false"
          :rowKey="record => record.id">
          <a slot="publicip" slot-scope="text, record" href="javascript:;">
            <router-link :to="{ path: '/s2svpnconn/' + record.id }">
              {{ text }}
            </router-link>
          </a>
          <template slot="state" slot-scope="text">
            <status :text="text ? text : ''" displayText />
          </template>
        </a-table>
        <a-pagination
          class="row-element pagination"
          size="small"
          :current="page"
          :pageSize="pageSize"
          :total="itemCounts.vpnConnections"
          :showTotal="total => `${$t('label.total')} ${total} ${$t('label.items')}`"
          :pageSizeOptions="['10', '20', '40', '80', '100']"
          @change="changePage"
          @showSizeChange="changePageSize"
          showSizeChanger>
          <template slot="buildOptionText" slot-scope="props">
            <span>{{ props.value }} / {{ $t('label.page') }}</span>
          </template>
        </a-pagination>
        <a-modal
          v-model="modals.vpnConnection"
          :title="$t('label.create.vpn.connection')"
          :maskClosable="false"
          :closable="true"
          :footer="null"
          @cancel="modals.vpnConnection = false"
          v-ctrl-enter="handleVpnConnectionFormSubmit">
          <a-spin :spinning="modals.vpnConnectionLoading">
            <a-form @submit.prevent="handleVpnConnectionFormSubmit" :form="vpnConnectionForm">
              <a-form-item :label="$t('label.vpncustomergatewayid')">
                <a-select
                  v-decorator="['vpncustomergateway']"
                  autoFocus
                  showSearch
                  optionFilterProp="children"
                  :filterOption="(input, option) => {
                    return option.componentOptions.children[0].text.toLowerCase().indexOf(input.toLowerCase()) >= 0
                  }" >
                  <a-select-option v-for="item in vpncustomergateways" :key="item.id" :value="item.id">
                    {{ item.name }}
                  </a-select-option>
                </a-select>
              </a-form-item>
              <a-form-item :label="$t('label.passive')">
                <a-checkbox v-decorator="['passive']"></a-checkbox>
              </a-form-item>

              <div :span="24" class="action-button">
                <a-button @click="modals.vpnConnection = false">{{ $t('label.cancel') }}</a-button>
                <a-button type="primary" htmlType="submit" @click="handleVpnConnectionFormSubmit">{{ $t('label.ok') }}</a-button>
              </div>
            </a-form>
          </a-spin>
        </a-modal>
      </a-tab-pane>
      <a-tab-pane :tab="$t('label.virtual.routers')" key="vr" v-if="$store.getters.userInfo.roletype === 'Admin'">
        <RoutersTab :resource="resource" :loading="loading" />
      </a-tab-pane>
      <a-tab-pane :tab="$t('label.annotations')" key="comments" v-if="'listAnnotations' in $store.getters.apis">
        <AnnotationsTab
          :resource="resource"
          :items="annotations">
        </AnnotationsTab>
      </a-tab-pane>
    </a-tabs>
  </a-spin>
</template>

<script>
import { api } from '@/api'
import { mixinDevice } from '@/utils/mixin.js'
import DetailsTab from '@/components/view/DetailsTab'
import Status from '@/components/widgets/Status'
import IpAddressesTab from './IpAddressesTab'
import RoutersTab from './RoutersTab'
import VpcTiersTab from './VpcTiersTab'
import AnnotationsTab from '@/components/view/AnnotationsTab'

export default {
  name: 'VpcTab',
  components: {
    DetailsTab,
    Status,
    IpAddressesTab,
    RoutersTab,
    VpcTiersTab,
    AnnotationsTab
  },
  mixins: [mixinDevice],
  props: {
    resource: {
      type: Object,
      required: true
    },
    loading: {
      type: Boolean,
      default: false
    }
  },
  data () {
    return {
      fetchLoading: false,
      privateGateways: [],
      vpnGateways: [],
      vpnConnections: [],
      networkAcls: [],
      modals: {
        gateway: false,
        gatewayLoading: false,
        vpnConnection: false,
        vpnConnectionLoading: false,
        networkAcl: false
      },
      placeholders: {
        vlan: null,
        ipaddress: null,
        gateway: null,
        netmask: null
      },
      physicalnetworks: [],
      vpncustomergateways: [],
      privateGatewaysColumns: [
        {
          title: this.$t('label.ip'),
          dataIndex: 'ipaddress',
          scopedSlots: { customRender: 'ipaddress' }
        },
        {
          title: this.$t('label.state'),
          dataIndex: 'state',
          scopedSlots: { customRender: 'state' }
        },
        {
          title: this.$t('label.gateway'),
          dataIndex: 'gateway'
        },
        {
          title: this.$t('label.netmask'),
          dataIndex: 'netmask'
        },
        {
          title: this.$t('label.vlan'),
          dataIndex: 'vlan'
        }
      ],
      vpnConnectionsColumns: [
        {
          title: this.$t('label.ip'),
          dataIndex: 'publicip',
          scopedSlots: { customRender: 'publicip' }
        },
        {
          title: this.$t('label.state'),
          dataIndex: 'state',
          scopedSlots: { customRender: 'state' }
        },
        {
          title: this.$t('label.gateway'),
          dataIndex: 'gateway'
        },
        {
          title: this.$t('label.ipsecpsk'),
          dataIndex: 'ipsecpsk'
        }
      ],
      networkAclsColumns: [
        {
          title: this.$t('label.name'),
          dataIndex: 'name',
          scopedSlots: { customRender: 'name' }
        },
        {
          title: this.$t('label.description'),
          dataIndex: 'description'
        }
      ],
      itemCounts: {
        privateGateways: 0,
        vpnConnections: 0,
        networkAcls: 0
      },
      page: 1,
      pageSize: 10,
      currentTab: 'details',
      annotations: []
    }
  },
  beforeCreate () {
    this.gatewayForm = this.$form.createForm(this)
    this.vpnConnectionForm = this.$form.createForm(this)
    this.networkAclForm = this.$form.createForm(this)
  },
  watch: {
    loading (newData, oldData) {
      if (!newData && this.resource.id) {
        this.handleFetchData()
      }
    },
    $route: function (newItem, oldItem) {
      this.setCurrentTab()
    }
  },
  created () {
    this.handleFetchData()
    this.setCurrentTab()
  },
  methods: {
    setCurrentTab () {
      this.currentTab = this.$route.query.tab ? this.$route.query.tab : 'details'
    },
    handleChangeTab (e) {
      this.currentTab = e
      this.page = 1
      this.pageSize = 10
      this.handleFetchData()
      const query = Object.assign({}, this.$route.query)
      query.tab = e
      history.replaceState(
        {},
        null,
        '#' + this.$route.path + '?' + Object.keys(query).map(key => {
          return (
            encodeURIComponent(key) + '=' + encodeURIComponent(query[key])
          )
        }).join('&')
      )
    },
    handleFetchData () {
      switch (this.currentTab) {
        case 'pgw':
          this.fetchPrivateGateways()
          break
        case 'vpngw':
          this.fetchVpnGateways()
          break
        case 'vpnc':
          this.fetchVpnConnections()
          break
        case 'acl':
          this.fetchAclList()
          break
        case 'comments':
          this.fetchComments()
          break
      }
    },
    fetchComments () {
      this.fetchLoading = true
      api('listAnnotations', { entityid: this.resource.id, entitytype: 'VPC', annotationfilter: 'all' }).then(json => {
        if (json.listannotationsresponse && json.listannotationsresponse.annotation) {
          this.annotations = json.listannotationsresponse.annotation
        }
      }).catch(error => {
        this.$notifyError(error)
      }).finally(() => {
        this.fetchLoading = false
      })
    },
    fetchPrivateGateways () {
      this.fetchLoading = true
      api('listPrivateGateways', {
        vpcid: this.resource.id,
        listAll: true,
        page: this.page,
        pagesize: this.pageSize
      }).then(json => {
        this.privateGateways = json.listprivategatewaysresponse.privategateway
        this.itemCounts.privateGateways = json.listprivategatewaysresponse.count
      }).catch(error => {
        this.$notifyError(error)
      }).finally(() => {
        this.fetchLoading = false
      })
    },
    fetchVpnGateways () {
      this.fetchLoading = true
      api('listVpnGateways', {
        vpcid: this.resource.id,
        listAll: true
      }).then(json => {
        this.vpnGateways = json.listvpngatewaysresponse.vpngateway ? json.listvpngatewaysresponse.vpngateway : []
      }).catch(error => {
        this.$notifyError(error)
      }).finally(() => {
        this.fetchLoading = false
      })
    },
    fetchVpnConnections () {
      this.fetchLoading = true
      api('listVpnConnections', {
        vpcid: this.resource.id,
        listAll: true,
        page: this.page,
        pagesize: this.pageSize
      }).then(json => {
        this.vpnConnections = json.listvpnconnectionsresponse.vpnconnection
        this.itemCounts.vpnConnections = json.listvpnconnectionsresponse.count
      }).catch(error => {
        this.$notifyError(error)
      }).finally(() => {
        this.fetchLoading = false
      })
    },
    fetchAclList () {
      this.fetchLoading = true
      api('listNetworkACLLists', {
        vpcid: this.resource.id,
        listAll: true,
        page: this.page,
        pagesize: this.pageSize
      }).then(json => {
        this.networkAcls = json.listnetworkacllistsresponse.networkacllist
        this.itemCounts.networkAcls = json.listnetworkacllistsresponse.count
        if (this.modals.gateway === true) {
          this.$nextTick(() => {
            this.gatewayForm.setFieldsValue({ acl: this.networkAcls[0].id })
          })
        }
      }).catch(error => {
        this.$notifyError(error)
      }).finally(() => {
        this.fetchLoading = false
      })
    },
    fetchPhysicalNetworks () {
      this.modals.gatewayLoading = true
      api('listPhysicalNetworks', { zoneid: this.resource.zoneid }).then(json => {
        this.physicalnetworks = json.listphysicalnetworksresponse.physicalnetwork
        if (this.modals.gateway === true) {
          this.$nextTick(() => {
            this.gatewayForm.setFieldsValue({ physicalnetwork: this.physicalnetworks[0].id })
          })
        }
      }).catch(error => {
        this.$notifyError(error)
      }).finally(() => {
        this.modals.gatewayLoading = false
      })
    },
    fetchVpnCustomerGateways () {
      this.modals.vpnConnectionLoading = true
      api('listVpnCustomerGateways', { listAll: true }).then(json => {
        this.vpncustomergateways = json.listvpncustomergatewaysresponse.vpncustomergateway
        if (this.modals.vpnConnection === true) {
          this.$nextTick(() => {
            this.vpnConnectionForm.setFieldsValue({ vpncustomergateway: this.vpncustomergateways[0].id })
          })
        }
      }).catch(error => {
        this.$notifyError(error)
      }).finally(() => {
        this.modals.vpnConnectionLoading = false
      })
    },
    handleOpenModals (e) {
      switch (e) {
        case 'privateGateways':
          this.modals.gateway = true
          this.gatewayForm.resetFields()
          this.fetchAclList()
          this.fetchPhysicalNetworks()
          break
        case 'vpnConnection':
          this.modals.vpnConnection = true
          this.vpnConnectionForm.resetFields()
          this.fetchVpnCustomerGateways()
          this.fetchVpnGateways()
          break
        case 'networkAcl':
          this.modals.networkAcl = true
          this.networkAclForm.resetFields()
          break
      }
    },
    handleGatewayFormSubmit () {
      if (this.modals.gatewayLoading) return
      this.modals.gatewayLoading = true

      this.gatewayForm.validateFieldsAndScroll(errors => {
        if (errors) {
          this.modals.gatewayLoading = false
          return
        }

        const data = this.gatewayForm.getFieldsValue()
        const params = {
          sourcenatsupported: data.nat,
          physicalnetworkid: data.physicalnetwork,
          vpcid: this.resource.id,
          ipaddress: data.ipaddress,
          gateway: data.gateway,
          netmask: data.netmask,
          vlan: data.vlan,
          aclid: data.acl
        }
        if (data.bypassvlanoverlapcheck) {
          params.bypassvlanoverlapcheck = data.bypassvlanoverlapcheck
        }

        api('createPrivateGateway', params).then(response => {
          this.$pollJob({
            jobId: response.createprivategatewayresponse.jobid,
            title: this.$t('message.success.add.private.gateway'),
            description: this.resource.id,
            successMethod: () => {
              this.modals.gateway = false
              this.handleFetchData()
            },
            errorMessage: this.$t('message.add.private.gateway.failed'),
            errorMethod: () => {
              this.modals.gateway = false
              this.handleFetchData()
            },
            loadingMessage: this.$t('message.add.private.gateway.processing'),
            catchMessage: this.$t('error.fetching.async.job.result'),
            catchMethod: () => {
              this.modals.gateway = false
              this.handleFetchData()
            }
          })
        }).catch(error => {
          this.$notifyError(error)
        }).finally(() => {
          this.modals.gatewayLoading = false
          this.modals.gateway = false
          this.handleFetchData()
        })
      })
    },
    handleVpnConnectionFormSubmit () {
      if (this.fetchLoading) return
      this.fetchLoading = true
      this.modals.vpnConnection = false

      this.vpnConnectionForm.validateFieldsAndScroll((errors, values) => {
        if (errors) {
          this.fetchLoading = false
          return
        }

        api('createVpnConnection', {
          s2svpngatewayid: this.vpnGateways[0].id,
          s2scustomergatewayid: values.vpncustomergateway,
          passive: values.passive ? values.passive : false
        }).then(response => {
          this.$pollJob({
            jobId: response.createvpnconnectionresponse.jobid,
            title: this.$t('label.vpn.connection'),
            description: this.vpnGateways[0].id,
            successMethod: () => {
              this.fetchVpnConnections()
              this.fetchLoading = false
            },
            errorMessage: this.$t('message.add.vpn.connection.failed'),
            errorMethod: () => {
              this.fetchVpnConnections()
              this.fetchLoading = false
            },
            loadingMessage: this.$t('message.add.vpn.connection.processing'),
            catchMessage: this.$t('error.fetching.async.job.result'),
            catchMethod: () => {
              this.fetchVpnConnections()
              this.fetchLoading = false
            }
          })
        }).catch(error => {
          this.$notifyError(error)
        }).finally(() => {
          this.fetchVpnConnections()
          this.fetchLoading = false
        })
      })
    },
    handleNetworkAclFormSubmit () {
      if (this.fetchLoading) return
      this.fetchLoading = true
      this.modals.networkAcl = false

      this.networkAclForm.validateFieldsAndScroll((errors, values) => {
        if (errors) {
          this.fetchLoading = false
        }

        api('createNetworkACLList', {
          name: values.name,
          description: values.description,
          vpcid: this.resource.id
        }).then(response => {
          this.$pollJob({
            jobId: response.createnetworkacllistresponse.jobid,
            title: this.$t('message.success.add.network.acl'),
            description: values.name || values.description,
            successMethod: () => {
              this.fetchLoading = false
            },
            errorMessage: this.$t('message.add.network.acl.failed'),
            errorMethod: () => {
              this.fetchLoading = false
            },
            loadingMessage: this.$t('message.add.network.acl.processing'),
            catchMessage: this.$t('error.fetching.async.job.result'),
            catchMethod: () => {
              this.fetchLoading = false
            }
          })
        }).catch(error => {
          this.$notifyError(error)
        }).finally(() => {
          this.fetchLoading = false
          this.fetchAclList()
        })
      })
    },
    handleCreateVpnGateway () {
      this.fetchLoading = true
      api('createVpnGateway', {
        vpcid: this.resource.id
      }).then(response => {
        this.$pollJob({
          jobId: response.createvpngatewayresponse.jobid,
          title: this.$t('message.success.add.vpn.gateway'),
          description: this.resource.id,
          successMethod: () => {
            this.fetchLoading = false
          },
          errorMessage: this.$t('message.add.vpn.gateway.failed'),
          errorMethod: () => {
            this.fetchLoading = false
          },
          loadingMessage: this.$t('message.add.vpn.gateway.processing'),
          catchMessage: this.$t('error.fetching.async.job.result'),
          catchMethod: () => {
            this.fetchLoading = false
          }
        })
      }).catch(error => {
        this.$notifyError(error)
      }).finally(() => {
        this.fetchLoading = false
        this.handleFetchData()
      })
    },
    changePage (page, pageSize) {
      this.page = page
      this.pageSize = pageSize
      this.handleFetchData()
    },
    changePageSize (currentPage, pageSize) {
      this.page = currentPage
      this.pageSize = pageSize
      this.handleFetchData()
    }
  }
}
</script>

<style lang="scss" scoped>
.list {

  &__item,
  &__row {
    display: flex;
    flex-wrap: wrap;
    width: 100%;
  }

  &__item {
    margin-bottom: -20px;
  }

  &__col {
    flex: 1;
    margin-right: 20px;
    margin-bottom: 20px;
  }

  &__label {
    font-weight: bold;
  }

}

.pagination {
  margin-top: 20px;
}

.table {
  margin-top: 20px;
  overflow-y: auto;
}
</style>
