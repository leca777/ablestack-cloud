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
  <div class="form" v-ctrl-enter="handleKeyboardSubmit">
    <a-alert type="warning">
      <span slot="message" v-html="$t('message.migrate.instance.to.host')" />
    </a-alert>
    <a-input-search
      class="top-spaced"
      :placeholder="$t('label.search')"
      v-model="searchQuery"
      @search="fetchData"
      autoFocus />
    <a-table
      class="top-spaced"
      size="small"
      style="overflow-y: auto"
      :loading="loading"
      :columns="columns"
      :dataSource="hosts"
      :pagination="false"
      :rowKey="record => record.id">
      <div slot="name" slot-scope="record">
        {{ record.name }}
        <a-tooltip v-if="record.name === $t('label.auto.assign')" :title="$t('message.migrate.instance.host.auto.assign')" placement="top">
          <a-icon type="info-circle" class="table-tooltip-icon" />
        </a-tooltip>
      </div>
      <div slot="suitability" slot-scope="record">
        <a-icon
          class="host-item__suitability-icon"
          type="check-circle"
          theme="twoTone"
          twoToneColor="#52c41a"
          v-if="record.suitableformigration" />
        <a-icon
          class="host-item__suitability-icon"
          type="close-circle"
          theme="twoTone"
          twoToneColor="#f5222d"
          v-else />
      </div>
      <div slot="memused" slot-scope="record">
        <span v-if="record.memoryused">
          {{ $bytesToHumanReadableSize(record.memoryused) }}
        </span>
      </div>
      <div slot="memoryallocatedpercentage" slot-scope="record">
        {{ record.memoryallocatedpercentage }}
      </div>
      <div slot="cluster" slot-scope="record">
        {{ record.clustername }}
      </div>
      <div slot="pod" slot-scope="record">
        {{ record.podname }}
      </div>
      <div slot="requiresstoragemigration" slot-scope="record">
        {{ record.requiresStorageMotion ? $t('label.yes') : $t('label.no') }}
      </div>
      <template slot="select" slot-scope="record">
        <a-radio
          class="host-item__radio"
          @click="handleSelectedHostChange(record)"
          :checked="record.id === selectedHost.id"
          :disabled="!record.suitableformigration"></a-radio>
      </template>
    </a-table>
    <a-pagination
      class="pagination"
      size="small"
      :current="page"
      :pageSize="pageSize"
      :total="totalCount"
      :showTotal="total => `${$t('label.total')} ${total} ${$t('label.items')}`"
      :pageSizeOptions="['10', '20', '40', '80', '100']"
      @change="handleChangePage"
      @showSizeChange="handleChangePageSize"
      showSizeChanger>
      <template slot="buildOptionText" slot-scope="props">
        <span>{{ props.value }} / {{ $t('label.page') }}</span>
      </template>
    </a-pagination>

    <a-form-item
      v-if="isUserVm"
      class="top-spaced">
      <tooltip-label slot="label" :title="$t('label.migrate.with.storage')" :tooltip="$t('message.migrate.with.storage')"/>
      <a-switch
        v-model="migrateWithStorage"
        :disabled="!selectedHost || !selectedHost.id || selectedHost.id === -1" />
    </a-form-item>
    <instance-volumes-storage-pool-select-list-view
      ref="volumeToPoolSelect"
      v-if="migrateWithStorage"
      class="top-spaced"
      :resource="resource"
      :clusterId="selectedHost.id ? selectedHost.clusterid : null"
      @select="handleVolumeToPoolChange" />

    <a-divider />

    <div class="actions">
      <a-button @click="closeModal">{{ $t('label.cancel') }}</a-button>
      <a-button type="primary" ref="submit" :disabled="!selectedHost.id" @click="submitForm">{{ $t('label.ok') }}</a-button>
    </div>
  </div>

</template>

<script>
import { api } from '@/api'
import TooltipLabel from '@/components/widgets/TooltipLabel'
import InstanceVolumesStoragePoolSelectListView from '@/components/view/InstanceVolumesStoragePoolSelectListView'

