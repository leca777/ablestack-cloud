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
    <a-tabs class="form-layout">
      <a-tab-pane
        key="1"
        :tab="$t('label.action.project.add.account')"
        v-ctrl-enter="addAccountToProject">
        <a-form
          :form="form"
          @submit="addAccountToProject"
          layout="vertical">
          <a-form-item>
            <tooltip-label slot="label" :title="$t('label.account')" :tooltip="apiParams.addAccountToProject.account.description"/>
            <a-input
              v-decorator="['account']"
              :placeholder="apiParams.addAccountToProject.account.description"
              autoFocus />
          </a-form-item>
          <a-form-item>
            <tooltip-label slot="label" :title="$t('label.email')" :tooltip="apiParams.addAccountToProject.email.description"/>
            <a-input
              v-decorator="['email']"
              :placeholder="apiParams.addAccountToProject.email.description"></a-input>
          </a-form-item>
          <a-form-item v-if="apiParams.addAccountToProject.projectroleid">
            <tooltip-label slot="label" :title="$t('label.project.role')" :tooltip="apiParams.addAccountToProject.projectroleid.description"/>
            <a-select
              v-decorator="['projectroleid']"
              :loading="loading"
              :placeholder="$t('label.project.role')"
              showSearch
              optionFilterProp="children"
              :filterOption="(input, option) => {
                return option.componentOptions.children[0].text.toLowerCase().indexOf(input.toLowerCase()) >= 0
              }" >
              <a-select-option v-for="role in projectRoles" :key="role.id">
                {{ role.name }}
              </a-select-option>
            </a-select>
          </a-form-item>
          <a-form-item v-if="apiParams.addAccountToProject.roletype">
            <tooltip-label slot="label" :title="$t('label.name')" :tooltip="apiParams.addAccountToProject.roletype.description"/>
            <a-select
              v-decorator="['roletype']"
              :placeholder="$t('label.roletype')"
              showSearch
              optionFilterProp="children"
              :filterOption="(input, option) => {
                return option.componentOptions.children[0].text.toLowerCase().indexOf(input.toLowerCase()) >= 0
              }" >
              <a-select-option value="Admin">Admin</a-select-option>
              <a-select-option value="Regular">Regular</a-select-option>
            </a-select>
          </a-form-item>
          <div :span="24" class="action-button">
            <a-button @click="closeAction">{{ this.$t('label.cancel') }}</a-button>
            <a-button type="primary" ref="submit" @click="addAccountToProject" :loading="loading">{{ $t('label.ok') }}</a-button>
          </div>
        </a-form>
      </a-tab-pane>
      <a-tab-pane
        key="2"
        :tab="$t('label.action.project.add.user')"
        v-if="apiParams.addUserToProject"
        v-ctrl-enter="addUserToProject">
        <a-form
          :form="form"
          @submit="addUserToProject"
          layout="vertical">
          <a-alert type="warning" style="margin-bottom: 20px">
            <span slot="message" v-html="$t('message.add.user.to.project')"></span>
          </a-alert>
          <a-form-item>
            <tooltip-label slot="label" :title="$t('label.name')" :tooltip="apiParams.addUserToProject.username.description"/>
            <a-input
              v-decorator="['username']"
              :placeholder="apiParams.addUserToProject.username.description"
              autoFocus />
          </a-form-item>
          <a-form-item>
            <tooltip-label slot="label" :title="$t('label.email')" :tooltip="apiParams.addUserToProject.email.description"/>
            <a-input
              v-decorator="['email']"
              :placeholder="apiParams.addUserToProject.email.description"></a-input>
          </a-form-item>
          <a-form-item>
            <tooltip-label slot="label" :title="$t('label.project.role')" :tooltip="apiParams.addUserToProject.roletype.description"/>
            <a-select
              v-decorator="['projectroleid']"
              :loading="loading"
              :placeholder="$t('label.project.role')"
              showSearch
              optionFilterProp="children"
              :filterOption="(input, option) => {
                return option.componentOptions.children[0].text.toLowerCase().indexOf(input.toLowerCase()) >= 0
              }" >
              <a-select-option v-for="role in projectRoles" :key="role.id">
                {{ role.name }}
              </a-select-option>
            </a-select>
          </a-form-item>
          <a-form-item>
            <tooltip-label slot="label" :title="$t('label.roletype')" :tooltip="apiParams.addUserToProject.roletype.description"/>
            <a-select
              v-decorator="['roletype']"
              :placeholder="$t('label.roletype')"
              showSearch
              optionFilterProp="children"
              :filterOption="(input, option) => {
                return option.componentOptions.children[0].text.toLowerCase().indexOf(input.toLowerCase()) >= 0
              }" >
              <a-select-option value="Admin">Admin</a-select-option>
              <a-select-option value="Regular">Regular</a-select-option>
            </a-select>
          </a-form-item>
          <div :span="24" class="action-button">
            <a-button @click="closeAction">{{ this.$t('label.cancel') }}</a-button>
            <a-button type="primary" ref="submit" @click="addUserToProject" :loading="loading">{{ $t('label.ok') }}</a-button>
          </div>
        </a-form>
      </a-tab-pane>
    </a-tabs>
  </div>
