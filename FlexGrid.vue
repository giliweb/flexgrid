<script>
    import _ from 'lodash'
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
                }
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

            }

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
                        if(column.total && ['sum', 'count'].indexOf(column.total) !== -1){
                            this.totals[i] = this.totals[i] || 0
                            if(column.total === 'sum') {
                                this.totals[i] += item[column.value] ? item[column.value] : 0
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
                    this.widthArray = _.map(visibleColumns, c => {
                        return _.floor(totalWidth / visibleColumns.length) - 3
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
                    _.forEach(row.children, (cell, k) => {
                        if(cell.className.indexOf('flex-grid-expanded-container') !== -1){
                            //console.log([cell])
                            _.forEach(cell.children, r => {
                                _.forEach(r.children, (c, j) => {
                                    c.style.width =  (this.widthArray[j]) + 'px'
                                })
                            })
                        } else {
                            cell.style.width =  (this.widthArray[k] ) + 'px'
                        }

                    })
                })

                if(this.showFooter){
                    let footerItems = this.$refs.footer.children
                    _.forEach(footerItems, (totalCell, k) => {
                        totalCell.style.width = (this.widthArray[k]) + 'px'
                    })
                }
            },
            fixBodyHeight(){
                this.$refs.rows.style.height = (this.$el.offsetHeight - this.$refs.headers.offsetHeight - (this.$refs.footer ? this.$refs.footer.offsetHeight : 0)) + 'px'
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
                _.each(data, (e, k) => {
                    if(this.expandedData[k]){
                        e.expandedData = this.expandedData[k]
                        e.collapsed = true
                    }

                })
                this.items = data
                this.originalItems = data

                requestAnimationFrame(() => { this.updateCells() })
            },
            getColumn(columnId){
                return _.find(this.columns, {value: columnId})
            },
            toggleExpandedData(row){
                row.collapsed = !row.collapsed
                this.fixRightBorderVisibility()
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
//	        this.height = this.$el.clientHeight
//			console.log([this.$el])
            //this.initItems(this.data)
            //this.updateCells()
            window.addEventListener('resize', () => {
                if(this.columnsSameWidth){
                    this.calculateColumnsWidth()
                    this.updateColumnsWidth()
                }
                this.fixBodyHeight()
                this.fixDynamicMarginRight()
            })
            //console.log(this.loadingAnimation)
        }
    }
</script>
<template>
	<div class="flex-grid" :class="{striped: striped}" :style="{borderRight: showRightBorder ? '1px solid #e6e6e6' : 'none'}">
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
					:style="{textAlign: [header.headerTextAlign]}"
			>
				{{header.title}}
				<span v-if="header.sortable && sortedBy.fieldName === header.value" :class="{'caret-down': sortedBy.direction === 'asc', 'caret-up': sortedBy.direction === 'desc'}"></span>
			</div>
		</div>
		<div ref="rows" class="flex-grid-rows">
			<div class="flex-grid-rows-scrollable-container">
				<div class="flex-grid-row" v-for="(row, rowId) in items" :key="rowId" :class="{expanded: !row.collapsed, expandable: row.expandedData}">
					<div
							class="flex-grid-cell"

							v-for="(column, columnId) in columns"
							:key="columnId"
							v-if="!column.hidden"
							:style="{
								textAlign: [column.align ? column.align : 'left'],
								paddingLeft: !row.expandedData && columnId === 1 ? '24px' : '10px'
							}"
					>
						<span v-if="row.expandedData && columnId === 1" :class="{'caret-down': row.collapsed, 'caret-up': !row.collapsed}" @click="toggleExpandedData(row)"></span>
						{{columns[columnId].renderer && row[column.value] ? columns[columnId].renderer(row[column.value]) : row[column.value]}}

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
									}"
							>
								{{subRow[column.value] ? (column.renderer ? column.renderer(subRow[column.value]) : subRow[column.value]) : ''}}
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
		background: #fff;
		height: 100%;
		overflow: hidden;
		border-left: 1px solid #e6e6e6;
		border-top: 1px solid #e6e6e6;
		border-bottom: 1px solid #e6e6e6;

		&.striped {
			.flex-grid-row:nth-child(odd), .flex-grid-expanded-row:nth-child(odd) {
				background-color: rgba(0,0,0,.05);
			}
		}
		.flex-grid-row.expandable:not(.expanded):hover, .flex-grid-row:not(.expandable):hover, .flex-grid-expanded-row:hover {
			background-color: rgba(0,0,0,.1)!important;
		}
		.flex-grid-headers {
			background-color: #e6e6e6;
			border-bottom: 1px solid #e0e0e0;

			.flex-grid-header {
				display: inline-block;
				min-width: 20px;
				padding: 5px 10px;
				font-weight: bold;
				word-break: break-all;
				white-space: nowrap;
				&:not(:last-child)::after {
					content: '';
					margin-right: -10px;
					height: 20px;
					padding-top: 1px;
					border-left: 1px solid #d8d8d8;
					float: right;
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
					.flex-grid-expanded-container {
						max-height: 400px;
						overflow: hidden;
						transition: all .2s linear;
						//padding-bottom: 10px;
						&.collapsed {
							max-height: 0;
							padding-bottom: 0;
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
					}
				}
			}
		}


		.flex-grid-footer {
			clear: both;
			width: calc(100% + 5px);
			background-color: #e6e6e6;
			border-top: 1px solid #e0e0e0;
			min-height: 30px;
			margin-left: -5px;
			.flex-grid-footer-cell {
				display: inline-block;
				min-width: 20px;
				word-break: break-all;
				white-space: nowrap;
				font-weight: 500;
				padding: 5px 4px 5px 10px;
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
	}
</style>