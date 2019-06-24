<script>
    import _ from 'lodash'
    import Color from 'chartjs-color'
    export default {
        data(){
            return {
                items: null,
                originalItems: [], // used to show the original list without any sorting applied
                widthArray: [],
                showFooter: false,
                totals: [],
                height: 0,
                dynamicMarginRight: 0,
                showRightBorder: true,
                sortedBy: {
                    fieldName: null,
                    direction: null
                },
                timeoutIdentifier: null,
                dynamicBackgroundTotals: {}
            }
        },
        props: {
            striped: {
                type: Boolean,
                default: true
            },
            columns: {
                type: Array,
                default: () => []
            },
            data: {
                type: Array,
                default: () => []
            },
            closable: {
                type: Boolean,
                default: false
            },
            columnsSameWidth: {
                type: Boolean,
                default: false
            },
            expandedData: {
                type: Array,
                default: () => []
            },
            loadingAnimation: {

            },
            title: {
                type: String,
                default: ""
            },
            bogusLines: {
                default: false,
                type: Boolean
            },

        },
        methods: {
            updateCells(){

                // check for columns total
                this.showFooter = false
                _.each(this.columns, i => {
                    if(i.total && ['sum', 'count'].indexOf(i.total) !== -1){
                        this.showFooter = true
                    }
                    if(i.footerText){
                        this.showFooter = true
                    }
                })
                if(this.showFooter){
                    this.calculateTotals()
                }
                if(!this.items) return
                requestAnimationFrame(() => {
                    this.calculateColumnsWidth()
                    this.updateColumnsWidth()


                    this.fixBodyHeight()
                    this.fixDynamicMarginRight()

                })

            },
            calculateTotals(){
                this.totals = []
                //console.log(this.columns)
                _.each(this.items, (item, k) => {
                    //console.log(item)


                    _.each(this.columns, (column, i) => {
                        if(column.total && ['sum', 'count', 'avg'].indexOf(column.total) !== -1){
                            this.totals[i] = this.totals[i] || 0
                            if(column.total === 'sum') {
                                this.totals[i] += item[column.value] ? item[column.value] : 0
                            } else if(column.total === 'avg'){
                                this.totals[i] += (item[column.value] ? item[column.value] / this.items.length: 0)
                            } else {
                                this.totals[i] = item.length
                            }
                        } else if(column.footerText){
                            this.totals[i] = column.footerText
                        } else {
                            this.totals[i] = null
                        }
                    })

                })

                //console.log(this.totals)
            },
            calculateColumnsWidth(){
                if(this.columnsSameWidth){
                    //console.log(this.columns)
                    let visibleColumns = _.filter(this.columns, e => { return e.hidden !== true })
                    let totalWidth = this.$el.clientWidth
                    //console.log(visibleColumns)
                    this.widthArray = _.map(visibleColumns, (c, k) => {
                        if(k === visibleColumns.length - 1){ // if it's the last column, i have to reduct it by the width of the scrollbar
                            return (totalWidth / visibleColumns.length) - .5 - (this.$refs.rows.offsetWidth - this.$refs.rows.clientWidth)
                        } else {
                            return (totalWidth / visibleColumns.length) - .5
                        }

                    })
                } else {
                    // consolidate columns width
                    let headers = this.$refs.headers.children
                    let rows = this.$refs.rows.children[0].children

                    _.forEach(headers, (header, k) => {
                        let max = header.offsetWidth + 20
                        _.forEach(rows, row => {

                            let cell = row.children[k]
                            max = Math.max(max, cell.offsetWidth)
                        })
                        this.widthArray[k] = max
                    })
                    //console.log(this.widthArray)
                }


            },
            updateColumnsWidth(){
                let rows = this.$refs.rows.children[0].children
                let headers = this.$refs.headers.children

                _.forEach(headers, (header, k) => {
                    header.style.width = (this.widthArray[k]) + 'px'
                })

                _.forEach(rows, row => {
                    if(row.children[0]){
                        _.forEach(row.children[0].children, (cell, k) => {
                            let width = this.widthArray[k]
                            if(k === row.children[0].children.length - 1) {
                                width -=10
                            }
                            cell.style.width =  `${width}px`
                            // cell.style.height = `${this.$refs.headers.clientHeight}px`
                        })
                    }

                    if(row.children[1]){
                        _.forEach(row.children[1].children, (subrow, k) => {
                            _.forEach(subrow.children, (subcell, k) => {
                                let width = this.widthArray[k]
                                if(k === subrow.children.length - 1) {
                                    width -=10
                                }
                                subcell.style.width =  `${width}px`
                                // cell.style.height = `${this.$refs.headers.clientHeight}px`
                            })
                        })
                    }
                })

                // have to reset all heights
                _.forEach(headers, (c, j) => {
                    c.style.height = `auto`
                })
                let height = this.$refs.headers.clientHeight

                // update headers cell height
                _.forEach(headers, (c, j) => {
                    c.style.height = `${height}px`
                })



                if(this.showFooter){
                    let footerItems = this.$refs.footer.children
                    _.forEach(footerItems, (totalCell, k) => {
                        totalCell.style.width = (this.widthArray[k]) + 'px'
                    })
                }
            },
            fixBodyHeight(){
                this.$refs.rows.style.height = ((this.$el.offsetHeight - this.$refs.headers.offsetHeight - (this.$refs.footer ? this.$refs.footer.offsetHeight : 0)) - (this.title !== '' ? 30 : 0)) - 10 + 'px'
            },

            fixDynamicMarginRight(){
                // the right margin of the totals row is affected by the scrollbar, if any
                this.dynamicMarginRight =  this.$refs.rows.offsetWidth - this.$refs.rows.clientWidth
                //console.log([this.$refs.rows], this.$refs.rows.offsetWidth, this.$refs.rows.clientWidth, this.dynamicMarginRight)
            },

            fixRightBorderVisibility(){
                setTimeout(() => {
                    this.showRightBorder = !this.$refs.rows || this.$refs.rows.children[0].offsetHeight < this.$refs.rows.offsetHeight;
                }, 200)

            },
            sortBy(header){
                this.sortedBy = {
                    fieldName: header.value,
                    direction: (() => {
                        if(this.sortedBy.fieldName !== header.value ){
                            return 'asc'
                        } else {
                            if(this.sortedBy.direction === 'desc')
                                return 'asc'
                            if(this.sortedBy.direction === 'asc')
                                return null
                            if(this.sortedBy.direction === null)
                                return 'desc'
                        }

                    })()
                }

                if(this.sortedBy.direction === 'asc'){
                    this.items = _.sortBy(this.items, this.sortedBy.fieldName)
                    this.items = _.reverse(this.items)
                } else if(this.sortedBy.direction === 'desc'){
                    this.items = _.sortBy(this.items, this.sortedBy.fieldName)
                } else {
                    this.items = _.cloneDeep(this.originalItems)
                }
                requestAnimationFrame(() => { this.updateColumnsWidth() })
            },
            initItems(data){
                data = _.cloneDeep(data)
                this.dynamicBackgroundTotals = {}
                _.each(data, (e, k) => {
                    _.forOwn(e, (value, fieldName) => {
                        this.dynamicBackgroundTotals[fieldName] = this.dynamicBackgroundTotals[fieldName] || 0
                        let column = _.find(this.columns, { value: fieldName })

                        if(column && column.dynamicBackground){
                            this.dynamicBackgroundTotals[fieldName] += value
                        }
                    })

                    if(this.expandedData[k]){
                        e.expandedData = this.expandedData[k]
                        e.collapsed = true
                    }
                })
                this.items = data
                this.originalItems = data

                requestAnimationFrame(() => {
                    this.updateCells()
                    this.bogusLines && this.updateBogusLines()
                })
                // console.log(this.dynamicBackgroundTotals)
            },
            getColumn(columnId){
                return _.find(this.columns, {value: columnId})
            },
            toggleExpandedData(row){
                row.collapsed = !row.collapsed
                this.fixRightBorderVisibility()
                setTimeout(() => {
                    this.bogusLines && this.updateBogusLines()
                }, 500)

            },
            updateBogusLines(){
                let rowsContainer = this.$refs.rows
                if(!rowsContainer) return
                let rows = rowsContainer.children[0]

                // remove all bogus lines first
                _.forEach(rows.getElementsByClassName('flex-grid-row--bogus'), elem => {
                    if(elem){
                        elem.parentNode.removeChild(elem)
                    }
                })
                if(rows.children.length <= 0){
                    return
                }

                while(rows.clientHeight < rowsContainer.clientHeight){
                    let bogusLine = document.createElement('div')
                    bogusLine.classList.add('flex-grid-row--bogus')
                    bogusLine.classList.add('flex-grid-row')

                    if(rowsContainer.clientHeight - rows.clientHeight < 32){
                        bogusLine.style.height = `${rowsContainer.clientHeight - rows.clientHeight - 1}px`
                        rows.appendChild(bogusLine)
                        break
                    } else {
                        // for(let i = 0; i < rows.children[0].children.length; i++){
                        // 	let bogusCell = document.createElement('div')
                        // 	bogusCell.classList.add('flex-grid-cell')
                        // 	bogusCell.innerHTML = '&nbsp;'
                        // 	bogusLine.appendChild(bogusCell)
                        // }
                        rows.appendChild(bogusLine)
                    }

                }

            },
            onAfterResize(){
                if(!this.$refs.rows || !this.$refs.headers) return

                if(this.columnsSameWidth){
                    this.calculateColumnsWidth()
                    this.updateColumnsWidth()
                }
                this.fixBodyHeight()
                this.fixDynamicMarginRight()
            },
            getBackgroundColor(fieldName, value, baseColor){
                // console.log(value, this.dynamicBackgroundTotals[fieldName])
                let factor = (value / this.dynamicBackgroundTotals[fieldName])
                factor = Math.min(factor, .3)
                baseColor = baseColor || 'white'
                return Color(baseColor).darken(factor).rgbString()
            }
        },
        watch: {
            data: {
                handler(data){
                    this.initItems(data)
                },
                deep: true
            }
        },
        computed: {

        },
        beforeMount(){

        },
        mounted(){
            const resizeObserver = new ResizeObserver(entries => {
                this.onAfterResize()
                clearTimeout(this.timeoutIdentifier)
                if(this.bogusLines) this.timeoutIdentifier = setTimeout(this.updateBogusLines, 1000)

            });
            requestAnimationFrame(() => {
                resizeObserver.observe(this.$el);
                this.initItems(this.data)
            })

        }
    }
