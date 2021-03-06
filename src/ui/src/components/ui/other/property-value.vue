<template>
    <user-value :value="value" v-if="isUser"></user-value>
    <compmoent :is="tag" v-bind="attrs" :class="`value-${theme}-theme`" v-else>{{displayValue}}</compmoent>
</template>

<script>
    import UserValue from './user-value'
    const ORG_CACHES = {}
    export default {
        name: 'cmdb-property-value',
        components: {
            UserValue
        },
        props: {
            value: {
                type: [String, Number, Array, Boolean],
                default: ''
            },
            property: {
                type: [Object, String],
                default: () => ({})
            },
            options: {
                type: [Array, String, Object],
                default: () => ([])
            },
            showUnit: {
                type: Boolean,
                default: true
            },
            showTitle: {
                type: Boolean,
                default: true
            },
            tag: {
                type: String,
                default: 'span'
            },
            className: {
                type: String,
                default: ''
            },
            theme: {
                type: String,
                default: 'default',
                validator (value) {
                    return ['primary', 'default'].includes(value)
                }
            }
        },
        data () {
            return {
                displayValue: ''
            }
        },
        computed: {
            attrs () {
                const attrs = {}
                if (this.className) {
                    attrs.class = this.className
                }
                if (this.showTitle) {
                    attrs.title = this.displayValue
                }

                return attrs
            },
            isUser () {
                const type = typeof this.property === 'object' ? this.property.bk_property_type : this.property
                return type === 'objuser'
            }
        },
        watch: {
            value (value) {
                this.setDisplayValue(value)
            }
        },
        created () {
            this.setDisplayValue(this.value)
        },
        methods: {
            async setDisplayValue (value) {
                if (this.isUser) return
                let displayValue
                const isPropertyObject = Object.prototype.toString.call(this.property) === '[object Object]'
                const type = isPropertyObject ? this.property.bk_property_type : this.property
                const unit = isPropertyObject ? this.property.unit : ''
                if (type === 'organization') {
                    displayValue = await this.getOrganization(value)
                } else {
                    displayValue = this.$options.filters['formatter'](value, this.property, this.options)
                }
                this.displayValue = (this.showUnit && unit && displayValue !== '--') ? `${displayValue}${unit}` : displayValue
            },
            async getOrganization (value) {
                let displayValue
                const cacheKey = (value || []).join('_')
                if (ORG_CACHES[cacheKey]) {
                    return ORG_CACHES[cacheKey]
                }

                if (!value || !value.length) {
                    displayValue = '--'
                } else {
                    const res = await this.$store.dispatch('organization/getDepartment', {
                        params: {
                            lookup_field: 'id',
                            exact_lookups: value.join(',')
                        },
                        config: {
                            fromCache: true,
                            requestId: `get_department_id_${cacheKey}`
                        }
                    })
                    const names = (res.results || []).map(item => item.full_name)
                    displayValue = names.join('; ') || '--'
                }

                ORG_CACHES[cacheKey] = displayValue
                return displayValue
            }
        }
    }
</script>

<style lang="scss" scoped>
    .value-primary-theme {
        color: $primaryColor;
        cursor: pointer;
    }
</style>