</template>
<script>
import { api } from '@/api'
import TooltipLabel from '@/components/widgets/TooltipLabel'

export default {
  name: 'AddAccountOrUserToProject',
  components: {
    TooltipLabel
  },
  props: {
    resource: {
      type: Object,
      required: true
    }
  },
  data () {
    return {
      users: [],
      accounts: [],
      projectRoles: [],
      selectedUser: null,
      selectedAccount: null,
      loading: false,
      load: {
        users: false,
        accounts: false,
        projectRoles: false
      }
    }
  },
  created () {
    this.fetchData()
  },
  beforeCreate () {
    this.form = this.$form.createForm(this)
    const apis = ['addAccountToProject']
    if ('addUserToProject' in this.$store.getters.apis) {
      apis.push('addUserToProject')
    }
    this.apiParams = {}
    for (var api of apis) {
      this.apiParams[api] = this.$getApiParams(api)
    }
  },
  methods: {
    fetchData () {
      this.fetchUsers()
      this.fetchAccounts()
      if (this.isProjectRolesSupported()) {
        this.fetchProjectRoles()
      }
    },
    fetchUsers () {
      this.load.users = true
      api('listUsers', { listall: true }).then(response => {
        this.users = response.listusersresponse.user ? response.listusersresponse.user : []
      }).catch(error => {
        this.$notifyError(error)
      }).finally(() => {
        this.load.users = false
      })
    },
    fetchAccounts () {
      this.load.accounts = true
      api('listAccounts', {
        domainid: this.resource.domainid
      }).then(response => {
        this.accounts = response.listaccountsresponse.account || []
      }).catch(error => {
        this.$notifyError(error)
      }).finally(() => {
        this.load.accounts = false
      })
    },
    fetchProjectRoles () {
      this.load.projectRoles = true
      api('listProjectRoles', {
        projectid: this.resource.id
      }).then(response => {
        this.projectRoles = response.listprojectrolesresponse.projectrole || []
      }).catch(error => {
        this.$notifyError(error)
      }).finally(() => {
        this.load.projectRoles = false
      })
    },
    isProjectRolesSupported () {
      return ('listProjectRoles' in this.$store.getters.apis)
    },
    addAccountToProject (e) {
      e.preventDefault()
      if (this.loading) return
      this.form.validateFieldsAndScroll((err, values) => {
        if (err) {
          return
        }
        this.loading = true
        var params = {
          projectid: this.resource.id
        }
        for (const key in values) {
          const input = values[key]
          if (input === undefined) {
            continue
          }
          params[key] = input
        }
        api('addAccountToProject', params).then(response => {
          this.$pollJob({
            jobId: response.addaccounttoprojectresponse.jobid,
            successMessage: `Successfully added account ${params.account} to project`,
            errorMessage: `Failed to add account: ${params.account} to project`,
            loadingMessage: `Adding Account: ${params.account} to project...`,
            catchMessage: 'Error encountered while fetching async job result'
          })
          this.closeAction()
        }).catch(error => {
          this.$notifyError(error)
        }).finally(() => {
          this.loading = false
        })
      })
    },
    addUserToProject (e) {
      e.preventDefault()
      if (this.loading) return
      this.form.validateFieldsAndScroll((err, values) => {
        if (err) {
          return
        }

        this.loading = true
        var params = {
          projectid: this.resource.id
        }
        for (const key in values) {
          const input = values[key]
          if (input === undefined) {
            continue
          }
          params[key] = input
        }
        api('addUserToProject', params).then(response => {
          this.$pollJob({
            jobId: response.addusertoprojectresponse.jobid,
            successMessage: `Successfully added user ${params.username} to project`,
            errorMessage: `Failed to add user: ${params.username} to project`,
            loadingMessage: `Adding User ${params.username} to project...`,
            catchMessage: 'Error encountered while fetching async job result'
          })
          this.closeAction()
        }).catch(error => {
          console.log('catch')
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
<style lang="scss" scoped>
  .form-layout {
    width: 80vw;

    @media (min-width: 600px) {
      width: 450px;
    }
  }
</style>
