<template>
  <div class="table-responsive">
    <table class="table table-bordered table-striped">
      <thead>
      <tr>
        <!--      <th v-for="col in columns" :key="col.field">{{ col.title }}</th>-->
        <TableColumn v-for="col in columns" :column="col" :key="col.field" v-model:sort="sortProp"/>
      </tr>
      </thead>
      <tbody class="bg-white divide-y divide-gray-100">
      <tr v-for="row in rowsProp" :key="row.id">
        <td v-for="col in columns" :key="col.field">{{ getRowValue(col, row) }}</td>
      </tr>
      </tbody>
    </table>
  </div>
</template>

<script>
import TableColumn from "@/components/Table/TableColumn";
import {useVModel} from "@vueuse/core";
import {toRefs} from "vue";

export default {
  name: "BondsTable",
  components: {TableColumn},
  props: {
    rows: {
      type: Array,
      default () {
        return [];
      }
    },
    columns: {},
    sort: {}
  },
  setup (props, {emit}) {
    const {rows: rowsProp} = toRefs(props)
    const sortProp = useVModel(props, "sort", emit)

    const update = () => {
      console.log('update called')
      rowsProp.value.sort((a, b) => {
        if (a[sortProp.value.column] < b[[sortProp.value.column]]) {
          return sortProp.value.descending ? 1 : -1;
        }
        if (a[sortProp.value.column] > b[sortProp.value.column]) {
          return sortProp.value.descending ? -1 : 1;
        }
        return 0;
      })
    }


    return {rowsProp, sortProp, update}
  },
  watch: {
    sortProp: {
      deep: true,
      handler () {
        this.update()
      }
    }
  },
  methods: {
    getRowValue (col, row) {
      if (!col.field) return null
      if (col.compute) return col.compute(row)
      if (typeof col.field === "string") return row[col.field]
      if (typeof col.field === "function") return col.field(row)
    }
  }
}
</script>

<style scoped>

</style>
