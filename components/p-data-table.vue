<template>
    <n-data-table :data="data" :columns="columns" v-bind="$attrs" :pagination="page" :row-class-name="getRowClass"
        @update:checked-row-keys="handleCheck" ref="myTable" :flex-height="height != false" striped
        :scroll-x="disableScollX ? '100%' : 200 * columns.length"
        :row-key="(rowData) => id.map((i) => rowData[i]).join('-')" :row-props="rowProps"
        :style="{ height: height ? `${height}px` : '100%' }"></n-data-table>
</template>
<style scoped>
:deep(.selectedRow td) {
    background-color: #212b59 !important;
    color: white !important;
}
</style>
<script setup>
import {
    SearchOutline,
    CheckmarkCircleOutline,
    BanOutline,
    CalendarClearOutline,
    PencilOutline,
    AddOutline,
    CloseCircleOutline,
    TrashOutline 
} from "@vicons/ionicons5";
import { NSpace, NIcon, NInput, NButton, NDatePicker } from "naive-ui";

const props = defineProps({
    data: { required: true },
    columns: { required: true },
    modelValue: { required: false },
    selectable: { type: Boolean },
    height: { required: false },
    pagination: { required: false },
    id: { required: true, type: Array },
    multiSelection: { type: Boolean },
    disableScollX: { type: Boolean },
    returnObject: { type: Boolean },
    rowProps: { required: false },
    editable: {
        type: Boolean,
        required: false,
        default: false,
    },
});
var defaultRowProps =
    (rowData, rowIndex) => ({
        onClick: () => {
            if (props.selectable) {
                if (props.returnObject) var item = rowData
                else var item = props.id.map((i) => rowData[i]).join('-')
                if (props.multiSelection) {
                    const index = multiSelected.value.indexOf(item);
                    if (index >= 0) multiSelected.value.splice(index, 1);
                    else multiSelected.value.push(item)
                } else {
                    if (selected.value == item) selected.value = null;
                    else selected.value = item;
                }
            }
        },
    })

var rowProps = computed(() => props.rowProps ? props.rowProps : defaultRowProps)
var myTable = ref(null)
function resetFilters() { myTable.value.clearFilters(); props.columns.forEach(c => c.filterOptionValue = null) }
function scrollTo(value) { myTable.value.scrollTo(value) }
function getData() { return myTable.value.mainTableInstRef.bodyInstRef.rawPaginatedData.map(({ __typename, ...o }) => o) }
const selected = computed({
    get: () => props.modelValue,
    set: (value) => emit('update:modelValue', value)
})
defineExpose({
    resetFilters, scrollTo, getData
});
var tempItem = ref({})
var page = computed(() => {
    if (!props.pagination && props.pagination != false) {
        return {
            defaultPageSize: 10,
            showSizePicker: true,
            pageSizes: [10, 25, 50, props.data.length],
            showQuickJumper: true,
        };
    }
    return props.pagination;
});
const emit = defineEmits(["update:modelValue", "checkedRow"]);
function handleCheck(rowKeys) {
    emit("checkedRow", rowKeys)
}
var multiSelected = ref([])

watch(
    selected,
    (newValue, oldValue) => {
        emit("update:modelValue", newValue);
    },
    { deep: true }
);
watch(multiSelected.value, (newValue) => { emit("update:modelValue", newValue); }, { deep: true })


function getRowClass(rowData) {
    if (props.selectable) {
        if (props.returnObject) var item = rowData
        else var item = props.id.map((i) => rowData[i]).join('-')
        if (props.multiSelection) {
            const index = multiSelected.value.indexOf(item);
            if (index >= 0) return " selectedRow "
            else return ""
        } else {
            if (selected.value == item) return " selectedRow "
            else return ""
        }
    }
}

