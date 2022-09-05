<template>
  <th class="relative">
    <span>{{ column.title }}</span>
    <div class="inline-flex items-start absolute right-2 gap-x-2">
      <span type="button" v-if="column.sortable" @click="changeSort" class="sort-btn">
        <ArrowsUpDownIcon v-if="sort.column !== column.field"/>
        <ArrowUpIcon v-else-if="sort.descending === false"/>
        <ArrowDownIcon v-else-if="sort.descending === true"/>
      </span>
    </div>
  </th>
</template>

<script>
import {toRef} from "vue"
import {
  ArrowDownIcon,
  ArrowUpIcon,
  ArrowsUpDownIcon
} from "@heroicons/vue/24/outline"
import {useVModel} from "@vueuse/core"

export default {
  name: "TableColumn",
  components: {
    ArrowDownIcon,
    ArrowUpIcon,
    ArrowsUpDownIcon,
  },

  props: {
    column: {
      type: Object,
      required: true,
    },
    sort: {},
    filter: {},
  },
  setup (props, {emit}) {
    const column = toRef(props, "column")
    const sort = useVModel(props, "sort", emit)
    const changeSort = () => {
      console.log('change sort')
      if (sort.value.column !== column.value.field) {
        sort.value = {column: column.value.field, descending: false}
      } else if (sort.value.descending === false) {
        sort.value = {column: column.value.field, descending: true}
      } else {
        sort.value = {}
      }
    }

    return {changeSort}
  },
}
</script>

<style>
.sort-btn svg {
  width: 10px
}
</style>
