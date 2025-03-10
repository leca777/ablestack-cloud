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
  <div>
    <a-button
      v-if="(('deleteTemplate' in $store.getters.apis) && this.selectedRowKeys.length > 0)"
      type="danger"
      icon="delete"
      style="width: 100%; margin-bottom: 15px"
      @click="bulkActionConfirmation()">
      {{ $t('label.action.bulk.delete.templates') }}
    </a-button>
    <a-table
      size="small"
      style="overflow-y: auto"
      :loading="loading || fetchLoading"
      :columns="columns"
      :dataSource="dataSource"
      :pagination="false"
      :rowSelection="{selectedRowKeys: selectedRowKeys, onChange: onSelectChange}"
      :rowKey="record => record.zoneid">
      <div slot="zonename" slot-scope="text, record">
        <span v-if="fetchZoneIcon(record.zoneid)">
          <resource-icon :image="zoneIcon" size="1x" style="margin-right: 5px"/>
        </span>
        <a-icon v-else type="global" style="margin-right: 5px" />
        <span> {{ record.zonename }} </span>
      </div>
      <div slot="isready" slot-scope="text, record">
        <span v-if="record.isready">{{ $t('label.yes') }}</span>
        <span v-else>{{ $t('label.no') }}</span>
      </div>
      <template slot="expandedRowRender" slot-scope="record">
        <a-table
          style="marginLeft: -50px; marginTop: 10px; marginBottom: 10px"
          slot="expandedRowRender"
          :columns="innerColumns"
          :data-source="record.downloaddetails"
          :pagination="false"
          :bordered="true"
          :rowKey="record => record.zoneid">
        </a-table>
      </template>
      <template slot="action" slot-scope="text, record">
        <tooltip-button
          style="margin-right: 5px"
          :dataSource="templates"
          :disabled="!('copyTemplate' in $store.getters.apis && record.isready) || templates.includes(record.id)"
          :title="$t('label.action.copy.template')"
          icon="copy"
          :loading="copyLoading || fetchLoading"
          @click="showCopyTemplate(record)" />
        <tooltip-button
          style="margin-right: 5px"
          :dataSource="templates"
          :disabled="!('deleteTemplate' in $store.getters.apis) || templates.includes(record.id)"
          :title="$t('label.action.delete.template')"
          type="danger"
          icon="delete"
          :loading="fetchLoading"
          @click="onShowDeleteModal(record)"/>
      </template>
    </a-table>
    <a-pagination
      class="row-element"
      size="small"
      :current="page"
      :pageSize="pageSize"
      :total="itemCount"
      :showTotal="total => `${$t('label.total')} ${total} ${$t('label.items')}`"
      :pageSizeOptions="['10', '20', '40', '80', '100']"
      @change="handleChangePage"
      @showSizeChange="handleChangePageSize"
      showSizeChanger>
      <template slot="buildOptionText" slot-scope="props">
        <span>{{ props.value }} / {{ $t('label.page') }}</span>
      </template>
    </a-pagination>

    <a-modal
      v-if="'copyTemplate' in $store.getters.apis"
      style="top: 20px;"
      :title="$t('label.action.copy.template')"
      :visible="showCopyActionForm"
      :closable="true"
      :maskClosable="false"
      :footer="null"
      :confirmLoading="copyLoading"
      @cancel="onCloseModal"
      v-ctrl-enter="handleCopyTemplateSubmit"
      centered>
      <a-spin :spinning="copyLoading">
        <a-form
          :form="form"
          @submit="handleCopyTemplateSubmit"
          layout="vertical">
          <a-form-item :label="$t('label.zoneid')">
            <a-select
              id="zone-selection"
              mode="multiple"
              :placeholder="$t('label.select.zones')"
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
              autoFocus>
              <a-select-option v-for="zone in zones" :key="zone.id" :label="zone.name">
                <div>
                  <span v-if="zone.icon && zone.icon.base64image">
                    <resource-icon :image="zone.icon.base64image" size="1x" style="margin-right: 5px"/>
                  </span>
                  <a-icon v-else type="global" style="margin-right: 5px" />
                  {{ zone.name }}
                </div>
              </a-select-option>
            </a-select>
          </a-form-item>

          <div :span="24" class="action-button">
            <a-button @click="onCloseModal">{{ $t('label.cancel') }}</a-button>
            <a-button type="primary" ref="submit" @click="handleCopyTemplateSubmit">{{ $t('label.ok') }}</a-button>
          </div>
        </a-form>
      </a-spin>
    </a-modal>

    <a-modal
      :title="selectedItems.length > 0 && showTable ? $t(message.title) : $t('label.action.delete.template')"
      :visible="showDeleteTemplate"
      :closable="true"
      :maskClosable="false"
      :footer="null"
      v-ctrl-enter="deleteTemplate"
      :width="showTable ? modalWidth : '30vw'"
      @ok="selectedItems.length > 0 ? deleteTemplates() : deleteTemplate(currentRecord)"
      @cancel="onCloseModal"
      :ok-button-props="getOkProps()"
      :cancel-button-props="getCancelProps()"
      :confirmLoading="deleteLoading"
      centered>
      <div v-if="selectedRowKeys.length > 0">
        <a-alert type="error">
          <a-icon slot="message" type="exclamation-circle" style="color: red; fontSize: 30px; display: inline-flex" />
          <span style="padding-left: 5px" slot="message" v-html="`<b>${selectedRowKeys.length} ` + $t('label.items.selected') + `. </b>`" />
          <span slot="message" v-html="$t(message.confirmMessage)" />
        </a-alert>
      </div>
      <a-alert v-else :message="$t('message.action.delete.template')" type="warning" />
      <br />
      <a-table
        v-if="selectedRowKeys.length > 0 && showTable"
        size="middle"
        :columns="selectedColumns"
        :dataSource="selectedItems"
        :rowKey="(record, idx) => record.zoneid || record.name"
        :pagination="true"
        style="overflow-y: auto">
      </a-table>
      <a-spin :spinning="deleteLoading">
        <a-form-item :label="$t('label.isforced')" style="margin-bottom: 0;">
          <a-switch v-model="forcedDelete" autoFocus></a-switch>
        </a-form-item>
        <div :span="24" class="action-button">
          <a-button @click="onCloseModal">{{ $t('label.cancel') }}</a-button>
          <a-button type="primary" ref="submit" @click="deleteTemplate">{{ $t('label.ok') }}</a-button>
        </div>
      </a-spin>
    </a-modal>
    <bulk-action-progress
      :showGroupActionModal="showGroupActionModal"
      :selectedItems="selectedItems"
      :selectedColumns="selectedColumns"
      :message="message"
      @handle-cancel="handleCancel" />
  </div>
