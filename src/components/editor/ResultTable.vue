<template>
  <div class="result-table">
    <div ref="tabulator"></div>
    <statusbar :mode="statusbarMode" class="tabulator-footer">
      <div class="col x12 flex flex-center">
        <span ref="paginationArea" class="tabulator-paginator" v-show="this.resultRecords >= this.limit"></span>
      </div>
    </statusbar>
  </div>
</template>

<script type="text/javascript">
  import Tabulator from 'tabulator-tables'
  import Converter from '../../mixins/data_converter'
  import Mutators from '../../mixins/data_mutators'
  import Statusbar from '../common/StatusBar'
  import { Parser } from 'node-sql-parser/build/mysql';
  import _ from 'lodash'
  import dateFormat from 'dateformat'
  import Papa from 'papaparse';
  import FileSaver from 'file-saver';

  export default {
    components: { Statusbar },
    mixins: [Converter, Mutators],
    data() {
      return {
        tabulator: null,
        limit: 1000,
        totalRecords: 0,
        resultRecords: 0
      }
    },
    props: ['result', 'tableHeight', 'query', 'active', 'connection'],
    watch: {
      active() {
        if (!this.tabulator) return;
        if (this.active) {
          this.tabulator.restoreRedraw()
          this.$nextTick(() => {
            this.tabulator.redraw()
          })
        } else {
          this.tabulator.blockRedraw()
        }
      },
      tableData: {
        handler() {
          // this.tabulator.replaceData(this.tableData)
        }
      },
      tableColumns: {
        handler() {
          this.tabulator.setColumns(this.tableColumns)
        }
      },
      tableHeight() {
        this.tabulator.setHeight(this.actualTableHeight)
      }
    },
    computed: {
      statusbarMode() {
        return null
      },
      tableData() {
          return this.dataToTableData(this.result, this.tableColumns)
      },
      tableTruncated() {
          return this.result.truncated
      },
      tableColumns() {
        const columnWidth = this.result.fields.length > 20 ? 125 : undefined
        const columns = this.result.fields.map((column) => {
          const result = {
            title: column.name,
            field: column.name,
            width: columnWidth,
            mutatorData: this.resolveDataMutator(this.result.rows[0][column.name]),
            formatter: this.cellFormatter
          }
          return result;
        })
        return columns
      },
      actualTableHeight() {
        return '100%'
        // let result = this.tableHeight
        // if (this.tableHeight == 0) {
        //   result = '100%'
        // }
        // return result
      },
    },
    beforeDestroy() {
      if (this.tabulator) {
        this.tabulator.destroy()
      }
    },
    async mounted() {
      this.tabulator = new Tabulator(this.$refs.tabulator, {
        //data: this.tableData, //link data to table
        ajaxURL: "http://fake",
        ajaxSorting: true,
        ajaxFiltering: true,
        pagination: "remote",
        paginationSize: this.limit,
        paginationElement: this.$refs.paginationArea,
        // callbacks
        ajaxRequestFunc: this.dataFetch,
        reactiveData: false,
        virtualDomHoz: true,
        columns: this.tableColumns, //define table columns
        height: this.actualTableHeight,
        nestedFieldSeparator: false,
        cellClick: this.cellClick,
        clipboard: true,
        keybindings: {
          copyToClipboard: false
        },
        downloadConfig: {
          columnHeaders: true
        }
      });
    },
    methods: {
      parseQuery(query) {
          // import mysql parser only
          const parser = new Parser();
          const opt = {
            database: 'MySQL' // MySQL is the default database
          }
          const ast = parser.astify(query, opt);
          /**
          {
            "with": null,
            "type": "select",
            "options": null,
            "distinct": null,
            "columns": "*",
            "from": [
              {
                "db": null,
                "table": "t",
                "as": null
              }
            ],
            "where": null,
            "groupby": null,
            "having": null,
            "orderby": null,
            "limit": null
          }
          **/
          let countast =  parser.astify(query, opt); 
          countast.columns = [{"expr":{"type":"number","value":1},"as":"count"}]
          countast.distinct = null
          let countsql = parser.sqlify(countast, opt);

          countsql = "SELECT SUM(count) count FROM (" + countsql + ") res"
          
          let limitsql = parser.sqlify(ast, opt);
          let limit = 1000
          if (!ast.limit || ast.limit.value[0].value > 1000) {
            let limitast =  parser.astify(query, opt);
            limitast.limit = {"seperator":"offset","value":[{"type":"number","value":this.limit}]}
            limitsql = parser.sqlify(limitast, opt);
          } else {
            limit = ast.limit.value[0].value
          }

          //const sql = parser.sqlify(ast, opt);
          return [countsql,limitsql,limit] 

      },
      dataFetch(url, config, params) {
        
        
        const result = new Promise((resolve, reject) => {
          (async () => {
            try {
              const [countsql,limitsql,limit] = this.parseQuery(this.result.query);
              let offset = '';
              if (params.page && params.page > 1) {
                offset = ' OFFSET ' + (params.page - 1) * limit;
              }
              const query = this.connection.query(limitsql + offset)
              const response = await query.execute()
              Object.freeze(response)
              const countQuery = await this.connection.query(countsql).execute()
              const count = countQuery[0].rows[0]['count']
              this.totalRecords = count
              this.resultRecords = this.result.rows.length
              const fields = response[0].fields
              const columnWidth = fields.length > 20 ? 125 : undefined
              const columns = fields.map((column) => {
                const result = {
                  title: column.name,
                  field: column.name,
                  width: columnWidth,
                  mutatorData: this.resolveDataMutator(this.result.rows[0][column.name]),
                  formatter: this.cellFormatter
                }
                return result;
              })
              
              const data = this.dataToTableData(response[0],columns)
              Object.freeze(data)

              resolve({
                last_page: Math.ceil(this.totalRecords / limit),
                data
              });
            } catch (error) {
              reject();
              throw error;
            }
          })();
        });
        return result;
      },
      cellClick(e, cell) {
        this.selectChildren(cell.getElement())
      },
      async download() {
        this.tabulator.modules.ajax.showLoader()
        this.tabulator.blockRedraw()
      
        const query = this.result.query
        //await this.connection.database.connection.streamQuery(query)
        let response = await this.connection.query(query).execute()
        Object.freeze(response)
        
        var dataString = Papa.unparse(response[0].rows);
        Object.freeze(dataString)
        const dateString = dateFormat(new Date(), 'yyyy-mm-dd_hMMss')
        const title = this.query.title ? _.snakeCase(this.query.title) : "query_results"
        FileSaver.saveAs(new Blob([dataString], { type: 'text/csv;charset=utf-8' }), `${title}-${dateString}.csv`);

        
        this.tabulator.modules.ajax.hideLoader()
        this.tabulator.restoreRedraw()
      },
      clipboard() {
        this.tabulator.copyToClipboard("table", true)
        this.$noty.info("Table data copied to clipboard")
      }
    }
	}
</script>
