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
  <a
    v-if="['vm', 'systemvm', 'router', 'ilbvm'].includes($route.meta.name) && 'listVirtualMachines' in $store.getters.apis"
    :href="server + '/console?cmd=access&vm=' + resource.id"
    target="_blank">
    <a-button style="margin-left: 5px" shape="circle" type="dashed" :size="size" :disabled="['Stopped', 'Error', 'Destroyed'].includes(resource.state)" >
      <a-icon type="code" />
    </a-button>
  </a>
</template>

<script>
import Vue from 'vue'
import { SERVER_MANAGER } from '@/store/mutation-types'

export default {
  name: 'Console',
  props: {
    resource: {
      type: Object,
      required: true
    },
    size: {
      type: String,
      default: 'small'
    }
  },
  computed: {
    server () {
      if (!this.$config.multipleServer) {
        return this.$config.apiBase.replace('/api', '')
      }
      const serverStorage = Vue.ls.get(SERVER_MANAGER)
      const apiBase = serverStorage.apiBase.replace('/api', '')
      if (!serverStorage.apiHost || serverStorage.apiHost === '/') {
        return [location.origin, apiBase].join('')
      }
      return [serverStorage.apiHost, apiBase].join('')
    }
  }
}
</script>
