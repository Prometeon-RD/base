<template>
    <n-data-table :data="data" :columns="columns" v-bind="$attrs" :pagination="page" :row-class-name="getRowClass"
        @update:checked-row-keys="handleCheck" ref="myTable" flex-height striped :scroll-x="200 * 20"
        :row-key="(rowData) => id.map((i) => rowData[i]).join('-')" :row-props="rowProps"
        :style="{ height: `${height}px`}"></n-data-table>
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
});
var rowProps =
    (rowData, rowIndex) => ({
        onClick: () => {
            if (props.selectable) {
                const item = props.id.map((i) => rowData[i]).join('-')
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

var page = computed(() => {
    if (!props.pagination && props.pagination != false) {
        return {
            defaultPageSize: 10,
            showSizePicker: true,
            pageSizes: [10, 25, 50],
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
        const item = props.id.map((i) => rowData[i]).join('-')
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
    while (step < 10) {
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
            field.render = (row) => {
                if (getValue(field.key, row))
                    return h(
                        NIcon,
                        { color: "green", size: 18 },
                        { default: () => h(CheckmarkCircleOutline) }
                    );
                else
                    return h(NIcon, { color: "red", size: 18 }, { default: () => h(BanOutline) });
            };
        }
        if (!field.filter && field.filter != false) {
            field.filterOptionValue = null;
            field.filter = (value, row) => {
                var v = getValue(field.key, row).toString();
                return !value || (v && v.toLowerCase().indexOf(value.toLowerCase()) >= 0);
            };
            field.renderFilterIcon = () => {
                return h(NIcon, null, { default: () => h(SearchOutline) });
            };
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
                                    if (key.code == "Enter") hide();
                                },
                                onInput: (text) => {
                                    field.filterOptionValue = text;
                                },
                            }),
                            h(
                                NButton,
                                {
                                    size: "tiny",
                                    onClick: () => {
                                        field.filterOptionValue = null;
                                        hide();
                                    },
                                },
                                { default: () => "Clear" }
                            ),
                        ],
                    }
                );
            };
        }
        if (field.type == "Currency") {
            field.align = "right";
            field.render = (row) =>
                getValue(field.key, row).toLocaleString("pt-br", { minimumFractionDigits: 2 });
        }
        if (field.type == "Date") {
            field.render = (row) => { var v = getValue(field.key, row); return v ? new Date(v).toLocaleDateString("pt-BR") : "" }

            field.filterOptionValue = null;
            field.renderFilterIcon = () => {
                return h(NIcon, null, { default: () => h(CalendarClearOutline) });
            };
            field.filter = (value, row) => {
                var v = getValue(field.key, row);
                return (
                    !field.filterOptionValue ||
                    (v &&
                        new Date(v) >= new Date(field.filterOptionValue[0]) &&
                        new Date(v) <= new Date(field.filterOptionValue[1]))
                );
            };
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
                                    field.filterOptionValue = [min, max];
                                    hide();
                                },
                            }),
                            h(
                                NButton,
                                {
                                    size: "tiny",
                                    onClick: () => {
                                        field.filterOptionValue = null;
                                        hide();
                                    },
                                },
                                { default: () => "Clear" }
                            ),
                        ],
                    }
                );
            };
        }
    });
}
watch(
    () => props.data,
    (first, second) => {
        // if (first.length > 0) processColumns();
    }
);
</script>