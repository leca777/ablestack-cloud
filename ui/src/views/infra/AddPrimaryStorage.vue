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
      <a-form :form="form" layout="vertical">
        <a-form-item>
          <tooltip-label slot="label" :title="$t('label.scope')" :tooltip="apiParams.scope.description"/>
          <a-select
            v-decorator="['scope', { initialValue: 'cluster' }]"
            @change="val => { this.scope = val }"
            autoFocus
            showSearch
            optionFilterProp="children"
            :filterOption="(input, option) => {
              return option.componentOptions.children[0].text.toLowerCase().indexOf(input.toLowerCase()) >= 0
            }" >
            <a-select-option :value="'cluster'"> {{ $t('label.clusterid') }} </a-select-option>
            <a-select-option :value="'zone'"> {{ $t('label.zoneid') }} </a-select-option>
          </a-select>
        </a-form-item>
        <div v-if="this.scope === 'zone'">
          <a-form-item>
            <tooltip-label slot="label" :title="$t('label.hypervisor')" :tooltip="apiParams.hypervisor.description"/>
            <a-select
              v-decorator="['hypervisor', { initialValue: hypervisors[0]}]"
              @change="val => this.selectedHypervisor = val"
              showSearch
              optionFilterProp="children"
              :filterOption="(input, option) => {
                return option.componentOptions.children[0].text.toLowerCase().indexOf(input.toLowerCase()) >= 0
              }" >
              <a-select-option :value="hypervisor" v-for="(hypervisor, idx) in hypervisors" :key="idx">
                {{ hypervisor }}
              </a-select-option>
            </a-select>
          </a-form-item>
        </div>
        <a-form-item>
          <tooltip-label slot="label" :title="$t('label.zoneid')" :tooltip="apiParams.zoneid.description"/>
          <a-select
            v-decorator="['zone', { initialValue: this.zoneSelected, rules: [{ required: true, message: `${$t('label.required')}`}] }]"
            @change="val => changeZone(val)"
            showSearch
            optionFilterProp="children"
            :filterOption="(input, option) => {
              return option.componentOptions.propsData.label.toLowerCase().indexOf(input.toLowerCase()) >= 0
            }" >
            <a-select-option :value="zone.id" v-for="(zone) in zones" :key="zone.id" :label="zone.name">
              <span>
                <resource-icon v-if="zone.icon" :image="zone.icon.base64image" size="1x" style="margin-right: 5px"/>
                <a-icon v-else type="global" style="margin-right: 5px" />
                {{ zone.name }}
              </span>
            </a-select-option>
          </a-select>
        </a-form-item>
        <div v-if="this.scope === 'cluster' || this.scope === 'host'">
          <a-form-item>
            <tooltip-label slot="label" :title="$t('label.podid')" :tooltip="apiParams.podid.description"/>
            <a-select
              v-decorator="['pod', { initialValue: this.podSelected, rules: [{ required: true, message: `${$t('label.required')}`}] }]"
              @change="val => changePod(val)"
              showSearch
              optionFilterProp="children"
              :filterOption="(input, option) => {
                return option.componentOptions.children[0].text.toLowerCase().indexOf(input.toLowerCase()) >= 0
              }" >
              <a-select-option :value="pod.id" v-for="(pod) in pods" :key="pod.id">
                {{ pod.name }}
              </a-select-option>
            </a-select>
          </a-form-item>
          <a-form-item>
            <tooltip-label slot="label" :title="$t('label.clusterid')" :tooltip="apiParams.clusterid.description"/>
            <a-select
              v-decorator="['cluster', { initialValue: this.clusterSelected, rules: [{ required: true, message: `${$t('label.required')}`}] }]"
              @change="val => fetchHypervisor(val)"
              showSearch
              optionFilterProp="children"
              :filterOption="(input, option) => {
                return option.componentOptions.children[0].text.toLowerCase().indexOf(input.toLowerCase()) >= 0
              }" >
              <a-select-option :value="cluster.id" v-for="cluster in clusters" :key="cluster.id">
                {{ cluster.name }}
              </a-select-option>
            </a-select>
          </a-form-item>
        </div>
        <div v-if="this.scope === 'host'">
          <a-form-item :label="$t('label.hostid')">
            <a-select
              v-decorator="['host', { initialValue: this.hostSelected, rules: [{ required: true, message: `${$t('label.required')}`}] }]"
              @change="val => this.hostSelected = val"
              showSearch
              optionFilterProp="children"
              :filterOption="(input, option) => {
                return option.componentOptions.children[0].text.toLowerCase().indexOf(input.toLowerCase()) >= 0
              }" >
              <a-select-option :value="host.id" v-for="host in hosts" :key="host.id">
                {{ host.name }}
              </a-select-option>
            </a-select>
          </a-form-item>
        </div>
        <a-form-item>
          <tooltip-label slot="label" :title="$t('label.name')" :tooltip="apiParams.name.description"/>
          <a-input v-decorator="['name', { rules: [{ required: true, message: `${$t('label.required')}` }] }]"/>
        </a-form-item>
        <a-form-item>
          <tooltip-label slot="label" :title="$t('label.protocol')" :tooltip="$t('message.protocol.description')"/>
          <a-select
            v-decorator="['protocol', { initialValue: this.protocols[0], rules: [{ required: true, message: `${$t('label.required')}`}] }]"
            @change="val => this.protocolSelected = val"
            showSearch
            optionFilterProp="children"
            :filterOption="(input, option) => {
              return option.componentOptions.children[0].text.toLowerCase().indexOf(input.toLowerCase()) >= 0
            }" >
            <a-select-option :value="protocol" v-for="(protocol,idx) in protocols" :key="idx">
              {{ protocol }}
            </a-select-option>
          </a-select>
        </a-form-item>
        <div
          v-if="protocolSelected === 'nfs' || protocolSelected === 'SMB' || protocolSelected === 'iscsi' || protocolSelected === 'vmfs'|| protocolSelected === 'Gluster' || protocolSelected === 'Linstor' ||
            (protocolSelected === 'PreSetup' && hypervisorType === 'VMware') || protocolSelected === 'datastorecluster'">
          <a-form-item>
            <tooltip-label slot="label" :title="$t('label.server')" :tooltip="$t('message.server.description')"/>
            <a-input v-decorator="['server', { rules: [{ required: true, message: `${$t('label.required')}` }] }]" />
          </a-form-item>
        </div>
        <div v-if="protocolSelected === 'nfs' || protocolSelected === 'SMB' || protocolSelected === 'ocfs2' || (protocolSelected === 'PreSetup' && hypervisorType !== 'VMware') || protocolSelected === 'SharedMountPoint'">
          <a-form-item>
            <tooltip-label slot="label" :title="$t('label.path')" :tooltip="$t('message.path.description')"/>
            <a-input v-decorator="['path', { rules: [{ required: true, message: `${$t('label.required')}` }] }]" />
          </a-form-item>
        </div>
        <div v-if="protocolSelected === 'SMB'">
          <a-form-item :label="$t('label.smbusername')">
            <a-input v-decorator="['smbUsername', { rules: [{ required: true, message: `${$t('label.required')}` }] }]"/>
          </a-form-item>
          <a-form-item :label="$t('label.smbpassword')">
            <a-input-password v-decorator="['smbPassword', { rules: [{ required: true, message: `${$t('label.required')}` }] }]"/>
          </a-form-item>
          <a-form-item :label="$t('label.smbdomain')">
            <a-input v-decorator="['smbDomain', { rules: [{ required: true, message: `${$t('label.required')}` }] }]"/>
          </a-form-item>
        </div>
        <div v-if="protocolSelected === 'iscsi'">
          <a-form-item :label="$t('label.iqn')">
            <a-input v-decorator="['iqn', { rules: [{ required: true, message: `${$t('label.required')}` }] }]"/>
          </a-form-item>
          <a-form-item :label="$t('label.lun')">
            <a-input v-decorator="['lun', { rules: [{ required: true, message: `${$t('label.required')}` }] }]"/>
          </a-form-item>
        </div>
        <div v-if="protocolSelected === 'vmfs' || (protocolSelected === 'PreSetup' && hypervisorType === 'VMware') || protocolSelected === 'datastorecluster'">
          <a-form-item>
            <tooltip-label slot="label" :title="$t('label.vcenterdatacenter')" :tooltip="$t('message.datacenter.description')"/>
            <a-input v-decorator="['vCenterDataCenter', { rules: [{ required: true, message: `${$t('label.required')}` }] }]"/>
          </a-form-item>
          <a-form-item>
            <tooltip-label slot="label" :title="$t('label.vcenterdatastore')" :tooltip="$t('message.datastore.description')"/>
            <a-input v-decorator="['vCenterDataStore', { rules: [{ required: true, message: `${$t('label.required')}` }] }]"/>
          </a-form-item>
        </div>
        <div v-if="protocolSelected !== 'Linstor'">
          <a-form-item>
            <tooltip-label slot="label" :title="$t('label.providername')" :tooltip="apiParams.provider.description"/>
            <a-select
              v-decorator="['provider', { initialValue: providerSelected, rules: [{ required: true, message: `${$t('label.required')}`}] }]"
              @change="updateProviderAndProtocol"
              showSearch
              optionFilterProp="children"
              :filterOption="(input, option) => {
                return option.componentOptions.children[0].text.toLowerCase().indexOf(input.toLowerCase()) >= 0
              }" >
              <a-select-option :value="provider" v-for="(provider,idx) in providers" :key="idx">
                {{ provider }}
              </a-select-option>
            </a-select>
          </a-form-item>
        </div>
        <div v-if="this.providerSelected !== 'DefaultPrimary' && this.providerSelected !== 'PowerFlex' && this.providerSelected !== 'Linstor'">
          <a-form-item>
            <tooltip-label slot="label" :title="$t('label.ismanaged')" :tooltip="apiParams.managed.description"/>
            <a-checkbox-group v-decorator="['managed']" >
              <a-checkbox value="ismanaged"></a-checkbox>
            </a-checkbox-group>
          </a-form-item>
          <a-form-item>
            <tooltip-label slot="label" :title="$t('label.capacitybytes')" :tooltip="apiParams.capacitybytes.description"/>
            <a-input v-decorator="['capacityBytes']" />
          </a-form-item>
          <a-form-item>
            <tooltip-label slot="label" :title="$t('label.capacityiops')" :tooltip="apiParams.capacityiops.description"/>
            <a-input v-decorator="['capacityIops']" />
          </a-form-item>
          <a-form-item>
            <tooltip-label slot="label" :title="$t('label.url')" :tooltip="apiParams.url.description"/>
            <a-input v-decorator="['url']" />
          </a-form-item>
        </div>
        <div v-if="this.providerSelected === 'PowerFlex'">
          <a-form-item>
            <tooltip-label slot="label" :title="$t('label.powerflex.gateway')" :tooltip="$t('label.powerflex.gateway')"/>
            <a-input v-decorator="['powerflexGateway', { rules: [{ required: true, message: `${$t('label.required')}` }] }]"/>
          </a-form-item>
          <a-form-item>
            <tooltip-label slot="label" :title="$t('label.powerflex.gateway.username')" :tooltip="$t('label.powerflex.gateway.username')"/>
            <a-input v-decorator="['powerflexGatewayUsername', { rules: [{ required: true, message: `${$t('label.required')}` }] }]"/>
          </a-form-item>
          <a-form-item>
            <tooltip-label slot="label" :title="$t('label.powerflex.gateway.password')" :tooltip="$t('label.powerflex.gateway.password')"/>
            <a-input-password v-decorator="['powerflexGatewayPassword', { rules: [{ required: true, message: `${$t('label.required')}` }] }]"/>
          </a-form-item>
          <a-form-item>
            <tooltip-label slot="label" :title="$t('label.powerflex.storage.pool')" :tooltip="$t('label.powerflex.storage.pool')"/>
            <a-input v-decorator="['powerflexStoragePool', { rules: [{ required: true, message: `${$t('label.required')}` }] }]"/>
          </a-form-item>
        </div>
        <div v-if="this.protocolSelected === 'RBD'">
          <a-form-item :label="$t('label.rados.monitor')">
            <a-input v-decorator="['radosmonitor']" />
          </a-form-item><a-form-item :label="$t('label.rados.pool')">
            <a-input v-decorator="['radospool']" />
          </a-form-item>
          <a-form-item :label="$t('label.rados.user')">
            <a-input v-decorator="['radosuser']" />
          </a-form-item>
          <a-form-item :label="$t('label.rados.secret')">
            <a-input v-decorator="['radossecret']" />
          </a-form-item>
        </div>
        <div v-if="protocolSelected === 'CLVM'">
          <a-form-item :label="$t('label.volumegroup')">
            <a-input v-decorator="['volumegroup', { rules: [{ required: true, message: `${$t('label.required')}`}] }]" />
          </a-form-item>
        </div>
        <div v-if="protocolSelected === 'Gluster'">
          <a-form-item :label="$t('label.volume')">
            <a-input v-decorator="['volume']" />
          </a-form-item>
        </div>
        <div v-if="protocolSelected === 'Linstor'">
          <a-form-item>
            <span slot="label">
              {{ $t('label.resourcegroup') }}
              <a-tooltip :title="$t('message.linstor.resourcegroup.description')">
                <a-icon type="info-circle" style="color: rgba(0,0,0,.45)" />
              </a-tooltip>
            </span>
            <a-input v-decorator="['resourcegroup', { rules: [{ required: true, message: `${$t('label.required')}` }] }]" />
          </a-form-item>
        </div>
        <a-form-item>
          <tooltip-label slot="label" :title="$t('label.storagetags')" :tooltip="apiParams.tags.description"/>
          <a-select
            mode="tags"
            v-model="selectedTags"
            showSearch
            optionFilterProp="children"
            :filterOption="(input, option) => {
              return option.componentOptions.children[0].text.toLowerCase().indexOf(input.toLowerCase()) >= 0
            }" >
            <a-select-option v-for="tag in storageTags" :key="tag.name">{{ tag.name }}</a-select-option>
          </a-select>
        </a-form-item>

        <div :span="24" class="action-button">
          <a-button @click="closeModal">{{ $t('label.cancel') }}</a-button>
          <a-button type="primary" ref="submit" @click="handleSubmit">{{ $t('label.ok') }}</a-button>
        </div>
      </a-form>
    </a-spin>
  </div>