</script>
<template>
    <div class="flex-grid" :class="{striped: striped}" :style="{borderRight: showRightBorder ? '1px solid #e6e6e6' : 'none'}">
        <div class="flex-grid-title" v-if="title !== ''">
            <span>{{title}}</span>
            <div class="flex-grid-close-button" v-if="closable" @click="$emit('onClose')">
                X
            </div>
        </div>

        <div v-if="items === null" v-bind:is="loadingAnimation" :show="true">

        </div>
        <div class="flex-grid-headers" ref="headers" v-if="items !== null">
            <div
                class="flex-grid-header"
                :class="{sortable: header.sortable}"
                v-for="(header, headerId) in columns"
                :key="headerId"
                v-if="!header.hidden"
                @click="sortBy(header)"
                :style="{justifyContent: [header.headerTextAlign], textAlign: [header.headerTextAlign]}"
            >
                <div>{{header.title}}</div>
                <span v-if="header.sortable && sortedBy.fieldName === header.value" :class="{'caret-down': sortedBy.direction === 'asc', 'caret-up': sortedBy.direction === 'desc'}"></span>
            </div>
        </div>
        <div ref="rows" class="flex-grid-rows">
            <div class="flex-grid-rows-scrollable-container">
                <div class="flex-grid-row"
                     v-for="(row, rowId) in items"
                     :key="rowId"
                     :class="{
                        expanded: !row.collapsed,
                        expandable: row.expandedData,
                        'odd-children': row.expandedData && row.expandedData.length % 2 !== 0,
                        'even-children': row.expandedData && row.expandedData.length % 2 === 0,
                        }
                ">

                    <div style="width: 100%;">
                        <div
                            class="flex-grid-cell"

                            v-for="(column, columnId) in columns"
                            :key="columnId"
                            v-if="!column.hidden"
                            :style="{
								textAlign: [column.align ? column.align : 'left'],
								paddingLeft: !row.expandedData && columnId === 1 ? '24px' : '10px',
								backgroundColor: column.dynamicBackground ? getBackgroundColor(column.value, row[column.value], column.dynamicBackgroundColor) : 'transparent'
							}"
                        >
                            <span
                                v-if="row.expandedData && columnId === 0"
                                :class="{'caret-right': row.collapsed, 'caret-down': !row.collapsed}"
                                @click="toggleExpandedData(row)"
                            ></span>
                            <span v-html="columns[columnId].renderer && row[column.value] ? columns[columnId].renderer(row[column.value], row) : row[column.value]">

                            </span>
                        </div>
                    </div>
                    <div v-if="row.expandedData" class="flex-grid-expanded-container" :class="{collapsed: row.collapsed}">
                        <div v-for="(subRow, subRowId) in row.expandedData" class="flex-grid-expanded-row" :key="subRowId">
                            <div
                                v-if="!column.hidden"
                                v-for="(column, columnId) in columns"
                                class="flex-grid-expanded-cell"
                                :key="columnId"
                                :style="{
										textAlign: [column.align ? column.align : 'left'],
										backgroundColor: column.dynamicBackground ? getBackgroundColor(column.value, subRow[column.value], column.dynamicBackgroundColor) : 'transparent'
									}"
                            >
                                <span v-html="subRow[column.value] ? (column.renderer ? column.renderer(subRow[column.value], subRow) : subRow[column.value]) : ''">

                                </span>
                            </div>
                        </div>
                    </div>
                </div>
            </div>

        </div>
        <div ref="footer" v-if="showFooter" class="flex-grid-footer" :style="{paddingRight: [dynamicMarginRight + 'px']}">
            <div
                class="flex-grid-footer-cell"
                v-for="(column, columnId) in columns"
                :key="columnId"
                v-if="!column.hidden"
                :style="{textAlign: [column.footerTextAlign ? column.footerTextAlign : (column.align ? column.align : 'left')]}"

            >
                {{columns[columnId].footerRenderer && totals[columnId] ? columns[columnId].footerRenderer(totals[columnId]) : totals[columnId]}}
            </div>
        </div>
    </div>