function getValue(field, value) {
    var steps = field.split(".");
    var v = value;
    var step = 0;
    while (step < steps.length) {
        v = v[steps[step]];
        if (!v) return v;
        step++;
    }
    return v;
}
function processColumns() {
    props.columns.forEach((field) => {
        if (field.resizable != false) field.resizable = true;
        field.sortable = true;
        if (field.type == "Bool") field.filter = "category";
        if (field.type == "Date") field.filter = "dateRange";
        if (field.filter == "category") {
            if (field.type == "Bool")
                var temp = [
                    { label: "Y", value: true },
                    { label: "N", value: false },
                ];
            else
                var temp = [...new Set(props.data.map((o) => getValue(field.key, o)))]
                    .sort()
                    .map((o) => {
                        return o
                            ? {
                                label:
                                    typeof o === "string"
                                        ? o.charAt(0).toUpperCase() + o.slice(1)
                                        : "N/A",
                                value: o,
                            }
                            : {};
                    });
            field.filterOptions = temp;
            field.filter = (value, row) => {
                return getValue(field.key, row) == value;
            };
        }
        if (field.type == "Bool") {
      field.render = (row, index) => {
        if (
          props.editable &&
          row.editable &&
          (!field.readOnly || row.new) &&
          !field.auto
        ) {
          return h(NSwitch, {
            value: tempItem[field.key],
            onUpdateValue(v) {
              tempItem[field.key] = v
            },
          })
        } else if (row[field.key] == true)
          return h(
            NIcon,
            { color: "green", size: 18 },
            { default: () => h(CheckmarkCircleOutline) }
          )
        else if (row[field.key] == false)
          return h(NIcon, { color: "red", size: 18 }, { default: () => h(BanOutline) })
        else return h(NIcon, { color: "orange", size: 18 }, { default: () => h(Warning) })
      }
    }
    if (field.type == "autocomplete" && props.editable) {
      field.render = (row, index) => {
        if ((!field.readOnly || row.new) && !field.auto && row.editable)
          return h(BAutocomplete, {
            ...field,
            modelValue: tempItem[field.key],
            multiple: false,
            onUpdateValue(v) {
              tempItem[field.key] = v
            },
          })
        return getValue(field.key, row)
      }
    }
    if (!field.type && props.editable) {
      field.render = (row, index) => {
        if ((!field.readOnly || row.new) && !field.auto && row.editable)
          return h(NInput, {
            value: tempItem[field.key],
            onUpdateValue(v) {
              tempItem[field.key] = v
            },
          })
        return getValue(field.key, row)
      }
    }
    if (field.type == "Int") {
      field.render = (row, index) => {
        if (
          props.editable &&
          row.editable &&
          (!field.readOnly || row.new) &&
          !field.auto
        ) {
          return h(NInputNumber, {
            value: tempItem[field.key],
            onUpdateValue(v) {
              tempItem[field.key] = v
            },
          })
        }
        return getValue(field.key, row)
      }
    }
    if (!field.filter && field.filter != false) {
      field.filterOptionValue = null
      field.filter = (value, row) => {
        var v = getValue(field.key, row)
        if (field.type == "Set")
          return (
            !value ||
            (v && v.some((va) => va.toLowerCase().indexOf(value.toLowerCase()) >= 0))
          )
        return !value || (v && v.toLowerCase().indexOf(value.toLowerCase()) >= 0)
      }
      field.renderFilterIcon = () => {
        return h(NIcon, null, { default: () => h(SearchOutline) })
      }
      field.renderFilterMenu = ({ hide }) => {
        return h(
          NSpace,
          { style: { padding: "12px" }, vertical: true },
          {
            default: () => [
              h(NInput, {
                placeholder: "Text Search",
                value: field.filterOptionValue,
                onKeyup: (key) => {
                  if (key.code == "Enter") hide()
                },
                onInput: (text) => {
                  field.filterOptionValue = text
                },
              }),
              h(
                NButton,
                {
                  size: "tiny",
                  onClick: () => {
                    field.filterOptionValue = null
                    hide()
                  },
                },
                { default: () => "Clear" }
              ),
            ],
          }
        )
      }
    }
    if (field.type == "Set") {
      field.render = (row) => {
        var temp = getValue(field.key, row)
        if (
          props.editable &&
          row.editable &&
          (!field.readOnly || row.new) &&
          !field.auto
        ) {
          return h(NDynamicInput, {
            value: tempItem[field.key],
            onUpdateValue(v) {
              tempItem[field.key] = v
            },
          })
        }
        return temp ? temp.join(", ") : ""
      }
    }
    if (field.type == "Currency") {
      field.align = "right"
      field.render = (row) =>
        row[field.key] == 0
          ? "-"
          : row[field.key].toLocaleString("pt-br", { minimumFractionDigits: 2 })
    }
    if (field.type == "Date") {
      field.render = (row) => {
        if (
          props.editable &&
          row.editable &&
          (!field.readOnly || row.new) &&
          !field.auto
        ) {
          return h(NDatePicker, {
            format: "dd/MM/yyyy",
            valueFormat: "yyyy-MM-dd",
            defaultFormattedValue: tempItem[field.key],
            onUpdateFormattedValue(v) {
              tempItem[field.key] = v
            },
          })
        }
        return row[field.key]
          ? new Date(row[field.key] + " GMT-0300").toLocaleDateString("pt-BR")
          : ""
      }
      field.filterOptionValue = null
      field.renderFilterIcon = () => {
        return h(NIcon, null, { default: () => h(CalendarClearOutline) })
      }
      field.filter = (value, row) => {
        return (
          !field.filterOptionValue ||
          (row[field.key] &&
            new Date(row[field.key] + " GMT-0300") >=
              new Date(field.filterOptionValue[0]) &&
            new Date(row[field.key] + " GMT-0300") <=
              new Date(field.filterOptionValue[1]))
        )
      }
      field.renderFilterMenu = ({ hide }) => {
        return h(
          NSpace,
          { style: { padding: "12px" }, vertical: true },
          {
            default: () => [
              h(NDatePicker, {
                type: "daterange",
                value: field.filterOptionValue,
                clearable: true,
                onConfirm: ([min, max]) => {
                  field.filterOptionValue = [min, max]
                  hide()
                },
              }),
              h(
                NButton,
                {
                  size: "tiny",
                  onClick: () => {
                    field.filterOptionValue = null
                    hide()
                  },
                },
                { default: () => "Clear" }
              ),
            ],
          }
        )
      }
    }
  })

  if (props.editable) {
    var col = {
      key: "actions",
      filter: false,
      title: () => {
        return h(
          NButton,
          {
            circle: true,
            strong: true,
            tertiary: true,
            type: "success",
            onClick: () => {
              if (!props.data.some((i) => i.new)) {
                props.data.forEach((r) => (r.editable = false))
                tempItem = { editable: true, new: true }
                props.data.unshift(tempItem)
              }
            },
          },
          {
            icon: () =>
              h(
                NIcon,
                {
                  color: "green",
                  size: 24,
                },
                {
                  default: () => h(AddOutline),
                }
              ),
          }
        )
      },
      render: (row) => {
        return row.editable
          ? [
              h(
                NButton,
                {
                  circle: true,
                  strong: true,
                  tertiary: true,
                  type: "success",
                  onClick: () => {
                    dialog.warning({
                      title: `Confirmação de ${row.new ? "Inclusão" : "Alteração"}`,
                      content: `Tem certeza que deseja efetuar ${
                        row.new ? "a Inclusão" : "as alterações"
                      }?`,
                      positiveText: "SIM",
                      negativeText: "NÃO",
                      onPositiveClick: () => {
                        message.success(
                          `Item ${row.new ? "incluido" : "alterado"} com sucesso`
                        )
                        emit("put", tempItem)
                        row.editable = false
                        tempItem = {}
                      },
                      onNegativeClick: () => {},
                    })
                  },
                },
                {
                  icon: () =>
                    h(
                      NIcon,
                      { color: "green", size: 22 },
                      {
                        default: () => h(CheckmarkCircleOutline),
                      }
                    ),
                }
              ),
              h(
                NButton,
                {
                  circle: true,
                  strong: true,
                  tertiary: true,
                  type: "warning",
                  onClick: () => {
                    if (row.new) props.data.splice(0, 1)
                    row.editable = false
                    tempItem = {}
                  },
                },
                {
                  icon: () =>
                    h(
                      NIcon,
                      { color: "orange", size: 22 },
                      {
                        default: () => h(CloseCircleOutline),
                      }
                    ),
                }
              ),
              h(
                NButton,
                {
                  circle: true,
                  strong: true,
                  tertiary: true,
                  type: "error",
                  onClick: () => {
                    dialog.error({
                      title: "Confirmação de Exclusão",
                      content: "Tem certeza que deseja deletar este item?",
                      positiveText: "SIM",
                      negativeText: "NÃO",
                      onPositiveClick: () => {
                        message.success("Item removido com sucesso")
                        emit("delete", tempItem)
                        row.editable = false
                        tempItem = {}
                      },
                      onNegativeClick: () => {},
                    })
                  },
                },
                {
                  icon: () =>
                    h(
                      NIcon,
                      { color: "red", size: 22 },
                      {
                        default: () => h(TrashOutline),
                      }
                    ),
                }
              ),
            ]
          : h(
              NButton,
              {
                circle: true,
                strong: true,
                tertiary: true,
                type: "info",
                onClick: () => {
                  row.editable = true
                  tempItem = ref(structuredClone(toRaw(row))).value
                  props.data.forEach((r) => (r.editable = false))
                  row.editable = true
                },
              },
              {
                icon: () =>
                  h(
                    NIcon,
                    {
                      color: "blue",
                      size: 20,
                    },
                    { default: () => h(PencilOutline) }
                  ),
              }
            )
      },
    }
    if (props.columns.some((f) => f.key == "actions")) props.columns.splice(-1)
    props.columns.push(col)
  }
}
watch(
    () => props.data,
    (first, second) => {
        if (first.length > 0) processColumns();
    }
);
watch(
    () => props.columns,
    (first, second) => {
        if (first.length > 0) processColumns();
    }
);
</script>