</template>

<script>
import { api } from '@/api'
import _ from 'lodash'
import ResourceIcon from '@/components/view/ResourceIcon'
import TooltipLabel from '@/components/widgets/TooltipLabel'

export default {
  name: 'AddPrimaryStorage',
  components: {
    TooltipLabel,
    ResourceIcon
  },
  props: {
    resource: {
      type: Object,
      required: true
    }
  },
  inject: ['parentFetchData'],
  data () {
    return {
      hypervisors: ['KVM', 'VMware', 'Hyperv', 'Any'],
      protocols: [],
      providers: [],
      scope: 'cluster',
      zones: [],
      pods: [],
      clusters: [],
      hosts: [],
      selectedTags: [],
      storageTags: [],
      zoneId: '',
      zoneSelected: '',
      podSelected: '',
      clusterSelected: '',
      hostSelected: '',
      hypervisorType: '',
      protocolSelected: 'nfs',
      providerSelected: 'DefaultPrimary',
      selectedHypervisor: 'KVM',
      size: 'default',
      loading: false
    }
  },
  beforeCreate () {
    this.form = this.$form.createForm(this)
    this.apiParams = this.$getApiParams('createStoragePool')
  },
  created () {
    this.fetchData()
  },
  methods: {
    fetchData () {
      this.getInfraData()
      this.listStorageTags()
      this.listStorageProviders()
    },
    getInfraData () {
      this.loading = true
      api('listZones', { showicon: true }).then(json => {
        this.zones = json.listzonesresponse.zone || []
        this.changeZone(this.zones[0] ? this.zones[0].id : '')
      }).finally(() => {
        this.loading = false
      })
    },
    changeZone (value) {
      this.zoneSelected = value
      if (this.zoneSelected === '') {
        this.podSelected = ''
        return
      }
      api('listPods', {
        zoneid: this.zoneSelected
      }).then(json => {
        this.pods = json.listpodsresponse.pod || []
        this.changePod(this.pods[0] ? this.pods[0].id : '')
      })
    },
    changePod (value) {
      this.podSelected = value
      if (this.podSelected === '') {
        this.clusterSelected = ''
        return
      }
      api('listClusters', {
        podid: this.podSelected
      }).then(json => {
        this.clusters = json.listclustersresponse.cluster || []
        if (this.clusters.length > 0) {
          this.clusterSelected = this.clusters[0].id
          this.fetchHypervisor()
        }
      }).then(() => {
        api('listHosts', {
          clusterid: this.clusterSelected
        }).then(json => {
          this.hosts = json.listhostsresponse.host || []
          if (this.hosts.length > 0) {
            this.hostSelected = this.hosts[0].id
          }
        })
      })
    },
    listStorageProviders () {
      this.providers = []
      this.loading = true
      api('listStorageProviders', { type: 'primary' }).then(json => {
        var providers = json.liststorageprovidersresponse.dataStoreProvider || []
        for (const provider of providers) {
          this.providers.push(provider.name)
        }
      }).finally(() => {
        this.loading = false
      })
    },
    listStorageTags () {
      this.loading = true
      api('listStorageTags').then(json => {
        this.storageTags = json.liststoragetagsresponse.storagetag || []
        if (this.storageTags) {
          this.storageTags = _.uniqBy(this.storageTags, 'name')
        }
      }).finally(() => {
        this.loading = false
      })
    },
    fetchHypervisor (value) {
      const cluster = this.clusters.find(cluster => cluster.id === this.clusterSelected)
      this.hypervisorType = cluster.hypervisortype
      if (this.hypervisorType === 'KVM') {
        this.protocols = ['nfs', 'SharedMountPoint', 'RBD', 'CLVM', 'Gluster', 'Linstor', 'custom']
      } else if (this.hypervisorType === 'XenServer') {
        this.protocols = ['nfs', 'PreSetup', 'iscsi', 'custom']
      } else if (this.hypervisorType === 'VMware') {
        this.protocols = ['nfs', 'vmfs', 'custom']
        if ('importVsphereStoragePolicies' in this.$store.getters.apis) {
          this.protocols = ['nfs', 'PreSetup', 'datastorecluster', 'custom']
        }
      } else if (this.hypervisorType === 'Hyperv') {
        this.protocols = ['SMB']
      } else if (this.hypervisorType === 'Ovm') {
        this.protocols = ['nfs', 'ocfs2']
      } else if (this.hypervisorType === 'LXC') {
        this.protocols = ['nfs', 'SharedMountPoint', 'RBD']
      } else {
        this.protocols = ['nfs']
      }
    },
    nfsURL (server, path) {
      var url
      if (path.substring(0, 1) !== '/') {
        path = '/' + path
      }
      if (server.indexOf('://') === -1) {
        url = 'nfs://' + server + path
      } else {
        url = server + path
      }

      return url
    },
    smbURL (server, path, smbUsername, smbPassword, smbDomain) {
      var url = ''
      if (path.substring(0, 1) !== '/') {
        path = '/' + path
      }
      if (server.indexOf('://') === -1) {
        url += 'cifs://'
      }
      url += (server + path)
      return url
    },
    presetupURL (server, path) {
      var url
      if (server.indexOf('://') === -1) {
        url = 'presetup://' + server + path
      } else {
        url = server + path
      }
      return url
    },
    datastoreclusterURL (server, path) {
      var url
      if (server.indexOf('://') === -1) {
        url = 'datastorecluster://' + server + path
      } else {
        url = server + path
      }
      return url
    },
    ocfs2URL (server, path) {
      var url
      if (server.indexOf('://') === -1) {
        url = 'ocfs2://' + server + path
      } else {
        url = server + path
      }
      return url
    },
    SharedMountPointURL (server, path) {
      var url
      if (server.indexOf('://') === -1) {
        url = 'SharedMountPoint://' + server + path
      } else {
        url = server + path
      }
      return url
    },
    rbdURL (monitor, pool, id, secret) {
      var url
      /*  Replace the + and / symbols by - and _ to have URL-safe base64 going to the API
          It's hacky, but otherwise we'll confuse java.net.URI which splits the incoming URI
      */
      secret = secret.replace(/\+/g, '-')
      secret = secret.replace(/\//g, '_')
      if (id !== null && secret !== null) {
        monitor = id + ':' + secret + '@' + monitor
      }
      if (pool.substring(0, 1) !== '/') {
        pool = '/' + pool
      }
      if (monitor.indexOf('://') === -1) {
        url = 'rbd://' + monitor + pool
      } else {
        url = monitor + pool
      }
      return url
    },
    clvmURL (vgname) {
      var url
      if (vgname.indexOf('://') === -1) {
        url = 'clvm://localhost/' + vgname
      } else {
        url = vgname
      }
      return url
    },
    vmfsURL (server, path) {
      var url
      if (server.indexOf('://') === -1) {
        url = 'vmfs://' + server + path
      } else {
        url = server + path
      }
      return url
    },
    iscsiURL (server, iqn, lun) {
      var url
      if (server.indexOf('://') === -1) {
        url = 'iscsi://' + server + iqn + '/' + lun
      } else {
        url = server + iqn + '/' + lun
      }
      return url
    },
    glusterURL (server, path) {
      var url
      if (server.indexOf('://') === -1) {
        url = 'gluster://' + server + path
      } else {
        url = server + path
      }
      return url
    },
    powerflexURL (gateway, username, password, pool) {
      var url = 'powerflex://' + encodeURIComponent(username) + ':' + encodeURIComponent(password) + '@' +
       gateway + '/' + encodeURIComponent(pool)
      return url
    },
    updateProviderAndProtocol (value) {
      if (value === 'PowerFlex') {
        this.protocols = ['custom']
        this.protocolSelected = 'custom'
        this.form.setFieldsValue({
          protocol: 'custom'
        })
      } else {
        this.fetchHypervisor(null)
      }
      this.providerSelected = value
    },
    closeModal () {
      this.$parent.$parent.close()
    },
    linstorURL (server) {
      var url
      if (server.indexOf('://') === -1) {
        url = 'http://' + server
      } else {
        url = server
      }

      return url
    },
    handleSubmit (e) {
      e.preventDefault()
      if (this.loading) return
      this.form.validateFieldsAndScroll((err, values) => {
        if (err) {
          return
        }
        var params = {
          scope: values.scope,
          zoneid: values.zone,
          name: values.name,
          provider: values.provider
        }
        if (values.scope === 'zone') {
          params.hypervisor = values.hypervisor
        }
        if (values.scope === 'cluster' || values.scope === 'host') {
          params.podid = values.pod
          params.clusterid = values.cluster
        }
        if (values.scope === 'host') {
          params.hostid = values.host
        }
        var server = values.server ? values.server : null
        var path = values.path ? values.path : null
        if (path !== null && path.substring(0, 1) !== '/') {
          path = '/' + path
        }
        var url = ''
        if (values.protocol === 'nfs') {
          url = this.nfsURL(server, path)
        } else if (values.protocol === 'SMB') {
          url = this.smbURL(server, path)
          const smbParams = {
            user: encodeURIComponent(values.smbUsername),
            password: encodeURIComponent(values.smbPassword),
            domain: values.smbDomain
          }
          Object.keys(smbParams).forEach((key, index) => {
            params['details[' + index.toString() + '].' + key] = smbParams[key]
          })
        } else if (values.protocol === 'PreSetup' && this.hypervisorType !== 'VMware') {
          server = 'localhost'
          url = this.presetupURL(server, path)
        } else if (values.protocol === 'PreSetup' && this.hypervisorType === 'VMware') {
          path = values.vCenterDataCenter
          if (path.substring(0, 1) !== '/') {
            path = '/' + path
          }
          path += '/' + values.vCenterDataStore
          url = this.presetupURL(server, path)
        } else if (values.protocol === 'datastorecluster' && this.hypervisorType === 'VMware') {
          path = values.vCenterDataCenter
          if (path.substring(0, 1) !== '/') {
            path = '/' + path
          }
          path += '/' + values.vCenterDataStore
          url = this.datastoreclusterURL(server, path)
        } else if (values.protocol === 'ocfs2') {
          url = this.ocfs2URL(server, path)
        } else if (values.protocol === 'SharedMountPoint') {
          server = 'localhost'
          url = this.SharedMountPointURL(server, path)
        } else if (values.protocol === 'CLVM') {
          var vg = (values.volumegroup.substring(0, 1) !== '/') ? ('/' + values.volumegroup) : values.volumegroup
          url = this.clvmURL(vg)
        } else if (values.protocol === 'RBD') {
          url = this.rbdURL(values.radosmonitor, values.radospool, values.radosuser, values.radossecret)
        } else if (values.protocol === 'vmfs') {
          path = values.vCenterDataCenter
          if (path.substring(0, 1) !== '/') {
            path = '/' + path
          }
          path += '/' + values.vCenterDataStore
          url = this.vmfsURL(server, path)
        } else if (values.protocol === 'Gluster') {
          var glustervolume = values.volume
          if (glustervolume.substring(0, 1) !== '/') {
            glustervolume = '/' + glustervolume
          }
          url = this.glusterURL(server, glustervolume)
        } else if (values.protocol === 'iscsi') {
          var iqn = values.iqn
          if (iqn.substring(0, 1) !== '/') {
            iqn = '/' + iqn
          }
          var lun = values.lun
          url = this.iscsiURL(server, iqn, lun)
        } else if (values.protocol === 'Linstor') {
          url = this.linstorURL(server)
          params.provider = 'Linstor'
          values.managed = false
          params['details[0].resourceGroup'] = values.resourcegroup
        }
        params.url = url
        if (values.provider !== 'DefaultPrimary' && values.provider !== 'PowerFlex') {
          if (values.managed) {
            params.managed = true
          } else {
            params.managed = false
          }
          if (values.capacityBytes && values.capacityBytes.length > 0) {
            params.capacityBytes = values.capacityBytes.split(',').join('')
          }
          if (values.capacityIops && values.capacityIops.length > 0) {
            params.capacityIops = values.capacityIops.split(',').join('')
          }
          if (values.url && values.url.length > 0) {
            params.url = values.url
          }
        }

        if (values.provider === 'PowerFlex') {
          params.url = this.powerflexURL(values.powerflexGateway, values.powerflexGatewayUsername,
            values.powerflexGatewayPassword, values.powerflexStoragePool)
        }

        if (this.selectedTags.length > 0) {
          params.tags = this.selectedTags.join()
        }
        this.loading = true
        api('createStoragePool', {}, 'POST', params).then(json => {
          this.$notification.success({
            message: this.$t('label.add.primary.storage'),
            description: this.$t('label.add.primary.storage')
          })
          this.closeModal()
          this.parentFetchData()
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
  @media (min-width: 1000px) {
    width: 500px;
  }
}
</style>