export default {
  name: 'VMMigrateWizard',
  components: {
    TooltipLabel,
    InstanceVolumesStoragePoolSelectListView
  },
  props: {
    resource: {
      type: Object,
      required: true
    }
  },
  data () {
    return {
      loading: true,
      hosts: [],
      selectedHost: {},
      searchQuery: '',
      totalCount: 0,
      page: 1,
      pageSize: 10,
      columns: [
        {
          title: this.$t('label.hostid'),
          scopedSlots: { customRender: 'name' }
        },
        {
          title: this.$t('label.suitability'),
          scopedSlots: { customRender: 'suitability' }
        },
        {
          title: this.$t('label.cpuused'),
          dataIndex: 'cpuused'
        },
        {
          title: this.$t('label.memoryallocated'),
          scopedSlots: { customRender: 'memoryallocatedpercentage' }
        },
        {
          title: this.$t('label.memused'),
          scopedSlots: { customRender: 'memused' }
        },
        {
          title: this.$t('label.cluster'),
          scopedSlots: { customRender: 'cluster' }
        },
        {
          title: this.$t('label.pod'),
          scopedSlots: { customRender: 'pod' }
        },
        {
          title: this.$t('label.storage.migration.required'),
          scopedSlots: { customRender: 'requiresstoragemigration' }
        },
        {
          title: this.$t('label.select'),
          scopedSlots: { customRender: 'select' }
        }
      ],
      migrateWithStorage: false,
      volumeToPoolSelection: []
    }
  },
  created () {
    this.fetchData()
  },
  computed: {
    isUserVm () {
      return this.$route.meta.name === 'vm'
    }
  },
  watch: {
    searchQuery (newValue, oldValue) {
      if (newValue !== oldValue) {
        this.page = 1
      }
    }
  },
  methods: {
    arrayHasItems (array) {
      return array !== null && array !== undefined && Array.isArray(array) && array.length > 0
    },
    fetchData () {
      this.loading = true
      api('findHostsForMigration', {
        virtualmachineid: this.resource.id,
        keyword: this.searchQuery,
        page: this.page,
        pagesize: this.pageSize
      }).then(response => {
        this.hosts = response.findhostsformigrationresponse.host || []
        this.hosts.sort((a, b) => {
          return b.suitableformigration - a.suitableformigration
        })
        for (const key in this.hosts) {
          if (this.hosts[key].suitableformigration && !this.hosts[key].requiresstoragemigration) {
            this.hosts.unshift({ id: -1, name: this.$t('label.auto.assign'), suitableformigration: true, requiresstoragemigration: false })
            break
          }
        }
        this.totalCount = response.findhostsformigrationresponse.count
      }).catch(error => {
        this.$message.error(`${this.$t('message.load.host.failed')}: ${error}`)
      }).finally(() => {
        this.loading = false
      })
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
    handleSelectedHostChange (host) {
      if (host.id === -1) {
        this.migrateWithStorage = false
      }
      this.selectedHost = host
      this.selectedVolumeForStoragePoolSelection = {}
      this.volumeToPoolSelection = []
      if (this.migrateWithStorage) {
        this.$refs.volumeToPoolSelect.resetSelection()
      }
    },
    handleVolumeToPoolChange (volumeToPool) {
      this.volumeToPoolSelection = volumeToPool
    },
    handleKeyboardSubmit () {
      if (this.selectedHost.id) {
        this.submitForm()
      }
    },
    closeModal () {
      this.$emit('close-action')
    },
    submitForm () {
      if (this.loading) return
      this.loading = true
      const migrateApi = this.isUserVm
        ? (this.selectedHost.requiresStorageMotion || this.volumeToPoolSelection.length > 0)
          ? 'migrateVirtualMachineWithVolume'
          : 'migrateVirtualMachine'
        : 'migrateSystemVm'
      var params = this.selectedHost.id === -1
        ? { autoselect: true, virtualmachineid: this.resource.id }
        : { hostid: this.selectedHost.id, virtualmachineid: this.resource.id }
      if (this.migrateWithStorage) {
        for (var i = 0; i < this.volumeToPoolSelection.length; i++) {
          const mapping = this.volumeToPoolSelection[i]
          params['migrateto[' + i + '].volume'] = mapping.volume
          params['migrateto[' + i + '].pool'] = mapping.pool
        }
      }
      api(migrateApi, params).then(response => {
        const jobId = response[migrateApi.toLowerCase() + 'response'].jobid
        this.$pollJob({
          jobId: jobId,
          title: `${this.$t('label.migrating')} ${this.resource.name}`,
          description: this.resource.name,
          successMessage: `${this.$t('message.success.migrating')} ${this.resource.name}`,
          successMethod: () => {
            this.closeModal()
          },
          errorMessage: this.$t('message.migrating.failed'),
          errorMethod: () => {
            this.closeModal()
          },
          loadingMessage: `${this.$t('message.migrating.processing')} ${this.resource.name}`,
          catchMessage: this.$t('error.fetching.async.job.result'),
          catchMethod: () => {
            this.closeModal()
          }
        })
        this.closeModal()
      }).catch(error => {
        this.$notification.error({
          message: this.$t('message.request.failed'),
          description: (error.response?.headers?.['x-description']) || error.message,
          duration: 0
        })
      }).finally(() => {
        this.loading = false
      })
    }
  }
}
</script>

<style scoped lang="scss">

  .form {
    width: 95vw;
    @media (min-width: 900px) {
      width: 850px;
    }
  }

  .host-item {
    padding-right: 20px;
    padding-bottom: 0;
    padding-left: 20px;

    &--selected {
      background-color: #e6f7ff;
    }

    &__row {
      display: flex;
      flex-direction: column;
      width: 100%;

      @media (min-width: 760px) {
        flex-direction: row;
      }
    }

    &__value {
      display: flex;
      flex-direction: column;
      align-items: flex-start;
      flex: 1;
      margin-bottom: 10px;

      &--small {

        @media (min-width: 760px) {
          flex: none;
          margin-right: 40px;
          margin-left: 40px;
        }
      }
    }

    &__title {
      font-weight: bold;
    }

    &__suitability-icon {
      margin-top: 5px;
    }

    &__radio {
      display: flex;
      align-items: center;
    }

  }

  .top-spaced {
    margin-top: 20px;
  }

  .pagination {
    margin-top: 20px;
  }

  .actions {
    display: flex;
    justify-content: flex-end;
    margin-top: 20px;

    button {
      &:not(:last-child) {
        margin-right: 10px;
      }
    }
  }
  .table-tooltip-icon {
    color: rgba(0,0,0,.45);
  }
</style>