</template>

<script>
import { api } from '@/api'
import OsLogo from '@/components/widgets/OsLogo'
import ResourceIcon from '@/components/view/ResourceIcon'
import TooltipButton from '@/components/widgets/TooltipButton'
import BulkActionProgress from '@/components/view/BulkActionProgress'
import Status from '@/components/widgets/Status'
import eventBus from '@/config/eventBus'

export default {
  name: 'TemplateZones',
  components: {
    TooltipButton,
    OsLogo,
    ResourceIcon,
    BulkActionProgress,
    Status
  },
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
      columns: [],
      dataSource: [],
      templates: [],
      page: 1,
      pageSize: 10,
      itemCount: 0,
      fetchLoading: false,
      showCopyActionForm: false,
      currentRecord: {},
      zones: [],
      zoneLoading: false,
      copyLoading: false,
      deleteLoading: false,
      showDeleteTemplate: false,
      forcedDelete: false,
      selectedRowKeys: [],
      showGroupActionModal: false,
      selectedItems: [],
      selectedColumns: [],
      filterColumns: ['Status', 'Ready'],
      showConfirmationAction: false,
      message: {
        title: this.$t('label.action.bulk.delete.templates'),
        confirmMessage: this.$t('label.confirm.delete.templates')
      },
      modalWidth: '30vw',
      showTable: false
    }
  },
  beforeCreate () {
    this.form = this.$form.createForm(this)
    this.apiParams = this.$getApiParams('copyTemplate')
  },
  created () {
    this.columns = [
      {
        title: this.$t('label.zonename'),
        dataIndex: 'zonename',
        scopedSlots: { customRender: 'zonename' }
      },
      {
        title: this.$t('label.status'),
        dataIndex: 'status'
      },
      {
        title: this.$t('label.isready'),
        dataIndex: 'isready',
        scopedSlots: { customRender: 'isready' }
      }
    ]
    this.innerColumns = [
      {
        title: this.$t('label.secondary.storage'),
        dataIndex: 'datastore'
      },
      {
        title: this.$t('label.download.percent'),
        dataIndex: 'downloadPercent'
      },
      {
        title: this.$t('label.download.state'),
        dataIndex: 'downloadState'
      }
    ]
    if (this.isActionPermitted()) {
      this.columns.push({
        title: '',
        dataIndex: 'action',
        width: 100,
        scopedSlots: { customRender: 'action' }
      })
    }

    const userInfo = this.$store.getters.userInfo
    if (!['Admin'].includes(userInfo.roletype) &&
      (userInfo.account !== this.resource.account || userInfo.domain !== this.resource.domain)) {
      this.columns = this.columns.filter(col => { return col.dataIndex !== 'status' })
    }
    this.fetchData()
  },
  watch: {
    loading (newData, oldData) {
      if (!newData && !this.showGroupActionModal) {
        this.fetchData()
      }
    }
  },
  methods: {
    fetchData () {
      const params = {}
      params.id = this.resource.id
      params.templatefilter = 'executable'
      params.listall = true
      params.page = this.page
      params.pagesize = this.pageSize

      this.dataSource = []
      this.itemCount = 0
      this.fetchLoading = true
      api('listTemplates', params).then(json => {
        this.dataSource = json.listtemplatesresponse.template || []
        this.itemCount = json.listtemplatesresponse.count || 0
      }).catch(error => {
        this.$notifyError(error)
      }).finally(() => {
      })
      this.templates = []
      api('listDesktopControllerVersions').then(json => {
        var items = json.listdesktopcontrollerversionsresponse.desktopcontrollerversion
        if (items != null) {
          for (var i = 0; i < items.length; i++) {
            for (var j = 0; j < items[i].templates.length; j++) {
              this.templates.push(items[i].templates[j].id)
            }
          }
        }
      }).finally(() => {
      })
      api('listDesktopMasterVersions').then(json => {
        var items = json.listdesktopmasterversionsresponse.desktopmasterversion
        if (items != null) {
          for (var i = 0; i < items.length; i++) {
            this.templates.push(items[i].templateid)
          }
        }
      }).finally(() => {
        this.fetchLoading = false
        this.$set(this.resource, 'templates', this.templates)
      })
      this.fetchZoneData()
    },
    fetchZoneIcon (zoneid) {
      const zoneItem = this.zones.filter(zone => zone.id === zoneid)
      if (zoneItem?.[0]?.icon?.base64image) {
        this.zoneIcon = zoneItem[0].icon.base64image
        return true
      }
      return false
    },
    handleChangePage (page, pageSize) {
      this.page = page
      this.pageSize = pageSize
      this.fetchData()
    },
    handleChangePageSize (currentPage, pageSize) {
      this.page = currentPage
      this.pageSize = pageSize
      this.fetchData()
    },
    isActionPermitted () {
      return (['Admin'].includes(this.$store.getters.userInfo.roletype) || // If admin or owner or belongs to current project
        (this.resource.domainid === this.$store.getters.userInfo.domainid && this.resource.account === this.$store.getters.userInfo.account) ||
        (this.resource.domainid === this.$store.getters.userInfo.domainid && this.resource.projectid && this.$store.getters.project && this.$store.getters.project.id && this.resource.projectid === this.$store.getters.project.id)) &&
        (this.resource.isready || !this.resource.status || this.resource.status.indexOf('Downloaded') === -1) && // Template is ready or downloaded
        this.resource.templatetype !== 'SYSTEM'
    },
    setSelection (selection) {
      this.selectedRowKeys = selection
      if (selection?.length > 0) {
        this.modalWidth = '50vw'
        this.$emit('selection-change', this.selectedRowKeys)
        this.selectedItems = (this.dataSource.filter(function (item) {
          return selection.indexOf(item.zoneid) !== -1
        }))
      } else {
        this.modalWidth = '30vw'
        this.selectedItems = []
      }
    },
    resetSelection () {
      this.setSelection([])
    },
    onSelectChange (selectedRowKeys, selectedRows) {
      this.setSelection(selectedRowKeys)
    },
    bulkActionConfirmation () {
      this.showConfirmationAction = true
      this.selectedColumns = this.columns.filter(column => {
        return !this.filterColumns.includes(column.title)
      })
      this.selectedItems = this.selectedItems.map(v => ({ ...v, status: 'InProgress' }))
      this.onShowDeleteModal(this.selectedItems[0])
    },
    handleCancel () {
      eventBus.$emit('update-bulk-job-status', this.selectedItems, false)
      this.showGroupActionModal = false
      this.selectedItems = []
      this.selectedColumns = []
      this.selectedRowKeys = []
      this.showTable = false
      this.fetchData()
      if (this.dataSource.length === 0) {
        this.$router.go(-1)
      }
    },
    getOkProps () {
      if (this.selectedRowKeys.length > 0) {
        return { props: { type: 'default' } }
      } else {
        return { props: { type: 'primary' } }
      }
    },
    getCancelProps () {
      if (this.selectedRowKeys.length > 0) {
        return { props: { type: 'primary' } }
      } else {
        return { props: { type: 'default' } }
      }
    },
    deleteTemplates (e) {
      this.showConfirmationAction = false
      this.selectedColumns.splice(0, 0, {
        dataIndex: 'status',
        title: this.$t('label.operation.status'),
        scopedSlots: { customRender: 'status' },
        filters: [
          { text: 'In Progress', value: 'InProgress' },
          { text: 'Success', value: 'success' },
          { text: 'Failed', value: 'failed' }
        ]
      })
      if (this.selectedRowKeys.length > 0 && this.showTable) {
        this.showGroupActionModal = true
      }
      for (const template of this.selectedItems) {
        this.deleteTemplate(template)
      }
    },
    deleteTemplate (template) {
      if (!template.id) {
        template = this.currentRecord
      }
      const params = {
        id: template.id,
        forced: this.forcedDelete,
        zoneid: template.zoneid
      }
      this.deleteLoading = true
      api('deleteTemplate', params).then(json => {
        const jobId = json.deletetemplateresponse.jobid
        eventBus.$emit('update-job-details', jobId, null)
        const singleZone = (this.dataSource.length === 1)
        this.$pollJob({
          jobId,
          title: this.$t('label.action.delete.template'),
          description: this.resource.name,
          successMethod: result => {
            if (singleZone) {
              const isResourcePage = (this.$route.params && this.$route.params.id)
              if (isResourcePage) {
                if (this.selectedItems.length === 0 && !this.showGroupActionModal) {
                  this.$router.push({ path: '/template' })
                }
              }
            } else {
              if (this.selectedItems.length === 0) {
                this.fetchData()
              }
            }
            if (this.selectedItems.length > 0) {
              eventBus.$emit('update-resource-state', this.selectedItems, template.zoneid, 'success')
            }
          },
          errorMethod: () => {
            if (this.selectedItems.length === 0) {
              this.fetchData()
            }
            if (this.selectedItems.length > 0) {
              eventBus.$emit('update-resource-state', this.selectedItems, template.zoneid, 'failed')
            }
          },
          showLoading: !(this.selectedItems.length > 0 && this.showGroupActionModal),
          loadingMessage: `${this.$t('label.deleting.template')} ${this.resource.name} ${this.$t('label.in.progress')}`,
          catchMessage: this.$t('error.fetching.async.job.result'),
          bulkAction: this.selectedItems.length > 0 && this.showGroupActionModal
        })
        this.onCloseModal()
        if (this.selectedItems.length === 0) {
          this.fetchData()
        }
      }).catch(error => {
        this.$notifyError(error)
      }).finally(() => {
        this.deleteLoading = false
      })
    },
    fetchZoneData () {
      this.zones = []
      this.zoneLoading = true
      api('listZones', { listall: true, showicon: true }).then(json => {
        const zones = json.listzonesresponse.zone || []
        this.zones = [...zones.filter((zone) => this.currentRecord.zoneid !== zone.id)]
      }).finally(() => {
        this.zoneLoading = false
      })
    },
    showCopyTemplate (record) {
      this.currentRecord = record
      this.form.setFieldsValue({
        zoneid: []
      })
      this.fetchZoneData()
      this.showCopyActionForm = true
    },
    onShowDeleteModal (record) {
      this.forcedDelete = false
      this.currentRecord = record
      this.showDeleteTemplate = true
      if (this.showConfirmationAction) {
        this.showTable = true
      } else {
        this.selectedItems = []
      }
    },
    onCloseModal () {
      this.currentRecord = {}
      this.showCopyActionForm = false
      this.showDeleteTemplate = false
      this.showConfirmationAction = false
      this.showTable = false
      this.selectedRowKeys = []
    },
    handleCopyTemplateSubmit (e) {
      e.preventDefault()
      if (this.copyLoading) return
      this.form.validateFieldsAndScroll((err, values) => {
        if (err) {
          return
        }
        const params = {
          id: this.currentRecord.id,
          sourcezoneid: this.currentRecord.zoneid,
          destzoneids: values.zoneid.join()
        }
        this.copyLoading = true
        api('copyTemplate', params).then(json => {
          const jobId = json.copytemplateresponse.jobid
          eventBus.$emit('update-job-details', jobId, null)
          this.$pollJob({
            jobId,
            title: this.$t('label.action.copy.template'),
            description: this.resource.name,
            successMethod: result => {
              this.fetchData()
            },
            errorMethod: () => this.fetchData(),
            loadingMessage: `${this.$t('label.action.copy.template')} ${this.resource.name} ${this.$t('label.in.progress')}`,
            catchMessage: this.$t('error.fetching.async.job.result')
          })
        }).catch(error => {
          this.$notification.error({
            message: this.$t('message.request.failed'),
            description: (error.response && error.response.headers && error.response.headers['x-description']) || error.message
          })
        }).finally(() => {
          this.copyLoading = false
          this.$emit('refresh-data')
          this.onCloseModal()
          this.fetchData()
        })
      })
    }
  }
}
</script>

<style lang="less" scoped>
.row-element {
  margin-top: 15px;
  margin-bottom: 15px;
}
</style>
