<template>
    <div class="card" :class="{ expanded: isExpanded }" @click="toggleExpand">
        <div class="header">
            <h2>{{ title }}</h2>
        </div>
        <div class="content" v-if="isExpanded">
            <slot></slot>
        </div>
    </div>
</template>  
<style scoped>
.card {
    border-radius: 8px;
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
    padding: 16px;
    background-color: #ffffff;
    cursor: pointer;
    transition: all 0.3s ease;
}

.card.expanded {
    width: 100%;
    height: auto;
}

.header {
    display: flex;
    justify-content: space-between;
    align-items: center;
}

.content {
    margin-top: 16px;
}
</style>
<script>
export default {
    data() {
        return {
            isExpanded: false,
        };
    },
    props: {
        title: {
            type: String,
            required: true,
        },
    },
    methods: {
        toggleExpand() {
            this.isExpanded = !this.isExpanded;
        },
    },
};
</script>
  