</template>
<style lang="scss">
    .flex-grid {
        width: 100%;
        position: absolute;
        background: #ffffff;
        height: 100%;
        overflow: hidden;
        border-left: 1px solid #e6e6e6;
        border-top: 1px solid #e6e6e6;
        border-bottom: 1px solid #e6e6e6;

        &.striped {

            /*.flex-grid-row:nth-child(even) {*/
            /*    background-color: #f9f9f9;*/
            /*    .flex-grid-expanded-row:nth-child(odd) {*/
            /*        background-color: #ffffff;*/
            /*    }*/
            /*    .flex-grid-expanded-row:nth-child(even) {*/
            /*        background-color: #f9f9f9;*/
            /*    }*/
            /*}*/

            /*.flex-grid-row:nth-child(odd) {*/
            /*    background-color: #ffffff;*/
            /*    .flex-grid-expanded-row:nth-child(odd) {*/
            /*        background-color: #f9f9f9;*/
            /*    }*/
            /*    .flex-grid-expanded-row:nth-child(even) {*/
            /*        background-color: #ffffff;*/
            /*    }*/
            /*}*/
            .flex-grid-row {

                &.expandable {
                    &:nth-child(odd) {
                        background-color: #f9f9f9 !important;
                        &+ .flex-grid-row {
                            .flex-grid-expanded-row:nth-child(even) {
                                background-color: #ffffff!important;
                            }
                            .flex-grid-expanded-row:nth-child(odd) {
                                background-color: #f9f9f9!important;
                            }
                        }
                    }
                    &:nth-child(even) {
                        background-color: #ffffff !important;
                    }
                    &.expanded {
                        &.odd-children {
                            &:nth-child(odd) {
                                background-color: #f9f9f9 !important;
                                .flex-grid-expanded-row:nth-child(even) {
                                    background-color: #ffffff!important;
                                }
                                .flex-grid-expanded-row:nth-child(odd) {
                                    background-color: #f9f9f9!important;
                                }
                                &+ .flex-grid-row:nth-child(odd) {
                                    background-color: #f9f9f9!important;
                                    .flex-grid-expanded-row:nth-child(even) {
                                        background-color: #f9f9f9!important;
                                    }
                                    .flex-grid-expanded-row:nth-child(odd) {
                                        background-color: #ffffff!important;
                                    }
                                }
                                &+ .flex-grid-row:nth-child(even) {
                                    background-color: #ffffff!important;
                                    .flex-grid-expanded-row:nth-child(even) {
                                        background-color: #ffffff!important;
                                    }
                                    .flex-grid-expanded-row:nth-child(odd) {
                                        background-color: #f9f9f9!important;
                                    }
                                }
                            }
                            &:nth-child(even) {
                                background-color: #ffffff !important;
                                &+ .flex-grid-row:nth-child(even) {
                                    background-color: #f9f9f9!important;
                                    .flex-grid-expanded-row:nth-child(even) {
                                        background-color: #f9f9f9!important;
                                    }
                                    .flex-grid-expanded-row:nth-child(odd) {
                                        background-color: #ffffff!important;
                                    }
                                }
                                &+ .flex-grid-row:nth-child(odd) {
                                    background-color: #ffffff!important;
                                    .flex-grid-expanded-row:nth-child(even) {
                                        background-color: #ffffff!important;
                                    }
                                    .flex-grid-expanded-row:nth-child(odd) {
                                        background-color: #f9f9f9!important;
                                    }
                                }
                            }

                        }
                        &.even-children {
                            &:nth-child(odd) {
                                background-color: #f9f9f9!important;
                                .flex-grid-expanded-row:nth-child(odd) {
                                    background-color: #ffffff!important;
                                }
                                .flex-grid-expanded-row:nth-child(even) {
                                    background-color: #f9f9f9!important;
                                }
                                &+ .flex-grid-row:nth-child(even) {
                                    background-color: #ffffff!important;
                                    .flex-grid-expanded-row:nth-child(even) {
                                        background-color: #ffffff!important;
                                    }
                                    .flex-grid-expanded-row:nth-child(odd) {
                                        background-color: #f9f9f9!important;
                                    }
                                }
                                &+ .flex-grid-row:nth-child(odd) {
                                    background-color: #f9f9f9!important;
                                    .flex-grid-expanded-row:nth-child(even) {
                                        background-color: #f9f9f9!important;
                                    }
                                    .flex-grid-expanded-row:nth-child(odd) {
                                        background-color: #ffffff!important;
                                    }
                                }
                            }
                            &:nth-child(even) {
                                &+ .flex-grid-row:nth-child(odd) {
                                    background-color: #ffffff!important;
                                    .flex-grid-expanded-row:nth-child(even) {
                                        background-color: #ffffff!important;
                                    }
                                    .flex-grid-expanded-row:nth-child(odd) {
                                        background-color: #f9f9f9!important;
                                    }
                                }
                                &+ .flex-grid-row:nth-child(even) {
                                    background-color: #f9f9f9!important;
                                    .flex-grid-expanded-row:nth-child(even) {
                                        background-color: #f9f9f9!important;
                                    }
                                    .flex-grid-expanded-row:nth-child(odd) {
                                        background-color: #ffffff!important;
                                    }
                                }
                            }

                        }
                    }
                }
            }
        }
        .flex-grid-row.expandable:not(.expanded):not(.flex-grid-row--bogus):hover,
        .flex-grid-row:not(.flex-grid-row--bogus):not(.expandable):hover,
        .flex-grid-expanded-row:not(.flex-grid-row--bogus):hover {
            background-color: rgba(230,230,230,1)!important;
            .flex-grid-cell {
                background-color: rgba(230,230,230,1)!important;
            }
        }





        .flex-grid-headers {
            background-color: #e6e6e6;
            border-bottom: 1px solid #e0e0e0;

            .flex-grid-header {
                display: inline-flex;
                min-width: 20px;
                padding: 5px 10px;
                font-weight: bold;
                position: relative;
                line-height: 18px;

                &:not(:last-child)::after {
                    content: '';
                    height: 20px;
                    border-left: 1px solid #d8d8d8;
                    position: absolute;
                    height: calc(100% - 10px);
                    right: 0;
                }
                &.sortable {
                    cursor: pointer;
                    &:hover {
                        background-color: #e0e0e0;
                    }
                }
            }
        }
        .flex-grid-rows {
            height: 100%;
            overflow-y: auto;
            border-left: 1px solid #e6e6e6;
            .flex-grid-rows-scrollable-container {
                .flex-grid-row {
                    clear: both;
                    width: 100%;
                    display: flex;
                    flex-direction: column;
                    > div {
                        display: inline-flex;
                        clear:both;
                        width:100%;
                    }
                    // height: 30px;
                    &--bogus {
                        height: 33px;
                    }
                    .flex-grid-expanded-container {
                        max-height: 400px;
                        overflow: hidden;
                        transition: all .2s linear;
                        display: flex;
                        flex-direction: column;
                        //padding-bottom: 10px;
                        margin-top: 0;
                        &.collapsed {
                            max-height: 0;
                            padding-bottom: 0;
                            margin-top: 0;
                        }
                        .flex-grid-expanded-row {
                            .flex-grid-expanded-cell {
                                &:first-child {
                                    padding-left: 36px;
                                }
                                display: inline-block;
                                min-width: 20px;
                                padding: 3px 10px;
                                word-break: break-all;
                                white-space: nowrap;
                                font-size: .9rem;
                                overflow: hidden;
                                text-overflow: ellipsis;
                            }
                        }

                    }
                    .flex-grid-cell {
                        overflow: hidden;
                        text-overflow: ellipsis;
                        display: inline-block;
                        min-width: 20px;
                        padding: 5px 10px;
                        word-break: break-all;
                        white-space: nowrap;
                        height: 33px;
                    }
                }
            }
        }


        .flex-grid-footer {
            clear: both;
            width: calc(100% + 5px);
            background-color: #e6e6e6;
            border-top: 1px solid #e0e0e0;
            min-height: 33px;
            margin-left: -5px;
            height: 33px;
            .flex-grid-footer-cell {
                display: inline-block;
                min-width: 20px;
                word-break: break-all;
                white-space: nowrap;
                font-weight: 500;
                padding: 5px 4px 5px 10px;

            }
        }
        .flex-grid-title {
            background-color: transparent;
            width: 100%;
            height: 33px;
            clear: both;
            span {
                display: block;
                padding: 0 10px;
                line-height: 30px;
            }
            .flex-grid-close-button {
                position: absolute;
                right: 5px;
                top: 5px;
                color: white;
                padding: 0px 5px;
                border-radius: 15px;
                line-height: 20px;
                cursor: pointer;
                &:hover {
                    color: #ccc;
                }
            }
        }

    }
    .caret-up {
        width: 0;
        height: 0;
        border-left: 5px solid transparent;
        border-right: 5px solid transparent;
        border-bottom: 5px solid black;
        vertical-align: middle;
        display: inline-block;
        cursor: pointer;
        margin-right: 5px;
    }

    .caret-right {
        width: 0;
        height: 0;
        border-top: 5px solid transparent;
        border-bottom: 5px solid transparent;
        border-left: 5px solid black;
        vertical-align: middle;
        display: inline-block;
        cursor: pointer;
        margin-right: 5px;
    }

    .caret-down {
        width: 0;
        height: 0;
        border-left: 5px solid transparent;
        border-right: 5px solid transparent;
        border-top: 5px solid #000;
        vertical-align: middle;
        display: inline-block;
        cursor: pointer;
        margin-right: 5px;
    }
</style>